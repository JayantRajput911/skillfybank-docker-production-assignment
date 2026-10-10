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

### Broken Deployment Rollout and Rollback

#### Objective

Simulate a failed Kubernetes Deployment rollout, diagnose the problem, and recover the application using Deployment rollback.

#### Failure Simulation

An invalid image tag was deliberately configured for the Transaction Service:

```bash
kubectl set image deployment/transaction-service \
  transaction-service=jayantrajput/skillfybank-transaction:broken-tag \
  -n skillfybank
```

The rollout was monitored using:

```bash
kubectl rollout status deployment/transaction-service \
  -n skillfybank --timeout=90s
```

#### Troubleshooting

The following commands were used to investigate the failed rollout:

```bash
kubectl get deployment transaction-service -n skillfybank
kubectl get pods -n skillfybank -l app=transaction-service
kubectl describe deployment transaction-service -n skillfybank
kubectl describe pod <FAILED_POD_NAME> -n skillfybank
kubectl logs deployment/transaction-service -n skillfybank --tail=50
```

The expected failure is an image-pull error, such as `ErrImagePull` or `ImagePullBackOff`, caused by the nonexistent image tag. Kubernetes pod Events provide the key diagnostic information when the container cannot start.

#### Recovery

The previous working Deployment revision was restored:

```bash
kubectl rollout undo deployment/transaction-service -n skillfybank
kubectl rollout status deployment/transaction-service -n skillfybank
```

Recovery was verified by checking the Deployment image, pod readiness, and HPA status.

#### Outcome

This exercise demonstrates how an invalid image reference can block a rollout, how Kubernetes Events and Deployment descriptions help identify the root cause, and how `kubectl rollout undo` restores the previous Deployment revision.

The failed image tag was used only for testing and should not be retained as the active Deployment image.


### Node Taints and Tolerations — Notification Service

#### Objective

Restrict Notification Service pods to a designated Kubernetes worker node using node labels, taints, tolerations, and a node selector.

#### Configuration

The designated node is `k8s-worker-2`.

A node label identifies it as suitable for Notification Service workloads:

```bash
kubectl label node k8s-worker-2 workload=notification --overwrite
```

A taint prevents pods without a matching toleration from being scheduled on the node:

```bash
kubectl taint nodes k8s-worker-2 dedicated=notification:NoSchedule
```

The Notification Service Deployment uses:

- **Node selector:** `workload=notification`
- **Toleration:** `dedicated=notification:NoSchedule`

Together, these settings allow Notification Service pods to run on the designated node and prevent this Deployment from being scheduled on other nodes.

#### Verification

```bash
kubectl describe node k8s-worker-2
kubectl get pods -n skillfybank -l app=notification-service -o wide
kubectl get nodes --show-labels
```

#### Outcome

The Notification Service is configured to run only on `k8s-worker-2`. The node's taint discourages unrelated workloads from being scheduled there unless they have an appropriate toleration. Kubernetes system components and other workloads with their own tolerations may still be present.
