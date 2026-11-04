# Kafka playground: basic services

This is the infrastructure-only phase of the root README plan: three Kafka
brokers in KRaft mode and Provectus Kafka UI. OIDC, IAM, ingress and
TLS are intentionally not configured yet.

## Deploy

Run from the repository root with `kubectl` connected to the intended cluster:

```bash
kubectl config current-context
kubectl apply -f src/kafka-poc/namespace.yaml
kubectl apply -f src/kafka-poc/
kubectl rollout status statefulset/kafka -n kafka-poc --timeout=300s
kubectl rollout status deployment/kafka-ui -n kafka-poc --timeout=300s
kubectl get pods,svc,pvc -n kafka-poc
```

Each broker requests a 10 GiB persistent volume using the dedicated
`kafka-poc-nfs` storage class (30 GiB total). The volumes survive Pod replacement. Broker IDs
are 1, 2 and 3 for `kafka-0`, `kafka-1` and `kafka-2`, respectively.

`storage-class.yaml` uses this lab's existing NFS subdirectory provisioner.
It intentionally omits `pathPattern`, so the provisioner's default directory
includes the unique PV name. The shared `nfs-client` class uses only namespace
and PVC name, which would reuse old NFS directories after a namespace reset.
The dedicated class does not modify the shared class. On a different cluster,
adapt the provisioner before deploying. An existing StatefulSet cannot change
its volume claim template storage class in place; use the approved reset
procedure below if discarding the old data is intended.

`kafka-0` uses a stable broker ID, PVC and single-controller quorum so
single-node data can be reused during an upgrade without reformatting KRaft storage.
The other two Pods are broker-only. This is a development POC, not a highly
available controller setup; losing `kafka-0` loses the active controller.
Broker Pods prefer different Kubernetes nodes but may share a node when needed.
This lab's worker has insufficient reserved capacity, so the broker and demo
consumer manifests explicitly tolerate the control-plane `NoSchedule` taint.
This is a lab-only scheduling choice, not a production recommendation; remove
that toleration when dedicated workers have sufficient capacity.

New automatically created topics default to three partitions and replication
factor three. Existing topics, including `orders.events` and
`__consumer_offsets`, keep their original partition and replica assignments.
Changing defaults does not redistribute existing data or increase its replicas.
On a fresh deployment, `__consumer_offsets` is created with 50 partitions and
replication factor three when consumers first need the group coordinator.

## Open the UI

```bash
kubectl port-forward -n kafka-poc service/kafka-ui 8080:80 --address=127.0.0.1
```

Keep the command running and open <http://localhost:8080>. If `kubectl` runs on a
remote machine, forward port 8080 with your SSH client or VS Code Ports view.
If port 8080 is occupied, use `18080:80` and open <http://localhost:18080>.

Select the `idp-kafka` cluster. There is no login. The UI can create/delete
topics and publish/read messages, so do not expose it through a public ingress,
NodePort, or a port-forward bound to `0.0.0.0`. ClusterIP is not an access-control
boundary: other workloads with network access can also reach the UI and the
plaintext, unauthenticated Kafka listener.

## Test through the UI

1. Open **Topics**, choose **Add a Topic**, and create `orders.keyed` with
   three partitions and replication factor `3`. If it already exists, reuse it.
2. Open that topic and use **Produce Message** to publish this JSON value
   (string serialization is sufficient; no Schema Registry is required):

   ```json
   {
     "orderId": "12345",
     "status": "CREATED"
   }
   ```

3. Open **Messages**, select the earliest offset, and start reading messages.
   Verify the JSON, partition and offset are displayed.
4. Publish another value and refresh/read again to verify multiple records.

Kafka clients running in `kafka-poc` can bootstrap through `kafka:9092`.
The brokers advertise `kafka-0.kafka:9092`, `kafka-1.kafka:9092` and
`kafka-2.kafka:9092`, which resolve inside that namespace.
External Kafka client access is not configured; UI access does not require it.

## Test message keys

The deployed Provectus UI v0.7.2 requires an explicit partition when producing.
Its producer API rejects an omitted partition and does not support `-1` as an
automatic-partition sentinel. Use the Kafka CLI producer to test automatic
key-based partitioning, then inspect the results in the UI:

```bash
kubectl exec -i kafka-0 -n kafka-poc -- \
  /opt/kafka/bin/kafka-console-producer.sh --bootstrap-server kafka:9092 \
  --topic orders.keyed --property parse.key=true --property 'key.separator=|' <<'EOF'
customer-a|{"orderId":"A1","status":"CREATED"}
customer-b|{"orderId":"B1","status":"CREATED"}
customer-a|{"orderId":"A2","status":"PAID"}
EOF
```

