# Docker-Cluster-Orchestrator — Execution Trace Analysis

Repo analyzed: `blessysam15/Docker-Cluster-Orchestrator` (commit `d0e9c7d`, single "initial commit").
Method: pure static reading of every file in the repo. Nothing was executed, so anything that depends on runtime state is explicitly flagged in section 17 and inline as "cannot determine statically".

---

## 1. Project Overview

This is a **miniature Kubernetes-like control plane for Docker**, built from three cooperating programs plus a CLI:

| Component | Language | Role | Listens on |
|---|---|---|---|
| **Manager** (`server/manager`) | Python / FastAPI | Brain. Owns the SQLite DB, exposes the REST API, accepts agent registrations/heartbeats, decides *where* to place workloads, and forwards Docker commands to agents. | HTTP `8000`, gRPC `9000` |
| **Agent** (`server/agent`) | Python / gRPC + Docker SDK | Muscle. Runs on every worker PC, talks to the local Docker Engine, and pushes a heartbeat to the Manager every 5 s. | gRPC `9001` |
| **CLI** (`server/cli`) | Python / Typer + httpx | Thin HTTP client of the Manager REST API. | — |
| **Dashboard** (`client`) | React 19 + Vite | Browser UI that polls the Manager REST API every 5 s. | dev `5173`, prod nginx `80` |

Three facts define the whole design and you should hold them in your head while reading everything else:

1. **The Manager never touches Docker.** It only speaks gRPC to agents. Every `docker` verb happens inside `ContainerManager` on the agent.
2. **There is no reconciliation / self-healing.** This is deliberate and stated in `manager/app/state/reconciler.py`, `models.py`, `main.py` and `START_HERE.md`. A container changes state only when a human triggers an API call. Containers are even created with `restart_policy={"Name": "no"}`.
3. **The gRPC transport does not use protobuf.** `proto/orchestrator.proto` is documentation only. The real wire format is **JSON bytes carried over gRPC generic handlers** (`common/rpc.py: dumps/loads`). This is why you will find no `*_pb2.py` files, and why request objects inside handlers are plain Python `dict`s.

---

## 2. Project Structure

```text
Docker-Cluster-Orchestrator/
├── client/                          # React dashboard (Vite build -> nginx image)
│   ├── index.html                   # HTML entry, loads /src/main.jsx
│   ├── vite.config.js               # dev server :5173 + (unused) /manager-api proxy
│   ├── Dockerfile, docker-compose.yml
│   └── src/
│       ├── main.jsx                 # ReactDOM.createRoot -> <App/>
│       ├── App.jsx                  # page switch + 5 s polling loop
│       ├── api.js                   # the ONLY place fetch() is called
│       ├── components/              # Layout, MachineCard, ContainerCard, Gauge, ...
│       └── pages/                   # Overview, Scheduler, Machines, Containers,
│                                    # Deploy, Logs, Images, Deployments, Database
└── server/
    ├── pyproject.toml               # console scripts: orchestrator, orchestrator-agent
    ├── requirements.txt
    ├── agent.yaml                   # committed agent state (see §17)
    ├── proto/orchestrator.proto     # contract DOC only, never compiled
    ├── common/rpc.py                # dumps()/loads() JSON codec used by both sides
    ├── manager/app/
    │   ├── main.py                  # >>> MANAGER ENTRY POINT (FastAPI `app`)
    │   ├── core/config.py           # Settings dataclass from env vars
    │   ├── api/routes.py            # all /api/* endpoints (571 lines)
    │   ├── api/scheduler.py         # /api/scheduler/scores, /recommendation
    │   ├── services/                # container_service, machine_service, scheduler_service
    │   ├── grpc/server.py           # Manager gRPC :9000 (Register, Heartbeat)
    │   ├── grpc/client.py           # AgentClient -> agent :9001
    │   ├── database/                # database.py (engine), models.py, repository.py
    │   ├── state/reconciler.py      # DEAD (never imported)
    │   └── registry/__init__.py     # empty package, DEAD
    ├── agent/
    │   ├── cli.py                   # >>> AGENT ENTRY POINT (typer: join/start/status/leave)
    │   ├── Dockerfile               # CMD ["orchestrator-agent", "start"]
    │   ├── agent.yaml               # committed agent state
    │   └── app/
    │       ├── main.py              # main(): wires Docker + gRPC server + heartbeat
    │       ├── core/config.py       # AgentConfig (agent.yaml load/save)
    │       ├── docker/docker_client.py, docker/container_manager.py
    │       ├── grpc/server.py       # Agent gRPC :9001 (10 methods)
    │       ├── heartbeat/heartbeat.py  # background thread, 5 s
    │       └── system/system_info.py   # self-IP discovery + psutil metrics
    └── cli/orchestrator/main.py     # >>> CLI ENTRY POINT (typer over httpx)
```

---

## 3. Entry Point

There is no single entry point — this is a distributed system with **four independent processes**. Here is each one and exactly how I identified it.

### 3.1 Manager — `server/manager/app/main.py` (the primary one)

How I determined it:
- `server/pyproject.toml` has no console script for the manager, but `README.md` and `START_HERE.md` both say `uvicorn manager.app.main:app --host 0.0.0.0 --port 8000`.
- `manager/app/main.py:51` creates the module-level ASGI object `app = FastAPI(..., lifespan=lifespan)`. That object *is* the entry point for uvicorn — uvicorn imports the module and calls `app`.
- Lines 95–103 also provide a fallback `if __name__ == "__main__": uvicorn.run("manager.app.main:app", ...)`, so `python -m manager.app.main` works too. Both paths converge on the same `app`.

**Critical detail:** because the import string is `manager.app.main:app`, the process CWD must be `server/`. The SQLite URL is relative (`sqlite:///./orchestrator.db`), so the DB file lands in whatever directory you launched uvicorn from.

### 3.2 Agent — `server/agent/cli.py` → `agent/app/main.py:main()`

How I determined it:
- `pyproject.toml`: `[project.scripts] orchestrator-agent = "agent.cli:app"`. `pip install -e .` generates a wrapper that imports `agent.cli` and calls the Typer object `app`.
- `agent/Dockerfile` line 6: `CMD ["orchestrator-agent", "start"]` — confirms `start` is the normal command.
- `agent/cli.py:28-30` `start()` calls `run_agent()`, which is `agent.app.main.main` (aliased at line 7).

### 3.3 CLI — `server/cli/orchestrator/main.py`

`pyproject.toml`: `orchestrator = "cli.orchestrator.main:app"`. A Typer instance is callable, so the generated console-script wrapper `app()` works. This process is a *pure HTTP client*; it imports nothing from `manager` or `agent`.

### 3.4 Dashboard — `client/index.html` → `client/src/main.jsx`

`index.html:24` `<script type="module" src="/src/main.jsx">`. Vite treats `index.html` as the build entry; `main.jsx` mounts `<App/>` into `#root`. In production, `client/Dockerfile` runs `npm run build` and copies `/app/dist` into nginx.

### 3.5 Which one is "the normal flow"?

Per `START_HERE.md` §"Startup Order": **Manager first**, then agents (`join` once, then `start`), then the dashboard. The Manager is the hub — every other component is a client of it. The rest of this document uses the Manager as the spine and hangs the other processes off it.

---

## 4. Complete Execution Flow

### 4.1 Manager startup

```text
uvicorn manager.app.main:app
  → import manager.app.main
      → manager/app/api/routes.py       (imports pull in services, repository, models)
      → manager/app/core/config.py      : Settings dataclass evaluated  → `settings`
      → manager/app/database/database.py: create_engine(settings.db_url) → `engine`, `SessionLocal`
  → FastAPI(...) constructed            (main.py:51)
  → CORSMiddleware added                (main.py:62)  origins 3000/5173 only
  → app.include_router(routes.router,    prefix="/api")
  → app.include_router(scheduler.router, prefix="/api")   → paths become /api/scheduler/*
  → uvicorn starts the ASGI lifespan
      → lifespan()                      (main.py:24)
          → database/database.py : init_db()
              → imports models (registers Machine/ContainerRecord/Deployment/Heartbeat)
              → Base.metadata.create_all(engine)   → CREATE TABLE IF NOT EXISTS x4
          → grpc/server.py : build_server()
              → grpc.server(ThreadPoolExecutor(max_workers=16))
              → ManagerGrpcService(settings.cluster_token)
              → registers 2 generic handlers: Register, Heartbeat
              → add_insecure_port("0.0.0.0:9000"); raises RuntimeError if bound == 0
          → grpc_server.start()          (non-blocking; gRPC owns its own threads)
          → LOG.info("manager started ...") + "reconciliation is DISABLED"
          → yield                        ← the server now serves HTTP forever
```

