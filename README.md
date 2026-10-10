# SkillfyBank - Docker Production-Level Assignment #1

## Student Details

**Name:** Jayant Rajput

**email:** jayantrajput911@gmail.com

**Assignment:** Docker Production-Level Assignment #1

## Problem Statement

Re-architect and deploy the SkillfyBank banking microservices using Docker,
Docker Swarm, Kubernetes, and Istio with high availability, service
communication, traffic routing, rolling updates, health monitoring,
self-healing, and failure recovery.

## Solution Approach

This project implements the assignment progressively using:

- Docker
- Docker Hub
- Docker networking
- Docker Swarm
- Docker Stack
- Overlay networking
- Kubernetes
- Istio

The implementation is performed on AWS EC2 infrastructure.

## Architecture

The project contains three microservices:

- Account Service - Node.js
- Transaction Service - Spring Boot
- Notification Service - Python Flask

Further architecture and implementation details will be documented as
each assignment phase is completed.

## Prerequisites

- AWS account
- Ubuntu EC2 instances
- Docker Engine
- Docker Hub account
- Git
- GitHub account

## Repository Structure

```text
docker/
swarm/
kubernetes/
istio/
README.md

## Docker Swarm

### Swarm Architecture

The Docker Swarm cluster consists of:

- 3 Manager Nodes
- 2 Worker Nodes

The services are deployed as Docker Swarm services using Docker Stack.

### Swarm Services

| Service | Replicas | External Access |
|---|---:|---|
| Account Service | 2 | Port 3000 |
| Transaction Service | 2 | Internal port 8080 |
| Notification Service | 2 | Port 5000 |

### Overlay Network

An attachable Docker overlay network named `skillfybank-overlay`
is used for communication between the services.

### Stack Deployment

```bash
docker network create --driver overlay --attachable skillfybank-overlay

docker stack deploy -c stack.yml skillfybank


Manager 3 failure
        ↓
Manager 1 remained Leader
Manager 2 remained Reachable
        ↓
Transaction task on Manager 3 failed
        ↓
Swarm created replacement task
        ↓
Transaction returned to 2/2
        ↓
Overlay connectivity remained functional


### Worker Failure

After validating Manager high availability, `swarm-worker-1`
was deliberately stopped.

Before the failure, Account Service had two replicas:

```text
Account replica 1 → swarm-worker-1
Account replica 2 → swarm-worker-2

After swarm-worker-1 was stopped, Docker Swarm detected the
failed task and scheduled a replacement task on an available node.

The Account Service was restored to:

2/2 replicas running
Combined Failure Test

The failure simulation demonstrated:

Non-leader manager failure
Manager quorum preservation
Transaction Service task reallocation
Worker failure
Account Service task reallocation
Continued overlay-network connectivity
Restoration of the desired service replica counts


### Worker Failure and Account Service Reallocation

`swarm-worker-1` was deliberately stopped while hosting one
Account Service replica.

Before the failure:

Account replica 1 → swarm-worker-2
Account replica 2 → swarm-worker-1

After swarm-worker-1 failed, Docker Swarm automatically
created a replacement task on swarm-manager-2.

The resulting placement was:

Account replica 1 → swarm-worker-2
Account replica 2 → swarm-manager-2

The service returned to the desired state:

2/2 replicas running

The original task on swarm-worker-1 remained in the task history
with a Shutdown state.

This demonstrated Docker Swarm's self-healing and service
reallocation capability after worker node failure.

## Docker Healthcheck

A custom Docker HEALTHCHECK was added to the Notification Service.

The healthcheck verifies the application's `/health` endpoint from
inside the container:

```dockerfile
HEALTHCHECK --interval=10s --timeout=3s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health', timeout=2)" || exit 1

The Docker image was rebuilt and deployed as:
jayantrajput/skillfybank-notification:v2

The health status was verified using:
docker inspect <CONTAINER_ID> \
  --format '{{.State.Health.Status}}'

The resulting status was:
healthy

This demonstrates application-level health monitoring using Docker's
built-in HEALTHCHECK mechanism.

### Horizontal Pod Autoscaling (HPA) — Transaction Service

#### Objective

Configure Kubernetes Horizontal Pod Autoscaler (HPA) to automatically adjust the number of Transaction Service replicas based on CPU utilization.

#### Implementation

- Installed Metrics Server to provide CPU and memory metrics for Kubernetes pods and nodes.
- Configured an HPA targeting the `transaction-service` Deployment.
- Set the target average CPU utilization to 50%.
- Configured a minimum of 2 replicas and a maximum of 5 replicas.
- Generated application traffic against the Transaction Service to exercise autoscaling.

#### Configuration

Manifest: `kubernetes/hpa/transaction-hpa.yaml`

Key settings:

| Parameter | Value |
|---|---|
| Target Deployment | `transaction-service` |
| Minimum replicas | 2 |
| Maximum replicas | 5 |
| Target CPU utilization | 50% |
| Scaling API | `autoscaling/v2` |

#### Verification

Check HPA status:

kubectl get hpa -n skillfybank


Check CPU and memory usage:


kubectl top pods -n skillfybank
kubectl top nodes

Check Transaction Service replicas:

kubectl get deployment transaction-service -n skillfybank
kubectl get pods -n skillfybank -l app=transaction-service


Inspect HPA conditions and events:


kubectl describe hpa transaction-service-hpa -n skillfybank


#### Observed Results

The HPA was successfully created, Metrics Server reported pod resource usage, and the Transaction Service scaled from its initial 2 replicas to 5 replicas.

Example verification output:

```text
NAME                      REFERENCE                        TARGETS       MINPODS   MAXPODS   REPLICAS
transaction-service-hpa   Deployment/transaction-service   cpu: 0%/50%   2         5         5
```

The Transaction Service Deployment reported:

```text
NAME                  READY   UP-TO-DATE   AVAILABLE
transaction-service   5/5     5            5
```

All five replicas were ready at the time of verification.

**Note:** The observed CPU utilization was 0% at the time of the final check. The five-replica state confirms the scale-up occurred, but a complete autoscaling test should also verify scale-down after load stops.

#### Cleanup

Delete the temporary load generator if it is still running:


kubectl delete pod transaction-load -n skillfybank --ignore-not-found


The HPA remains configured to manage the Transaction Service between 2 and 5 replicas.
