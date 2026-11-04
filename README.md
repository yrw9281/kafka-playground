# Kafka Developer Playground POC

## Overview

This Proof of Concept (POC) demonstrates how the IDP (Internal Developer Platform) can provide a self-service Kafka experience for developers through a modern web-based interface integrated with the organization's Identity Provider (IdP).

The objective is to validate the following capabilities:

- Kafka cluster deployment on Kubernetes
- Web-based Kafka management and testing UI
- Single Sign-On (SSO) through the corporate IAM/IdP
- Topic exploration and message inspection
- Message publishing and consumption for development testing
- Foundation for future self-service Kafka capabilities in the IDP Platform

---

## Architecture

```text
+---------------------+
|     IDP Portal      |
+----------+----------+
           |
           | OIDC
           |
           v
+---------------------+
|   Corporate IAM     |
|      (IdP)          |
+----------+----------+
           |
           v
+---------------------+
|      Kafka UI       |
|    (Provectus)      |
+----------+----------+
           |
           v
+---------------------+
|       Kafka         |
|   (KRaft Mode)      |
+---------------------+
```

---

## Components

### Kafka

This POC deploys a single-node Kafka cluster running in KRaft mode, eliminating the need for ZooKeeper and simplifying deployment.

Responsibilities:

- Topic management
- Message storage
- Producer and Consumer testing
- Foundation for future event-driven services

### Kafka UI

The POC uses Provectus Kafka UI as the web interface.

Features:

- Browse Topics
- View Messages
- Inspect Consumer Groups
- Publish Test Messages
- Search Records
- Manage Kafka Resources

### Identity Provider (IdP)

Kafka UI is integrated with the organization's existing IAM solution using OpenID Connect (OIDC).

Benefits:

- Single Sign-On (SSO)
- Centralized authentication
- Reusable platform security model
- Consistent user experience

---

## Deployment

Deploy all resources into the `kafka-poc` namespace.

```bash
kubectl apply -f namespace.yaml
kubectl apply -f kafka.yaml
kubectl apply -f kafka-service.yaml
kubectl apply -f kafka-ui.yaml
kubectl apply -f kafka-ui-service.yaml
kubectl apply -f ingress.yaml
```

Verify deployment:

```bash
kubectl get pods -n kafka-poc
```

Expected output:

```text
NAME                       READY   STATUS
kafka-0                    1/1     Running
kafka-ui-xxxxxxxxxx-xxxxx  1/1     Running
```

---

## Access

Once deployment is complete:

```text
https://kafka.idp.lab.company.com
```

Authentication is delegated to the corporate IdP through OIDC.

---

## POC Validation Scenarios

### Scenario 1: Authentication

Objective:

Verify OIDC integration.

Expected Result:

- User is redirected to corporate login page
- Authentication succeeds
- User is redirected back to Kafka UI

---

### Scenario 2: Topic Management

Create a topic:

```text
orders.events
```

Verify:

- Topic is created successfully
- Topic appears in Kafka UI
- Partition information is visible

---

### Scenario 3: Publish Message

Publish a test event:

```json
{
  "orderId": "12345",
  "status": "CREATED"
}
```

Verify:

- Message is accepted
- Message appears under Topic Messages

---

### Scenario 4: Consume Message

Use Kafka UI Message Browser.

Verify:

- Messages can be consumed successfully
- Offset information is displayed
- Message content is correctly rendered

---

### Scenario 5: Role-Based Access Control

Future validation with IAM group mapping:

| IAM Group | Platform Role |
|------------|--------------|
| idp-kafka-admin | Admin |
| idp-kafka-developer | Read/Write |
| idp-kafka-viewer | Read Only |

Expected Result:

- Different groups receive different permissions
- Access control is centrally managed through IAM

---

## Future Roadmap

### Phase 1 - Infrastructure POC

- Kubernetes Deployment
- Kafka (KRaft)
- Kafka UI
- OIDC Authentication

### Phase 2 - Self-Service APIs

Provide Topic lifecycle management APIs.

Example:

```http
POST /api/kafka/topics
```

Request:

```json
{
  "name": "payment-events",
  "partitions": 3,
  "retentionDays": 7
}
```

Capabilities:

- Create Topic
- Update Topic Configuration
- Delete Topic
- Audit Operations

---

### Phase 3 - IAM Integration

Introduce role mappings:

```text
IAM Group
      ↓
Platform Role
      ↓
Kafka Permissions
```

Example:

```text
idp-kafka-admin
idp-kafka-developer
idp-kafka-viewer
```

---

### Phase 4 - Event Playground

Provide self-service developer testing capabilities.

Features:

- Topic Explorer
- Publish Events
- Consume Events
- Consumer Group Viewer
- Schema Validation
- Event Replay
- Dead Letter Queue Inspection

---

## Success Criteria

The POC is considered successful when:

- Kafka runs successfully on Kubernetes
- Kafka UI successfully connects to Kafka
- OIDC authentication works end-to-end
- Developers can browse Topics
- Developers can create Topics
- Developers can publish Messages
- Developers can read Messages
- Architecture can be integrated into the IDP Platform

---

## Long-Term Vision

Transform Kafka from an infrastructure component into a first-class platform capability within the Internal Developer Platform.

```text
IDP Platform
└── Messaging
    ├── Kafka Clusters
    ├── Topic Management
    ├── ACL Management
    ├── Event Playground
    ├── Schema Registry
    ├── Consumer Monitoring
    ├── Audit Logging
    └── Governance Controls
```

Developers should be able to discover, provision, test, and manage Kafka resources through a unified platform experience without direct infrastructure access.

This POC serves as the first milestone toward delivering Kafka as a Platform Service within the IDP ecosystem.