At this instant two listeners are live in **one** process: uvicorn's HTTP loop on 8000 and gRPC's thread pool on 9000. They share the same SQLAlchemy `engine` (hence `check_same_thread: False` in `database.py:13`).

### 4.2 Agent join (one-time, a short-lived process)

```text
orchestrator-agent join --manager 10.0.0.5:9000 --name machine-2 --token abc123
  → agent/cli.py : join()
      → agent/app/core/config.py : AgentConfig.new(name, manager_address, token, port=9001)
          → machine_id = str(uuid.uuid4())          ← identity is minted on the AGENT
      → AgentConfig.save()
          → yaml.safe_dump(asdict(self)) → writes CONFIG_PATH (env AGENT_CONFIG, default "agent.yaml" in CWD)
      → typer.echo(...) ; process EXITS
```

No network traffic happens during `join`. It only writes a YAML file.

### 4.3 Agent start (long-running)

```text
orchestrator-agent start
  → agent/cli.py : start() → agent/app/main.py : main()
      1. AgentConfig.load()            → FileNotFoundError if not joined
      2. DockerClient()                → docker.from_env(); client.ping()
                                         → RuntimeError("Cannot connect to Docker Engine") on failure
      3. ContainerManager(docker_client.client)
      4. HeartbeatWorker(config, docker_client, containers)   (interval=5)
      5. build_server(config, containers, docker_client)
             → registers 10 handlers under "orchestrator.AgentService"
             → add_insecure_port("0.0.0.0:9001")
      6. server.start()                → agent is now reachable by the Manager
      7. heartbeat.start()
             → register()   ← SYNCHRONOUS, blocking, must succeed
                 → collect_system_info(manager_address)
                     → local_ip_for(): UDP socket "connect" to manager host:1,
                       getsockname()[0]  → the LAN IP the OS would use to reach the manager
                       (no packet is actually sent; falls back to 127.0.0.1 on OSError)
                     → psutil.cpu_count(), virtual_memory()
                 → _manager_call("Register", payload)
                     → grpc.insecure_channel(manager_address)
                     → unary_unary("/orchestrator.ManagerService/Register",
                                   request_serializer=common.rpc.dumps,
                                   response_deserializer=common.rpc.loads)
                     → rpc(request_with_token, timeout=5)
                 → if manager returns a different machine_id: overwrite config + save() to agent.yaml
             → threading.Thread(target=_run, daemon=True).start()
      8. signal handlers for SIGINT/SIGTERM installed
      9. while not stop: time.sleep(1)   ← main thread parks here forever
```

### 4.4 Heartbeat loop (agent background thread, every 5 s)

```text
HeartbeatWorker._run()
  → send_once()
      → collect_system_info()   → cpu_percent (psutil, 0.1 s sample), memory_percent, ip, hostname
      → containers.list_containers(all_containers=False)  → len() = running count
      → docker_client.version()
      → _manager_call("Heartbeat", {...13 fields...})
  → stop_event.wait(5)
  ↺ (any exception is caught and logged as "heartbeat failed"; the loop never dies)
```

Manager side of that call:

```text
grpc :9000  /orchestrator.ManagerService/Heartbeat
  → common.rpc.loads(bytes) → dict
  → manager/app/grpc/server.py : ManagerGrpcService.heartbeat(request, context)
      → _authorized()          → abort UNAUTHENTICATED if token mismatch
      → machine_id present?    → abort INVALID_ARGUMENT if not
      → with SessionLocal() as db: repository.save_heartbeat(db, request)
            → db.get(Machine, machine_id)   (fallback: lookup by name)
            → if None → return False        → manager aborts NOT_FOUND "machine is not registered"
            → update machine.cpu_count/memory_mb/docker_version/ip/agent_port
            → machine.status = "healthy"; machine.last_heartbeat = utcnow()
            → db.add(Heartbeat(machine_id, timestamp, cpu_percent, memory_percent,
                               running_containers, health))     ← append-only metrics row
            → db.commit()
      → return {"ok": True, "server_time": <iso>}
  → common.rpc.dumps(dict) → bytes back to the agent
```

Two tables are written per heartbeat: an **UPDATE** on `machines` and an **INSERT** into `heartbeats`. The `heartbeats` table grows forever — one row per agent per 5 s (~17k rows/day/agent). Nothing prunes it.

### 4.5 The signature flow: deploying a workload

This is the path worth memorizing. Trigger: user clicks **DEPLOY WORKLOAD** on the dashboard.

```text
client/src/pages/Deploy.jsx : deploy()
  → builds payload {machine, replicas, image, name_prefix, env:[], ports:{}, volumes:[], network:null}
       machine = null           if "Automatic — weighted scheduler"
       machine = "all"          if "All active machines"
       machine = ["m1","m2"]    if "Select machines"
  → client/src/api.js : api.runContainer(payload)
      → request("POST", "/api/deployments", {body, timeout:60000})
      → fetch(`${API_BASE}/api/deployments`)          API_BASE = VITE_MANAGER_URL || http://localhost:8000
          │
          ▼  HTTP POST over the network
manager/app/api/routes.py : run(req: RunRequest, db = Depends(get_db))
  │  FastAPI first validates the body against RunRequest (pydantic):
  │    replicas ge=1 le=100, image min_length=1, machine: str|list[str]|None
  │  get_db() yields a Session from SessionLocal and closes it after the response
  → _deploy_request(req, db)
      ├─ branch A: req.machine == "all" (case-insensitive)
      │     → _active_machine_rows(db)
      │         → repository.list_machine_health(...)  → keep active ids
      │         → repository.list_machines(db)         → filter to those ids
      │     → mode = "all-machines"; fall through to the per-machine loop
      ├─ branch B: isinstance(req.machine, list)
      │     → dedupe refs, repository.get_machine() each, verify each is active
      │     → mode = "selected-machines"; fall through to the loop
      └─ branch C: single name or None  ────────────────────────────────────┐
            → _select_deployment_machine(db, req.machine)                   │
                ├ requested is truthy → get_machine() (404-ish ValueError)  │
                │                     → list_machine_health() → must be active
                │                     → returns (machine.name, {"mode":"manual", ...})
                └ requested is None   → services/scheduler_service.py : scheduler_snapshot(db, 10, 15)
                                         → repository.list_machine_health(db, 10, 15)
                                         → WeightedResourceScheduler().explain(machines)
                                             → rank() → sorted by score desc
                                             → selected = first with status in (healthy, warning)
                                         → returns {"selected":…, "ranking":[…], "weights":{…}}
                                      → raises ValueError if selected is None
            → deploy(db, settings.cluster_token, machine_ref, image, replicas, ...)  ← single call
            → result["scheduling"] = scheduling ; RETURN (branch C ends here)
      (branches A/B) for machine in target_machines:  deploy(...) inside try/except
            → collect per-machine {ok:true, ...} or {ok:false, error:str}
            → return {mode, image, replicas_per_machine, target_machines,
                      successful_machines, failed_machines, results:[...]}
```

Now the shared `deploy()`:

