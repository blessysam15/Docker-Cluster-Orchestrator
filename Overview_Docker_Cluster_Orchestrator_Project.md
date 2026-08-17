# Simple Docker Cluster Orchestrator

## Project Goal

Build a lightweight Docker-based container orchestrator capable of
managing multiple Docker hosts from a centralized management node. Each
Docker machine joins the cluster by running an agent. The management
server maintains cluster state and remotely controls containers on every
node.

Example:

``` bash
orchestrator run --machine 4 --replicas 4 --image nginx:latest
```

This command creates four `nginx` containers on Machine 4.

------------------------------------------------------------------------

# High-Level Architecture

``` text
                    +----------------------+
                    |   Management Server  |
                    |----------------------|
                    | Cluster Registry     |
                    | Scheduler            |
                    | State Manager        |
                    | REST/gRPC API        |
                    +----------+-----------+
                               |
                 +-------------+--------------+
                 |             |              |
          +------+----+  +-----+-----+  +-----+-----+
          | Agent-1   |  | Agent-2   |  | Agent-N   |
          +------+----+  +-----+-----+  +-----+-----+
                 |              |               |
            Docker API     Docker API      Docker API
                 |              |               |
             Docker Host    Docker Host    Docker Host
                 |              |               |
             Containers     Containers     Containers
```

------------------------------------------------------------------------

# Functional Requirements

## 1. Cluster Management

-   Multiple Docker machines can join the cluster.
-   Machines register with the manager.
-   Manager stores machine information.

Each machine should report:

-   Machine ID
-   Hostname
-   IP Address
-   CPU
-   Memory
-   Docker Version
-   Status
-   Running Containers

------------------------------------------------------------------------

## 2. Centralized Management

The management node should:

-   Register machines
-   Remove machines
-   Monitor machine health
-   Deploy containers
-   Maintain cluster state

------------------------------------------------------------------------

## 3. Agent

Every Docker host runs an Agent.

Responsibilities:

-   Connect to Docker Engine
-   Create containers
-   Delete containers
-   Start containers
-   Stop containers
-   List containers
-   Send heartbeat

------------------------------------------------------------------------

## 4. Container Deployment

Example:

``` bash
orchestrator run --machine 4 --replicas 4 --image nginx
```

Expected Result

``` text
Machine-4

web-1
web-2
web-3
web-4
```

------------------------------------------------------------------------

## 5. List Machines

``` bash
orchestrator machines
```

------------------------------------------------------------------------

## 6. List Containers

``` bash
orchestrator ps
```

Machine specific

``` bash
orchestrator ps --machine 4
```

------------------------------------------------------------------------

## 7. Container Lifecycle

Support:

``` text
run
start
stop
restart
rm
inspect
logs
ps
```

------------------------------------------------------------------------

## 8. Join Cluster

``` bash
orchestrator-agent join \
    --manager 192.168.1.10:9000 \
    --name machine-4 \
    --token abc123
```

------------------------------------------------------------------------

## 9. Heartbeat

Every five seconds:

Agent → Manager

Heartbeat includes:

-   CPU usage
-   Memory usage
-   Running containers
-   Health

------------------------------------------------------------------------

## 10. Desired State

Manager stores

Desired

``` text
Machine-4

4 nginx containers
```

Actual

``` text
Machine-4

3 nginx containers
```

Difference should be reported.

------------------------------------------------------------------------

## 11. Self Healing (Basic)

If Desired \> Actual

Manager creates missing containers.

------------------------------------------------------------------------

## 12. Scheduler

Initially

``` bash
orchestrator run --machine 4 ...
```

Later

``` bash
orchestrator run --replicas 4
```

Scheduler selects machine automatically.

------------------------------------------------------------------------

## 13. Networking

Support

``` bash
--network
--publish
```

------------------------------------------------------------------------

## 14. Volumes

Support

``` bash
--volume data:/data
```

------------------------------------------------------------------------

## 15. Environment Variables

Support

``` bash
--env KEY=value
```

------------------------------------------------------------------------

## 16. Logs

``` bash
orchestrator logs web-1
```

------------------------------------------------------------------------

## 17. Inspect

``` bash
orchestrator inspect web-1
```

------------------------------------------------------------------------

## 18. Persistent Storage

Use SQLite.

Tables

-   machines
-   containers
-   deployments
-   heartbeats

------------------------------------------------------------------------

## 19. Authentication

Initial

Cluster Token

Future

TLS / mTLS

------------------------------------------------------------------------

# CLI

``` bash
orchestrator machines

orchestrator run \
    --machine 4 \
    --replicas 4 \
    --image nginx

orchestrator ps

orchestrator inspect web-1

orchestrator logs web-1

orchestrator stop web-1

orchestrator start web-1

orchestrator restart web-1

orchestrator rm web-1
```

------------------------------------------------------------------------

# Agent CLI

``` bash
orchestrator-agent join \
    --manager 10.0.0.1:9000 \
    --name machine-4

orchestrator-agent status

orchestrator-agent leave
```

------------------------------------------------------------------------

# Technology Stack

  Component   Technology
  ----------- ------------
  Language    Go
  CLI         Cobra
  API         gRPC
  Docker      Docker SDK
  Database    SQLite
  Config      YAML
  Logging     slog / Zap
  Metrics     Prometheus

------------------------------------------------------------------------

# MVP Flow

``` text
CLI
 |
 | Run 4 Containers
 v
Manager
 |
 | Locate Machine-4
 |
 | gRPC
 v
Agent
 |
 | Docker SDK
 v
Docker Engine
 |
 +--> web-1
 +--> web-2
 +--> web-3
 +--> web-4
```

The first version should focus on centralized management, machine
registration, remote container lifecycle management, heartbeat
monitoring, and persistent cluster state. Advanced features such as
automatic scheduling, rolling updates, service discovery, and leader
election can be added in later releases.