Inspect the key, partition and offset in **Messages**. With the default
key-based partitioner and unchanged partition count, the two `customer-a`
records go to the same partition and preserve their producer send order.
Different keys may hash to the same partition; they are not guaranteed separate
partitions. Producing through this UI explicitly chooses a partition and
bypasses key-based selection, even when a key is supplied.
Adding partitions can change the mapping of a key for future messages.

## Test consumer groups

Create `orders.keyed` as above, then start the optional demo consumers:

```bash
kubectl apply -f src/kafka-poc/examples/consumer-group.yaml
kubectl rollout status deployment/kafka-demo-consumer -n kafka-poc --timeout=300s
kubectl exec kafka-0 -n kafka-poc -- \
  /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group playground-workers --members --verbose
```

In the UI, open **Consumers** / **Consumer Groups** and select
`playground-workers`. Once the group is stable, the three consumer processes
each own one partition of `orders.keyed`. Produce more keyed messages in the UI,
then inspect the consumer output and committed offsets/lag:

```bash
kubectl logs -n kafka-poc -l app=kafka-demo-consumer --prefix --tail=20
kubectl exec kafka-0 -n kafka-poc -- \
  /opt/kafka/bin/kafka-consumer-groups.sh --bootstrap-server kafka:9092 \
  --describe --group playground-workers
```

Scale the Deployment to observe rebalancing:

```bash
kubectl scale deployment kafka-demo-consumer -n kafka-poc --replicas=2
# After rebalancing, the two consumers own 2 and 1 partitions.
kubectl scale deployment kafka-demo-consumer -n kafka-poc --replicas=4
# After rebalancing, one consumer is idle: there are only three partitions.
kubectl scale deployment kafka-demo-consumer -n kafka-poc --replicas=3
```

Wait for stable membership after each scale operation. Consumers in one group
share partitions; a second group with a different group ID independently reads
the same topic. Broker count controls storage/replication capacity, not the
number of active consumers per topic within one group. The UI message browser
is not a substitute for these long-running group members.

Remove only the demo consumers when finished:

```bash
kubectl delete -f src/kafka-poc/examples/consumer-group.yaml
```

## Troubleshooting

```bash
kubectl describe pod kafka-0 -n kafka-poc
kubectl get events -n kafka-poc --sort-by=.lastTimestamp
kubectl logs kafka-0 -n kafka-poc --tail=100
kubectl logs deployment/kafka-ui -n kafka-poc --tail=100
kubectl get pvc -n kafka-poc
```

Pending PVCs require the NFS provisioner referenced by `kafka-poc-nfs`. `ImagePullBackOff`
requires cluster access to the configured container registries. A healthy UI
process alone does not prove Kafka connectivity: confirm `idp-kafka` is online
and complete the publish/read test above.

Stop the port-forward with Ctrl+C. To remove the application while retaining
Kafka data:

```bash
kubectl delete deployment kafka-ui kafka-demo-consumer -n kafka-poc --ignore-not-found
kubectl delete statefulset kafka -n kafka-poc
kubectl delete service kafka kafka-ui -n kafka-poc
```

Do not delete the namespace or its PVC unless you intend to remove the stored
test data. OIDC and controlled network exposure should be addressed before
sharing this playground beyond a trusted development environment.

## Reset all playground data

Only use this procedure when no existing playground data needs to be retained.
It deletes the entire `kafka-poc` namespace, including its Kafka PVCs, topics,
messages and consumer group progress, and provisions fresh volumes on redeploy.
Deleting Pods alone does not reset Kafka.

```bash
kubectl config current-context
kubectl delete namespace kafka-poc --wait=true --timeout=300s
kubectl apply -f src/kafka-poc/namespace.yaml
kubectl apply -f src/kafka-poc/
kubectl rollout status statefulset/kafka -n kafka-poc --timeout=300s
kubectl rollout status deployment/kafka-ui -n kafka-poc --timeout=300s
kubectl exec kafka-0 -n kafka-poc -- \
  /opt/kafka/bin/kafka-topics.sh --bootstrap-server kafka:9092 \
  --create --if-not-exists --topic orders.keyed --partitions 3 --replication-factor 3
kubectl apply -f src/kafka-poc/examples/consumer-group.yaml
kubectl rollout status deployment/kafka-demo-consumer -n kafka-poc --timeout=300s
```

Wait for `playground-workers` to become stable, then use the message-key test
above. The former `orders.events` topic is not recreated by this procedure.
Restart the UI port-forward after namespace replacement.

The dedicated storage class uses the `Retain` reclaim policy. Old PVs and their
underlying NFS data remain retained/released and are not reused automatically;
this reset is not secure data erasure or storage reclamation. Any physical
cleanup requires separately verifying the exact old volumes and NFS paths.