```text
manager/app/services/container_service.py : deploy(db, token, machine_ref, image, replicas, ...)
  → services/machine_service.py : resolve_machine(db, machine_ref)
      → repository.get_machine(db, ref)   (PK lookup, then by name)  → ValueError if missing
  → freshness guard:
      last = machine.last_heartbeat ; None → ValueError("has never sent a heartbeat")
      naive datetimes are coerced to UTC (SQLite loses tzinfo)
      age > settings.heartbeat_offline_seconds (15) → ValueError("is offline")
  → deployment_id = uuid4()
  → prefix = name_prefix or image.split("/")[-1].split(":")[0].replace(".", "-")
    prefix = f"{prefix}-{deployment_id[:6]}"        e.g. "web-3f9ac1"
  → AgentClient(f"{machine.ip}:{machine.agent_port}", token, timeout=60)
  → labels = {"orchestrator.managed":"true", "orchestrator.deployment_id": deployment_id}
  → client.create_containers(image=…, replicas=…, name_prefix=prefix, env, ports, volumes,
                             network, labels)
        → manager/app/grpc/client.py : _call("CreateContainers", kwargs, timeout=60)
             → injects request["token"] = token
             → dumps() → gRPC → agent :9001
                 │
                 ▼
        agent/app/grpc/server.py : AgentGrpcService.create_containers(request, context)
             → _auth()  → abort UNAUTHENTICATED on token mismatch
             → agent/app/docker/container_manager.py : ContainerManager.create_containers(...)
                   → validate replicas >= 1, image non-empty, name_prefix non-empty
                   → self.client.images.pull(image)          ← can be slow; hence 60 s timeout
                   → _normalize_env / _normalize_ports / _normalize_volumes
                   → for index in 1..replicas:
                         name = f"{prefix}-{index}"
                         self.client.containers.get(name)
                             → NotFound  → ok, continue
                             → found     → raise ValueError("already exists")
                         container = client.containers.create(image, name, environment, ports,
                                       volumes, network, labels, detach=True,
                                       restart_policy={"Name": "no"})   ← no self-healing
                         container.start() ; container.reload()
                         created.append(serialize(container))
                         on exception → container.remove(force=True) then re-raise
                   → returns list[dict]
             → {"ok": True, "containers": items}  → dumps() → back over gRPC
  → finally: client.close()
  → db.add(Deployment(id, machine_id, image, replicas, name_prefix=prefix,
                      env_json, ports_json, volumes_json, network)) ; db.commit()
  → for item in response["containers"]:
        repository.save_container(db, {**item, "machine_id": machine.id}, deployment_id)
            → upsert into ContainerRecord (PK = docker container id) ; commit ; refresh
  → return {"deployment_id", "machine", "desired_replicas", "containers"}
        ↑ back to routes.run() → FastAPI serializes to JSON → HTTP 200
              ↑ api.js resolves the promise → Deploy.jsx setResult(response) + setMessage(...)
                  ↑ the DEPLOYMENT RESULTS table renders
```

**Ordering nuance worth knowing:** the `Deployment` row is written *after* Docker confirms creation. If the agent fails, the exception propagates up through `deploy()` → `run()` → `HTTPException(502, "Agent operation failed: ...")` and **no DB rows are written at all**. That is the correct ordering, but it also means a partially-created batch (say replica 3 of 5 fails) leaves replicas 1–2 running on Docker with **no** Manager record, because `ContainerManager` only rolls back the single container that failed to start.

### 4.6 Container lifecycle action (start / stop / restart / remove)

```text
UI ContainerCard button (or `orchestrator stop NAME`)
  → api.js stopContainer(id) → POST /api/containers/{ref}/stop
  → routes.py : stop_container(ref, db)
      → _find_container(db, ref)                       ← O(number_of_machines) fan-out
            for machine in repository.list_machines(db):     ← ALL machines, incl. offline
                client = container_service.agent_for(machine, token)   (AgentClient, 8 s timeout)
                client.list_containers(True)  → match on id | short_id | name
                except → continue (silently skip unreachable machines)
                finally → client.close()
            not found → HTTPException(404)
      → client = agent_for(machine, settings.cluster_token)
      → client.stop_container(container["id"])
            → gRPC StopContainer → AgentGrpcService.stop_container
                → ContainerManager.stop(id) → get() → container.stop() → reload() → serialize()
      → repository.update_container_status(db, container_id, "exited")
            → db.get(ContainerRecord, id); returns None silently if the container
              was not created through the Manager
      → return {"ok":true, "action":"stop", "container_id", "result",
                "message":"Container stopped. No reconciler will restart it."}
```

`remove` is the only one with extra bookkeeping (routes.py:472):

```text
remove_container(ref, force, db)
  → _find_container → get_container_record(db, id) → remember deployment_id
  → client.remove_container(id, force)          ← Docker first
  → repository.delete_container(db, id)         ← DB record only after Docker confirms
  → if deployment_id: deployment.replicas = max(0, replicas-1)
                      if replicas == 0: deployment.active = False
                      db.commit()
```

---

## 5. Detailed File → Function → Function Flow

Ten concrete chains. Read these as "what a debugger would show you."

**A. Manager boot**
```
manager/app/main.py            → module import
manager/app/core/config.py     → Settings()                      → settings
manager/app/database/database.py → create_engine()               → engine, SessionLocal
manager/app/main.py            → lifespan()
manager/app/database/database.py → init_db()
manager/app/database/models.py  → Machine/ContainerRecord/Deployment/Heartbeat class bodies
sqlalchemy                      → Base.metadata.create_all(engine) → CREATE TABLEs
manager/app/grpc/server.py      → build_server() → ManagerGrpcService.__init__(token)
grpc                            → server.add_insecure_port("0.0.0.0:9000") ; server.start()
```

**B. Agent registration (cross-process)**
```
agent/app/main.py               → main()
agent/app/core/config.py        → AgentConfig.load()             → dataclass from agent.yaml
agent/app/docker/docker_client.py → DockerClient.__init__()      → docker.from_env().ping()
agent/app/heartbeat/heartbeat.py → HeartbeatWorker.start() → register()
agent/app/system/system_info.py → collect_system_info() → local_ip_for()
common/rpc.py                   → dumps(payload)
        ~~~~~~ gRPC :9000 ~~~~~~
common/rpc.py                   → loads(bytes)
manager/app/grpc/server.py      → ManagerGrpcService.register()  → _authorized()
manager/app/database/repository.py → upsert_machine(db, request)
sqlite                          → INSERT/UPDATE machines
manager/app/grpc/server.py      → {"ok":true,"machine_id":…}
agent/app/heartbeat/heartbeat.py → config.machine_id = canonical ; config.save()
agent/app/core/config.py        → AgentConfig.save()             → rewrites agent.yaml
```

**C. Automatic placement**
```
client/src/pages/Deploy.jsx     → deploy()
client/src/api.js               → runContainer() → request() → fetch()
manager/app/api/routes.py       → run() → _deploy_request() → _select_deployment_machine()
manager/app/services/scheduler_service.py → scheduler_snapshot()
manager/app/database/repository.py → list_machine_health() → get_latest_heartbeat() (per machine)
manager/app/services/scheduler_service.py → WeightedResourceScheduler.explain() → rank() → score()
manager/app/services/container_service.py → deploy()
manager/app/services/machine_service.py   → resolve_machine() → repository.get_machine()
manager/app/grpc/client.py      → AgentClient.create_containers() → _call()
common/rpc.py                   → dumps()
        ~~~~~~ gRPC :9001 ~~~~~~
agent/app/grpc/server.py        → create_containers() → _auth()
agent/app/docker/container_manager.py → create_containers() → images.pull() → containers.create()
docker-py                       → Docker Engine API (unix socket / npipe)
agent/app/docker/container_manager.py → serialize()
        ~~~~~~ reply ~~~~~~
manager/app/services/container_service.py → db.add(Deployment) → repository.save_container()
manager/app/api/routes.py       → JSON response
client/src/pages/Deploy.jsx     → setResult() → DEPLOYMENT RESULTS table
```

**D. Dashboard 5-second poll**
```
client/src/main.jsx  → ReactDOM.createRoot().render(<App/>)
client/src/App.jsx   → useEffect → refreshCoreData() → setInterval(5000)
client/src/api.js    → Promise.all([getMachineHealth(true), getSchedulerScores()])
   GET /api/health/machines?active_only=true → routes.machine_health() → list_machine_health()
   GET /api/scheduler/scores                 → api/scheduler.py scores() → scheduler_snapshot()
client/src/App.jsx   → setMachines(), setScores() → props → pages/components re-render
```

