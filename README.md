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