**E. Logs**
```
pages/Logs.jsx fetchLogs() → api.getContainerLogs(name, tail)
GET /api/containers/{ref}/logs?tail=200
routes.logs() → _find_container() → agent_for() → AgentClient.logs()
agent grpc server.logs() → ContainerManager.logs() → container.logs(tail=…).decode("utf-8","replace")
→ {"ok":true,"logs":"<text>"} → Logs.jsx setLogs(data.logs) → <pre>
```

**F–J (abbreviated, same shape):** `/api/containers` → fan-out `ListContainers`; `/api/images` → fan-out `ListImages`; `/api/machines/{ref}/ping` → `resolve_machine` → `AgentClient.ping` → agent `docker.version()`; `/api/database` → `database_stats` + `Path(engine.url.database)` size + `os.getpid()`; `/api/deployments` (GET) → `list_deployments` → pure DB read (the only read endpoint that never touches an agent, besides `/api/stats`, `/api/machines`, `/api/health*` and `/api/scheduler/*`).

---

## 6. Data Flow

### 6.1 Telemetry (agent → manager → UI)

```
psutil.cpu_percent(interval=0.1), virtual_memory().percent, socket.gethostname()
   ↓ collect_system_info() dict
   ↓ + machine_id/name/agent_port/running_containers/health/docker_version
   ↓ common.rpc.dumps → JSON bytes → gRPC Heartbeat
   ↓ loads → dict → save_heartbeat()
   ├→ UPDATE machines  SET cpu_count, memory_mb, docker_version, ip, agent_port,
   │                       status='healthy', last_heartbeat=now
   └→ INSERT heartbeats(machine_id, timestamp, cpu_percent, memory_percent,
                        running_containers, health)
   ↓ (later, on any API read)
   list_machine_health() → per machine: get_latest_heartbeat() → age = now - timestamp
   ↓ derives status ∈ {healthy, warning, offline} + active flag + a 20-key dict
   ↓ JSON over HTTP
   App.jsx setMachines() → props → MachineCard → Gauge / ResourceBar / EcgChart / StatusBadge
```

### 6.2 Deployment request

```
Deploy.jsx form state (image, prefix, replicas, mode, selectedMachines)
   ↓ payload object
   ↓ JSON.stringify → POST body
   ↓ pydantic RunRequest validation (bounds: replicas 1..100, image non-empty)
   ↓ placement decision → machine_ref: str
   ↓ prefix derivation → "web-3f9ac1"
   ↓ AgentClient kwargs dict (+ token injected in _call)
   ↓ ContainerManager normalizers:
        env    ["K=V", …]              → passed through
        ports  {"80/tcp": 8080}        → {"80/tcp": 8080} | {"80/tcp": ("1.2.3.4", 8080)}
        volumes ["/src:/dst:ro"]       → {"/src": {"bind":"/dst","mode":"ro"}}
   ↓ docker create + start
   ↓ serialize(container) → 13-key dict (id, short_id, name, image, status, state, health,
        health_log[-5:], started_at, finished_at, exit_code, restart_count, labels, ports)
   ↓ Deployment row (env_json/ports_json/volumes_json are json.dumps'd TEXT columns)
   ↓ ContainerRecord row per container
   ↓ HTTP JSON response → React state → table
```

### 6.3 Persisted artifacts

| Artifact | Written by | Where |
|---|---|---|
| `orchestrator.db` (SQLite, 4 tables) | Manager only | Manager CWD |
| `agent.yaml` | `AgentConfig.save()` on `join` and on machine_id canonicalization | Agent CWD (or `$AGENT_CONFIG`) |
| Docker containers/images | Agent via docker-py | Agent's Docker Engine |
| `client/dist/` | `npm run build` | baked into the nginx image |

---

## 7. Classes and Object Flow

**Manager process**

- `Settings` (`core/config.py`) — frozen dataclass, instantiated once at import as `settings`. Its fields' defaults call `os.getenv(...)` **at class-definition time**, so env vars are read exactly once, when the module is first imported. Re-instantiating `Settings()` later would *not* re-read the environment.
- `Base` / `Machine` / `ContainerRecord` / `Deployment` / `Heartbeat` (`database/`) — SQLAlchemy 2.0 declarative models using `Mapped[...]`/`mapped_column`. Instances are created in `upsert_machine`, `save_heartbeat`, `save_container` and `container_service.deploy`.
- `ManagerGrpcService` (`grpc/server.py`) — constructed once inside `build_server()` with the cluster token; its bound methods `register`/`heartbeat` are handed to `grpc.unary_unary_rpc_method_handler`. It is shared across all 16 gRPC worker threads, so it must stay stateless — and it is (it opens a fresh `SessionLocal()` per call).
- `AgentClient` (`grpc/client.py`) — **created and destroyed per operation**. `__init__` opens `grpc.insecure_channel(address)`; every caller wraps usage in `try/finally: client.close()`. There is no channel pooling, so a `/api/containers` call across N machines opens and closes N channels.
- `WeightedResourceScheduler` / `MachineScore` (`services/scheduler_service.py`) — the scheduler is instantiated fresh on every `scheduler_snapshot()` call; `MachineScore` is a dataclass converted with `asdict()` for JSON.
- `RunRequest` (`api/routes.py`) — pydantic model; FastAPI builds it from the POST body before your code runs.

Object-creation chain for a deployment:

```
routes.run(req: RunRequest)          ← pydantic constructs RunRequest
  → container_service.deploy()
      → AgentClient(...)             ← __init__ opens channel
      → Deployment(...)              ← ORM object, added+committed
      → save_container() → ContainerRecord(...) or mutate existing
      → AgentClient.close()          ← channel torn down in finally
```

**Agent process** (all four objects are created once in `main()` and live for the process lifetime)

```
main()
  → AgentConfig.load()                     → config      (mutable: machine_id may be rewritten)
  → DockerClient()                         → .client = docker.DockerClient
  → ContainerManager(docker_client.client)  → holds the raw docker client
  → HeartbeatWorker(config, docker_client, containers)
       → owns threading.Event + threading.Thread
  → build_server(config, containers, docker_client) → AgentGrpcService(...)
       ↑ note: the SAME ContainerManager instance is shared by the heartbeat thread
         (running-container count) and by all 16 gRPC handler threads.
         docker-py's client is thread-safe for these calls, so this is fine.
```

**Browser**

`App` is the only stateful container component: it owns `page`, `machines`, `scores`, `loading`, `connectionError`, and passes `{machines, scores, refreshCoreData, refreshMs}` to whichever page is selected. `Overview`, `Containers`, `Logs`, `Images`, `Deployments`, `Database` additionally fetch their own data on mount. `Machines` and `Scheduler` are pure props consumers — they render entirely from `App`'s 5 s poll.

---

## 8. API / Request Flow

Full lifecycle of `POST /api/containers/web-3f9ac1-1/stop`:

```
Browser fetch (api.js request(), AbortController with 30 s timeout)
  → CORS preflight is NOT triggered (no custom headers on non-body requests; POST without body
    sends only Accept) — but note the Manager's allowed origins are localhost:3000/5173 only
  → uvicorn → Starlette → CORSMiddleware → FastAPI router
  → path match: /api + /containers/{ref}/stop  → routes.stop_container
  → dependency resolution: Depends(get_db) → SessionLocal() (generator; closed after response)
  → handler body → _find_container() fan-out → AgentClient.stop_container()
  → repository.update_container_status()
  → return dict → FastAPI jsonable_encoder → JSONResponse 200
  → get_db() finally: db.close()
  → response → api.js parses text → JSON → ContainerCard onChanged() → reload list
```

Endpoint inventory (all mounted under `/api`):

| Method + path | Handler | Touches agents? | Used by |
|---|---|---|---|
| GET `/health` | `routes.health` | no | nobody in-repo (api.js defines it, no page calls it) |
| GET `/health/machines` | `routes.machine_health` | no | App.jsx poll, CLI `health` |
| GET `/stats` | `routes.stats` | no | api.js only |
| GET `/machines` | `routes.machines` | no | CLI `machines` |
| DELETE `/machines/{ref}` | `routes.delete_machine` | no | nobody |
| GET `/deployments` | `routes.deployments` | no | Deployments page |
| POST `/deployments` | `routes.run` | **yes** | Deploy page, CLI `run` |
| POST `/deployments/auto` | `routes.run_auto` | **yes** | nobody (api.js defines `runContainerAuto`, unused) |
| GET `/containers` | `routes.containers` | **yes (fan-out)** | Overview, Containers, Logs, CLI `ps` |
| GET `/images` | `routes.images` | **yes (fan-out)** | Images page, CLI `images` |
| POST `/machines/{ref}/ping` | `routes.ping_machine` | **yes** | nobody |
| POST `/containers/{ref}/start|stop|restart` | resp. handlers | **yes** | ContainerCard, CLI |
| DELETE `/containers/{ref}` | `routes.remove_container` | **yes** | ContainerCard, CLI `rm` |
| GET `/containers/{ref}/inspect` | `routes.inspect_container` | **yes** | ContainerCard |
| GET `/containers/{ref}/logs` | `routes.logs` | **yes** | Logs page, ContainerCard, CLI |
| GET `/scheduler/scores` | `api/scheduler.py scores` | no | App.jsx poll |
| GET `/scheduler/recommendation` | `api/scheduler.py recommendation` | no | nobody |

The gRPC "API" is the second interface: `orchestrator.ManagerService` (Register, Heartbeat) on 9000 and `orchestrator.AgentService` (Ping, ListContainers, ListImages, CreateContainers, Start/Stop/Restart/Remove/Inspect/Logs) on 9001.

---

## 9. Database Flow

Engine: SQLAlchemy 2.0 + SQLite (`sqlite:///./orchestrator.db`), created in `database.py` at import, `check_same_thread=False` so the gRPC threads and the HTTP threads can share it. Schema is created by `Base.metadata.create_all` in `init_db()` — **there are no migrations**, so adding a column later would require manually dropping the DB.

Session strategy — two distinct patterns:
- **HTTP:** `Depends(get_db)` — a generator dependency that yields one `Session` per request and closes it in `finally`.
- **gRPC:** `with SessionLocal() as db:` inside each handler in `manager/app/grpc/server.py`.

Tables:

| Table | PK | Written by | Read by |
|---|---|---|---|
| `machines` | agent-generated UUID string | `upsert_machine`, `save_heartbeat` | everything |
| `heartbeats` | autoincrement int | `save_heartbeat` (append-only) | `get_latest_heartbeat` |
| `containers` (`ContainerRecord`) | Docker container id | `save_container`, `update_container_status`, `delete_container` | `get_container_record`, `database_stats` |
| `deployments` | UUID string | `container_service.deploy`, decremented in `remove_container` | `list_deployments` |

The interesting query is `list_machine_health` (`repository.py:143`): it calls `list_machines()` and then `get_latest_heartbeat()` **once per machine** — a classic N+1. With the dashboard polling twice every 5 s, this is N+1 queries × 2 × every 5 s. Fine for a handful of machines, the first thing to fix at scale.

Note also that SQLite stores `DateTime(timezone=True)` without tz info; three separate places defensively re-attach UTC (`repository._aware`, `container_service.deploy`, `reconciler.mark_stale_machines`).

---

## 10. External Services / Dependencies

| Dependency | Where it enters | Why |
|---|---|---|
| **Docker Engine** | `agent/app/docker/docker_client.py: docker.from_env()` + `.ping()` | The only thing that actually runs containers. Reached via unix socket / named pipe on the agent host. Agent refuses to start without it. |
| **gRPC (`grpcio`)** | `manager/app/grpc/{server,client}.py`, `agent/app/grpc/server.py`, `heartbeat.py` | Transport between manager and agents. Used with **generic handlers + JSON codec**, not protobuf. Insecure channels only. |
| **FastAPI / Starlette / uvicorn** | `manager/app/main.py` | REST API + ASGI server + CORS. |
| **SQLAlchemy 2.0 + SQLite** | `manager/app/database/*` | Manager-only persistence. |
| **pydantic** | `RunRequest` in `routes.py` | Request validation/coercion. |
| **psutil** | `agent/app/system/system_info.py` | CPU count/percent, RAM total/percent for heartbeats. |
| **PyYAML** | `agent/app/core/config.py` | Reads/writes `agent.yaml`. |
| **Typer + httpx** | `cli/orchestrator/main.py`, `agent/cli.py` | CLIs; httpx is the CLI→Manager HTTP client. |
| **React 19 + Vite** | `client/` | Dashboard SPA. |
| **nginx** | `client/Dockerfile` stage 2 | Serves the built static bundle on port 80 (published as 5173). |
| **`proto/orchestrator.proto`** | nowhere at runtime | Documentation of the RPC contract for a future protobuf migration. |

Listed in `requirements.txt` but **not imported anywhere in the repo**: `streamlit`, `streamlit-autorefresh`, `altair`, `pandas`, `numpy`, `plotly`, `pyarrow`, `pydeck`, `pywin32`, `python-multipart`, `Jinja2`. These are leftovers from the Streamlit dashboard that `README.md` still documents but which no longer exists in the tree (see §17).

No external HTTP APIs, no message queue, no ML model, no auth provider. "Authentication" is a single shared `CLUSTER_TOKEN` string compared with `!=` in two places.

---

## 11. Configuration Flow

**Manager** — `manager/app/core/config.py`, read once at import:

```
environment
  MANAGER_HOST                (default 0.0.0.0)   → gRPC bind address (NOT the HTTP bind;
                                                    HTTP host comes from the uvicorn CLI flag)
  MANAGER_HTTP_PORT           (8000)              → reported in /api/health, used by __main__ block
  MANAGER_GRPC_PORT           (9000)              → grpc/server.py add_insecure_port
  MANAGER_DB_URL              (sqlite:///./orchestrator.db) → database.py create_engine
  CLUSTER_TOKEN               (dev-token-change-me) → ManagerGrpcService auth AND the token the
                                                      Manager presents to agents in every AgentClient
  RECONCILE_INTERVAL          (10)                → DEAD: nothing reads settings.reconcile_interval
  HEARTBEAT_WARNING_SECONDS   (10)                → list_machine_health / scheduler
  HEARTBEAT_OFFLINE_SECONDS   (15)                → list_machine_health, deploy() freshness guard
        ↓
  Settings frozen dataclass → module-level `settings`
        ↓
  imported by main.py, routes.py, scheduler.py, container_service.py, grpc/server.py, reconciler.py
```

**Agent** — `agent/app/core/config.py`:

```
AGENT_CONFIG env (default relative path "agent.yaml", resolved against the process CWD)
        ↓
agent.yaml { machine_id, name, manager_address, token, port }
        ↓
AgentConfig dataclass → main() → HeartbeatWorker (token, manager_address, identity)
                              → AgentGrpcService (_auth token, port)
        ↑ written back by AgentConfig.save() when the Manager canonicalizes machine_id
```

Because the default path is *relative*, the agent picks up a different `agent.yaml` depending on where you launch it: `server/agent.yaml` (name `Srikanth`, manager `10.65.30.227:9000`) if you run from `server/`, or `server/agent/agent.yaml` (name `machine-1`, manager `10.171.21.14:9000`) if you run from `server/agent/`. Both are committed to git with `token: abc123`.

**CLI** — `ORCHESTRATOR_MANAGER` (default `http://127.0.0.1:8000`), read at module import in `cli/orchestrator/main.py:12`.

**Dashboard** — `VITE_MANAGER_URL` (default `http://localhost:8000`) and `VITE_REFRESH_MS` (default 5000). These are **build-time** substitutions in Vite: in the Docker image they are frozen at `npm run build`, so changing them requires a rebuild, not a container restart. There is no `.env` file in the repo.

The token must match in three places or nothing works: the Manager's `CLUSTER_TOKEN`, each agent's `agent.yaml: token` (checked when the Manager calls the agent), and the same value again when the agent calls the Manager.

---

## 12. Error Handling and Branches

**Auth (both directions).** `if request.get("token") != self.token: context.abort(UNAUTHENTICATED)`. True → RPC dies with a gRPC error the caller surfaces as HTTP 502/503; False → continue. Normal path is False.

**`upsert_machine` identity resolution** (`repository.py:28`) — the trickiest branch in the codebase:
```
machine = db.get(Machine, machine_id)
if machine is None:
    existing_by_name = SELECT * FROM machines WHERE name = :name
    if existing_by_name:
        same_host = existing.hostname == hostname OR existing.ip == ip
        if not same_host: raise ValueError(...)      → gRPC ALREADY_EXISTS, agent registration fails
        else: machine = existing_by_name             → adopt the old row, agent adopts its id
if machine is None: create new row
```
TRUE path (a genuinely new agent) → new row. The `same_host` path is what makes "re-join the same PC" preserve identity — the agent then rewrites its own `agent.yaml` with the Manager's id (`heartbeat.py:62-70`). The error path is what stops two different PCs from claiming `machine-2`.

**Heartbeat for an unknown machine** → `save_heartbeat` returns `False` → `abort(NOT_FOUND, "machine is not registered")`. The agent logs `heartbeat failed` and retries in 5 s forever. This is what you see if the Manager's DB was deleted while agents kept running.

**Health thresholds** (`list_machine_health`): `age is None or age > 15` → offline; `age > 10` → warning; else healthy. `active = status != "offline"`, so a *warning* machine is still deployable.

**Placement guard** (`deploy`): `last_heartbeat is None` → "has never sent a heartbeat"; `age > 15` → "is offline". Both are `ValueError` → `HTTPException(400)`.

**Scheduler selection**: `next((x for x in ranking if x.status in ("healthy","warning")), None)`; `None` → `ValueError("No active machine is available for automatic scheduling")` → 400. Note `score()` short-circuits offline machines to `0`, and they are also excluded by the status filter.

**Name collision on create** (`container_manager.create_containers`): `client.containers.get(name)` raising `NotFound` is the *happy* path (`try/except NotFound: pass / else: raise ValueError`). This is an easy inversion to misread — the `else` clause runs when *no* exception occurred, i.e. the name is taken.

**Partial-create rollback**: if `container.start()` throws, the just-created container is `remove(force=True)`d (best-effort, inner `except: pass`) and the exception re-raised. Earlier replicas in the same loop are **not** rolled back.

**Broadcast deploys** (`_deploy_request` branches A/B): each machine is wrapped in its own `try/except Exception` and recorded as `{"ok": False, "error": str(exc)}`, so one bad node cannot fail the whole batch. Contrast with the single-machine branch C, which has no per-machine catch and lets the error become a 400/502.

**Fan-out reads** (`containers`, `images`): a failing agent contributes an `{"machine":…, "error":…}` entry to the array instead of failing the request. The UI renders those rows as-is. `_find_container` instead swallows failures with a bare `except: continue`, so an unreachable machine simply "doesn't have" the container — which can produce a confusing 404 rather than a connectivity error.

**HTTP error mapping in `routes.run`**: `ValueError` → 400, everything else → `502 "Agent operation failed: …"`. Other handlers use 404 (unknown machine/container), 503 (`ping_machine` when the agent is unreachable), 502 (start/stop/restart/remove failures).

**Frontend**: `api.js request()` normalizes three cases — `ManagerAPIError` (HTTP >= 400, message includes `detail`), `AbortError` → "Request timed out", anything else → "Cannot connect to Manager at <base>". `App.refreshCoreData` catches and sets `connectionError`, which flips the sidebar to **CONTROL PLANE OFFLINE** (`Layout.jsx:20`).

**Agent resilience**: `HeartbeatWorker._run` catches every exception per iteration, so a Manager restart or LAN blip never kills the agent. Conversely, `register()` in `start()` is *not* retried — if the Manager is down when the agent starts, `main()` raises after stopping the gRPC server, and the agent process exits.

---

## 13. Background / Async Flow

| Concurrency unit | Started where | Behavior |
|---|---|---|
| uvicorn ASGI event loop | `uvicorn` CLI | Serves HTTP. All route handlers are **sync `def`**, so Starlette runs each in a thread-pool worker — blocking gRPC calls inside them do not block the event loop. |
| FastAPI lifespan | `main.py:24` `@asynccontextmanager` | Runs `init_db()` + gRPC start before serving; the code after `yield` runs at shutdown. |
| Manager gRPC thread pool | `build_server()` → `ThreadPoolExecutor(max_workers=16)` | Handles Register/Heartbeat concurrently with HTTP, in the same process, sharing `engine`. |
| Agent gRPC thread pool | `agent/app/grpc/server.py: build_server()` → 16 workers | Handles Manager commands. |
| Agent heartbeat thread | `HeartbeatWorker.start()` → `threading.Thread(daemon=True)` | 5 s loop; `threading.Event` doubles as sleep and stop signal (`stop_event.wait(5)`). |
| Agent main thread | `agent/app/main.py:39` | `while not stop: time.sleep(1)` — a 1 s-granularity park loop, woken by the SIGINT/SIGTERM handler setting `stop = True` via `nonlocal`. |
| Browser polling | `App.jsx:70` `setInterval(refreshCoreData, 5000)` | Cleared on unmount. Page-level `useEffect`s fetch once on mount (Overview re-fetches whenever `machines` changes). |

There are **no** async task queues, no schedulers/cron, no websockets, and no server-side background jobs in the Manager. `reconciler_loop` exists as an `async def` stub but is never awaited by anything.

---

## 14. Complete Execution Diagram

```text
┌─────────────────────── MANAGER PROCESS (one host) ───────────────────────┐
   uvicorn manager.app.main:app
        ↓
   import config → Settings          import database → engine, SessionLocal
        ↓
   FastAPI(app) + CORS + include_router(/api) + include_router(/api/scheduler)
        ↓
   lifespan(): init_db() → CREATE TABLEs
               build_server() → gRPC :9000 → start()
        ↓
   ┌── HTTP :8000 (uvicorn) ──┐        ┌── gRPC :9000 (16 threads) ──┐
   │  serves REST forever      │        │  Register / Heartbeat        │
   └───────────────────────────┘        └──────────────────────────────┘
└──────────────────────────────────────────────────────────────────────────┘
            ▲                                        ▲
            │ HTTP                                   │ gRPC (JSON payloads)
            │                                        │
┌───────────┴───────────┐              ┌─────────────┴──────────────────────┐
│  BROWSER (React SPA)  │              │  AGENT PROCESS (each worker host)   │
│  main.jsx → App       │              │  orchestrator-agent join → agent.yaml│
│  setInterval 5 s      │              │  orchestrator-agent start → main()   │
│   ├ /api/health/machines             │    AgentConfig.load()                │
│   └ /api/scheduler/scores            │    DockerClient() → docker.ping()    │
│  page fetches:        │              │    ContainerManager(client)          │
│   /api/containers     │              │    build_server() → gRPC :9001       │
│   /api/images         │              │    HeartbeatWorker.start()           │
│   /api/deployments    │              │       ├ register()  (once, blocking) │
│   /api/database       │              │       └ thread: send_once() every 5s │
│  actions:             │              │    while not stop: sleep(1)          │
│   POST /api/deployments ─────────────┼──→ CreateContainers                  │
│   POST /api/containers/{r}/stop  ────┼──→ StopContainer                     │
└───────────────────────┘              │             ↓                        │
                                       │      docker-py → Docker Engine       │
                                       │             ↓                        │
                                       │        CONTAINERS (restart: no)      │
                                       └──────────────────────────────────────┘

DEPLOY REQUEST, END TO END
  Deploy.jsx deploy()
        ↓  POST /api/deployments
  routes.run() → _deploy_request()
        ↓                       ↘ (machine == "all" | list) loop over active machines
  _select_deployment_machine()   ↘
        ↓ (auto)                  ↘
  scheduler_snapshot() → list_machine_health() → WeightedResourceScheduler.explain()
        ↓ selected.name
  container_service.deploy()
        ↓ resolve_machine() + heartbeat freshness check
        ↓ AgentClient.create_containers()   ── gRPC :9001 ──▶ AgentGrpcService.create_containers()
                                                                    ↓
                                                          ContainerManager.create_containers()
                                                                    ↓ images.pull / create / start
                                                                  DOCKER
                                                                    ↓ serialize()
        ◀────────────────────── {"ok":true,"containers":[…]} ──────┘
        ↓ INSERT deployments ; INSERT containers (per replica)
        ↓ {"deployment_id","machine","desired_replicas","containers"}
  HTTP 200 → Deploy.jsx setResult() → DEPLOYMENT RESULTS table
  (server keeps running; next poll in ≤5 s shows the new containers)
```

---

## 15. Function-Level Execution Sequence

**Manager boot**
1. `uvicorn` imports `manager.app.main`
2. `manager.app.core.config` → `Settings()` → `settings`
3. `manager.app.database.database` → `create_engine()`, `sessionmaker()`
4. `manager.app.api.routes` imports services/repository/models
5. `main.py` → `FastAPI(lifespan=lifespan)`
6. `main.py` → `app.add_middleware(CORSMiddleware, …)`
7. `main.py` → `app.include_router(router, prefix="/api")`
8. `main.py` → `app.include_router(scheduler_router, prefix="/api")`
9. `lifespan()` → `init_db()` → `Base.metadata.create_all(engine)`
10. `lifespan()` → `build_server()` → `ManagerGrpcService(token)` → `add_insecure_port` → `start()`
11. `lifespan()` → `yield` (serving)

**Agent boot + registration**
12. `agent.cli:app` → `start()` → `agent.app.main.main()`
13. `AgentConfig.load()`
14. `DockerClient.__init__()` → `docker.from_env()` → `client.ping()`
15. `ContainerManager.__init__()`
16. `HeartbeatWorker.__init__()`
17. `agent.app.grpc.server.build_server()` → `AgentGrpcService.__init__()` → `add_insecure_port(":9001")`
18. `server.start()`
19. `HeartbeatWorker.start()` → `register()`
20. `collect_system_info()` → `local_ip_for()` → `psutil.*`
21. `DockerClient.version()`
22. `_manager_call("Register", payload)` → `common.rpc.dumps`
23. → Manager `ManagerGrpcService.register()` → `_authorized()`
24. → `repository.upsert_machine()` → `db.commit()` → `db.refresh()`
25. ← `{"ok":true,"machine_id":…}` → maybe `AgentConfig.save()`
26. `threading.Thread(_run).start()`
27. `signal.signal(SIGINT/SIGTERM, handle_signal)`
28. `while not stop: time.sleep(1)`

**Heartbeat, repeating**
29. `HeartbeatWorker._run()` → `send_once()`
30. `collect_system_info()` + `ContainerManager.list_containers(all=False)` + `DockerClient.version()`
31. `_manager_call("Heartbeat", …)` → `ManagerGrpcService.heartbeat()` → `repository.save_heartbeat()`
32. `stop_event.wait(5)` → back to 29

**Dashboard poll, repeating**
33. `main.jsx` → `ReactDOM.createRoot().render(<App/>)`
34. `App.useEffect` → `refreshCoreData()` → `setInterval(…, 5000)`
35. `api.getMachineHealth(true)` → `routes.machine_health()` → `repository.list_machine_health()` → `get_latest_heartbeat()` per machine
36. `api.getSchedulerScores()` → `scheduler.scores()` → `scheduler_snapshot()` → `WeightedResourceScheduler.explain()` → `rank()` → `score()`
37. `setMachines()` / `setScores()` → `Layout` + active page re-render

**Deploy**
38. `Deploy.deploy()` → `api.runContainer(payload)` → `request("POST","/api/deployments")`
39. `routes.run()` → `_deploy_request()`
40. `_select_deployment_machine()` (or `_active_machine_rows()` for "all")
41. `scheduler_snapshot()` when automatic
42. `container_service.deploy()` → `machine_service.resolve_machine()` → `repository.get_machine()`
43. `AgentClient.__init__()` → `create_containers()` → `_call()` → `dumps()`
44. `AgentGrpcService.create_containers()` → `_auth()`
45. `ContainerManager.create_containers()` → `images.pull()` → `_normalize_env/ports/volumes`
46. per replica: `containers.get()` (expect `NotFound`) → `containers.create()` → `start()` → `reload()` → `serialize()`
47. ← reply → `AgentClient.close()`
48. `db.add(Deployment)` → `db.commit()`
49. `repository.save_container()` per container
50. return dict → JSON 200 → `Deploy.setResult()` / `setMessage()`

**Stop a container**
51. `ContainerCard.action("stop", …)` → `api.stopContainer(id)`
52. `routes.stop_container()` → `_find_container()` → per machine `agent_for()` + `list_containers()`
53. `AgentClient.stop_container()` → `AgentGrpcService.stop_container()` → `ContainerManager.stop()` → `container.stop()`
54. `repository.update_container_status(db, id, "exited")`
55. JSON 200 → `onChanged()` → `Containers.load()` → refreshed list

---

## 16. Where Execution Ends

**Manager** — never ends on its own. `lifespan` yields and uvicorn serves indefinitely. Per-request execution ends when a handler returns a dict, FastAPI serializes it, and `get_db()`'s `finally: db.close()` runs. On SIGINT/SIGTERM, uvicorn exits the lifespan context → `finally: grpc_server.stop(3).wait()` (3 s grace) → `LOG.info("manager stopped")` → process exits. The SQLite file persists.

**Agent** — never ends on its own either. Final state is the `while not stop: time.sleep(1)` park loop. Shutdown: signal → `handle_signal` sets `stop` → loop exits → `heartbeat.stop()` (`stop_event.set()`, `thread.join(timeout=2)`) → `server.stop(3).wait()` → `LOG.info("agent stopped")`. **Containers are not stopped when the agent stops** — they keep running under Docker, they just become unmanageable until the agent returns.

**CLI** — a genuinely terminating script: `orchestrator <cmd>` → `request()` → `typer.echo(json.dumps(...))` → exit 0. Errors exit non-zero via `typer.BadParameter`.

**`orchestrator-agent join`** — also terminating: writes `agent.yaml`, prints the machine id, exits.

**Dashboard** — a browser SPA. Execution "ends" only when the tab closes; `useEffect` cleanups clear the interval. Between renders it sits idle waiting on the 5 s timer.

So: three of the five entry points are servers/daemons (Manager, Agent, browser SPA) and two are one-shot scripts (`orchestrator`, `orchestrator-agent join`).

---

## 17. Unused / Dead / Alternative Code

How I determined each: repo-wide `rg` for every symbol name, then checking whether the caller itself is reachable from an entry point. Items flagged as "reachable but unused by in-repo callers" are still valid public surface — an operator with `curl` can use them.

**Definitely dead (no reference anywhere):**
- `manager/app/state/reconciler.py` — both `mark_stale_machines()` and `reconciler_loop()`. The module's own docstring says it is "NOT started by the Manager application", and grep confirms `reconciler` appears nowhere outside this file and two log/message strings. `Machine.status` is therefore only maintained by `upsert_machine`/`save_heartbeat` (which always set it to `"healthy"`); real status is computed dynamically in `list_machine_health`. So the `machines.status` column exposed by `GET /api/machines` is effectively meaningless.
- `manager/app/registry/` — a package containing only an empty `__init__.py`.
- `manager/app/services/container_service.py: container_to_dict()` — defined, never called.
- `settings.reconcile_interval` / `RECONCILE_INTERVAL` env var — never read.
- `import shutil` in `agent/cli.py` — unused import.
- `client/vite.config.js`'s `/manager-api` proxy — `api.js` always builds absolute URLs from `API_BASE`, so no request ever hits `/manager-api`. Dead in both dev and prod.

**Reachable, but nothing in this repo calls them** (deliberately keeping them, not deleting):
- `POST /api/deployments/auto` and `api.runContainerAuto()` — the Deploy page always posts to `/api/deployments` with `machine: null` for automatic mode, which produces identical behavior.
- `GET /api/scheduler/recommendation` and `api.getSchedulerRecommendation()`.
- `DELETE /api/machines/{ref}` (+ `machine_service.remove_machine`, `repository.delete_machine`) — no UI and no CLI command.
- `POST /api/machines/{ref}/ping` and `api.pingMachine()` — the README advertises an "agent connectivity test" that the React UI never wired up.
- `api.health()`, `api.getMachines()`, `api.getStats()` in `client/src/api.js` — the CLI uses the corresponding endpoints; the React app does not.
- `orchestrator-agent status` / `leave` — operator conveniences, outside the normal flow.
- `AgentClient.list_images/ping/inspect_container/...` are all used; nothing dead in that class.

**Stale documentation / alternative implementation:**
- `README.md` documents a **Streamlit dashboard** (`streamlit run dashboard/app.py`, `dashboard/requirements.txt`). Neither file exists. `requirements.txt` still pins `streamlit`, `streamlit-autorefresh`, `altair`, `pandas`, `numpy`, `plotly`, `pyarrow`, `pydeck`. The React `client/` is the surviving UI, documented in `START_HERE.md`. Treat `START_HERE.md` as current and the README's dashboard section as historical.
- `README.md`'s architecture diagram and `pyproject.toml`'s title still describe the Streamlit-era layout.
- `proto/orchestrator.proto` is never compiled or imported — a design document, by its own comment.
- `requirements.txt` pins `pywin32` unconditionally, which will fail to install on Linux/macOS. `pyproject.toml`'s dependency list is the portable one; prefer `pip install -e .`.

**Duplicated code:** a `PageHeader` component is redefined at the bottom of nearly every file in `client/src/pages/` (9 copies) instead of being imported once. `Overview.jsx` is the only page that exports its shared bits (`SectionTitle`, `Metric`, `Empty`), and `Scheduler.jsx` imports `SectionTitle` from it — a page importing from a sibling page.

**Committed runtime state / security:** `server/agent.yaml` and `server/agent/agent.yaml` are checked into git, each with `token: abc123` and a private LAN IP. These are supposed to be generated per host by `orchestrator-agent join`. Anyone with the repo has the cluster token for those deployments. `.gitignore` should cover `agent.yaml` and `orchestrator.db`.

---

## 18. Important Things to Understand

1. **JSON-over-gRPC, not protobuf.** `common/rpc.py` is 12 lines and is the single most important file for understanding the wire protocol. Every handler receives a `dict`. If you ever migrate to real protobuf, `build_server()` on both sides and `AgentClient._call` are the only places to change.
2. **The Manager is a proxy, not a controller.** Every container read is a live fan-out to agents (`GET /api/containers` calls `ListContainers` on *every* machine in the DB, including long-dead ones, each with an 8 s timeout). The DB's `containers` table is a *record of what the Manager created*, not the source of truth about what Docker is running. The two can diverge freely — for example, if you stop a container with `docker stop` directly on a worker, the DB row still says `running`.
3. **`_find_container` is the hidden cost centre.** Every start/stop/restart/remove/inspect/logs call first performs a full cluster scan to locate the container by id/short_id/name. With M machines that is M `ListContainers` RPCs before the actual operation. It also silently skips machines that error, so connectivity failures masquerade as 404s.
4. **No self-healing, on purpose.** `restart_policy={"Name":"no"}`, no reconciler, and explicit "No reconciler will restart it" messages in the API responses. If a container dies, it stays dead until a human clicks start.
5. **Identity handshake.** The agent mints its own UUID at `join`, but the Manager can override it during `Register` (when the same name+host already exists), and the agent then persists the Manager's id back into `agent.yaml`. This is why re-joining a PC doesn't create ghost machines.
6. **Health is computed, never stored.** `heartbeat_age > 15 s` = offline, `> 10 s` = warning. `machines.status` in the DB is vestigial.
7. **Scheduler math**, `scheduler_service.py:24-33` (dense one-liner style, so here it is expanded):
   `score = 0.40·(100−cpu%) + 0.35·(100−mem%) + 0.15·max(0, 100·(1 − running/100)) + 0.10·health`
   where `health ∈ {healthy:100, warning:40, offline:0, other:20}`, and the whole score is forced to `0` when the machine is offline. Selection = highest score among healthy/warning. `max_containers` is hard-coded to 100.
8. **Two config-freshness traps.** Manager env vars are read at *import* time (frozen dataclass defaults), and `VITE_*` vars are baked in at *build* time. Neither responds to a plain restart of a running container.
9. **Everything hinges on one shared token** compared with `!=` on insecure gRPC channels. The README says as much: LAN/VPN only.
10. **The `heartbeats` table grows without bound** and `list_machine_health` does an N+1 query over it on every poll. This is the first scaling wall you will hit.
11. **CORS is hard-coded** to `localhost`/`127.0.0.1` on ports 3000 and 5173. Serving the dashboard from any other origin (a real hostname, another port) requires editing `manager/app/main.py:62`.
12. **CWD matters a lot.** The SQLite path, the `agent.yaml` path, and the `manager.app.main:app` import string are all relative.

---

## 19. Explain This Project in 5 Minutes

Imagine you have three PCs with Docker installed and you want one web page to run containers on all of them. That is this project.

One PC runs the **Manager** — a FastAPI app you start with `uvicorn manager.app.main:app`. On startup it does three things: creates a SQLite file with four tables, opens a gRPC listener on port 9000, and then serves a REST API on port 8000 forever.

Every other PC runs an **Agent**. You run `orchestrator-agent join --manager <ip>:9000 --name machine-2 --token abc123` once, which just writes an `agent.yaml` with a freshly generated UUID. Then `orchestrator-agent start` boots the real thing: it connects to the local Docker Engine, opens its own gRPC listener on port 9001, and calls the Manager's `Register` RPC. During registration it figures out its own LAN IP with a neat trick — opening a UDP socket "toward" the Manager and reading back which local address the OS chose (`system_info.local_ip_for`). The Manager stores the machine in the `machines` table and may hand back a different, canonical UUID, which the agent writes back into its own `agent.yaml`. From then on a background thread sends a `Heartbeat` RPC every 5 seconds carrying CPU %, memory %, and the count of running containers. Each heartbeat updates the machine row and appends a row to the `heartbeats` table.

The **React dashboard** polls two endpoints every 5 seconds: `/api/health/machines` and `/api/scheduler/scores`. The first one computes health from heartbeat age — under 10 s is healthy, 10–15 s is warning, over 15 s is offline. The second one runs the weighted scheduler: it scores every machine as 40 % free CPU + 35 % free memory + 15 % spare container capacity + 10 % health, and picks the highest scorer among healthy/warning machines.

When you click **Deploy**, `Deploy.jsx` POSTs `{image, replicas, name_prefix, machine}` to `/api/deployments`. `routes.run()` branches on `machine`: `null` means "let the scheduler choose", `"all"` means "every active machine", a list means "these specific machines", and a plain string means "this one, if it's online". Whichever it is, it converges on `container_service.deploy()`, which double-checks the machine's heartbeat is fresh, generates a deployment UUID, builds a container name prefix like `web-3f9ac1`, and opens a gRPC connection to that agent's port 9001 with a 60-second timeout. The agent's `ContainerManager.create_containers()` pulls the image, then for each replica checks the name is free, creates the container with `restart_policy: no` and two `orchestrator.*` labels, starts it, and returns a serialized description. Back on the Manager, a `Deployment` row and one `ContainerRecord` per container are committed — deliberately *after* Docker confirmed, so a failure leaves no phantom rows. The JSON response flows back to the browser and renders in a results table.

Stopping a container works the same way in reverse, with one quirk: the Manager doesn't remember which machine a container is on, so `_find_container` asks *every* machine "do you have `web-3f9ac1-1`?" until one says yes, then sends `StopContainer` to that agent and marks the DB row `exited`.

The one design decision that explains everything else: **there is no reconciler**. No desired-state loop, no auto-restart, no self-healing. If a container crashes, it stays crashed. The API responses literally say "No reconciler will restart it." This is a control plane for *explicit human actions* — which is exactly why the code is small enough to read in an afternoon.

Two things to be careful about: the shared `CLUSTER_TOKEN` on insecure gRPC is the entire security model (LAN/VPN only), and the README's Streamlit dashboard no longer exists — `START_HERE.md` and `client/` are the current truth.
