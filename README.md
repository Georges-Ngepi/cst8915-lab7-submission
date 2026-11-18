# CST8915 Lab 7 – Introduction to Kubernetes Basics

**Student:** Georges Ndongmo Ngepi
**Course:** CST8915 Full-stack Cloud-native Development

## Demo video

https://youtu.be/8TKyuIB9qD0

The video shows:
1. Deploying the Algonquin Pet Store with `algonquin-pet-store-all-in-one.yaml` on Azure Kubernetes Service (AKS).
2. The RabbitMQ configuration problem: orders are waiting in the queue, I delete the RabbitMQ Pod, and the orders are lost.

## Environment

- AKS cluster, Free tier, **1 node** (`Standard_D2s_v6`), region **Sweden Central**
  (my subscription cannot use the usual VM sizes in Canada Central)
- Cluster created with the Azure CLI (`az aks create`), managed with `kubectl` from Git Bash on Windows
- All Azure resources were deleted after the lab

## What I practised

| Object | With `kubectl` | With YAML |
| --- | --- | --- |
| Pod | `kubectl run my-pod --image=nginx --port=80` | `my-pod.yaml` + `kubectl apply -f` |
| Deployment | `kubectl create deployment my-deployment --image=nginx` | `my-deployment.yaml` (3 replicas) |
| Service | `kubectl expose deployment my-deployment --type=LoadBalancer --port=80` | `my-service.yaml` (LoadBalancer, selector `app: nginx`) |
| Whole app | – | `algonquin-pet-store-all-in-one.yaml` (4 Deployments + 4 Services, separated by `---`) |

`kubectl` commands are fast for testing. YAML files are better for real work: they can be saved in Git, reviewed, and applied again in the same way.

## The RabbitMQ configuration problem

### What I observed

1. I placed orders in the store. RabbitMQ showed `order_queue` with **MESSAGES_BEFORE** messages
   (`kubectl exec deploy/rabbitmq -- rabbitmqctl list_queues name durable messages`).
2. `kubectl get pvc` returned **No resources found**: RabbitMQ has no persistent storage.
3. I deleted the RabbitMQ Pod (`kubectl delete pod -l app=rabbitmq`).
4. The Deployment created a **new** Pod automatically (self-healing).
5. In the new Pod, the queue was **empty / gone**. All waiting orders were **lost**.

### Is RabbitMQ stateless or stateful?

RabbitMQ is a **stateful** application.

A stateless service (like `product-service` or `store-front`) keeps nothing important inside the container. Any copy can answer any request, so Kubernetes can kill and replace it at any time.

RabbitMQ is different. Its job is to **store messages** until a consumer reads them. It also stores queue definitions, users and its node identity in `/var/lib/rabbitmq`. That data is the "state". If the state is lost, the application loses its main value.

### Implications of running RabbitMQ without persistent storage

In the YAML file, RabbitMQ is a **Deployment** with **no volume**. Its data is written to the container's own filesystem, which is temporary.

- **Data loss:** every order waiting in `order_queue` is lost when the Pod is replaced.
- **"Durable" is not enough:** the Order Service declares a durable queue and sends persistent messages. RabbitMQ does save them to disk, but the disk is the container's temporary disk. It disappears with the Pod.
- **No stable identity:** a Deployment gives the Pod a random name (`rabbitmq-5d4b559c94-xxxxx`). RabbitMQ uses the hostname as its node name, so even with a volume, a new Pod would start as a **new node** and might not find its old data.
- **Silent failure:** the store still says "Order placed successfully", and the Order Service already returned success. Nobody is warned that the orders disappeared. For a real shop, this means lost sales and unhappy customers.
- **Single point of failure:** `replicas: 1` and no clustering. During the restart, new orders can fail.

### What happens when the RabbitMQ Pod is deleted or restarted

1. Kubernetes stops the old container. Its temporary filesystem is deleted.
2. The Deployment sees 0 Pods instead of 1 and creates a new Pod (new name, new empty filesystem).
3. RabbitMQ starts as a fresh broker: no queues and no messages. Only the user from the environment variables (`myuser`) is created again.
4. The **Service** `rabbitmq` keeps the same name and IP, so new connections reach the new Pod. The queue is created again when the Order Service sends the next order, but the old messages are gone forever.

The same thing happens when a node is restarted, upgraded or scaled down, or when the Pod is evicted for lack of memory.

In Lab 5 (Docker Compose), I did not have this problem because the Compose file gave RabbitMQ a **named volume** (`rabbitmq-data:/var/lib/rabbitmq`) and a **fixed hostname** (`hostname: rabbitmq`). The Kubernetes YAML in this lab has neither.

### Potential solutions (research-based)

1. **Use a StatefulSet with a PersistentVolumeClaim (PVC).**
   A StatefulSet gives each Pod a stable name (`rabbitmq-0`) and its own persistent volume. On AKS, the PVC can use an **Azure Disk** (storage class `managed-csi`). The data survives Pod restarts, and the stable name keeps the same RabbitMQ node identity.
   Example (not tested in this lab, to keep costs low):

   ```yaml
   apiVersion: apps/v1
   kind: StatefulSet
   metadata:
     name: rabbitmq
   spec:
     serviceName: rabbitmq
     replicas: 1
     selector:
       matchLabels:
         app: rabbitmq
     template:
       metadata:
         labels:
           app: rabbitmq
       spec:
         containers:
           - name: rabbitmq
             image: rabbitmq:3-management
             ports:
               - containerPort: 5672
               - containerPort: 15672
             volumeMounts:
               - name: rabbitmq-data
                 mountPath: /var/lib/rabbitmq
     volumeClaimTemplates:
       - metadata:
           name: rabbitmq-data
         spec:
           accessModes: ["ReadWriteOnce"]
           storageClassName: managed-csi
           resources:
             requests:
               storage: 1Gi
   ```

2. **Use the RabbitMQ Cluster Kubernetes Operator.**
   The official operator creates RabbitMQ as a StatefulSet with persistent storage, and it can run a cluster of 3 nodes. With **quorum queues**, messages are copied to several nodes, so one node can fail without losing data.

3. **Keep durable queues and persistent messages**, and add **publisher confirms** and **consumer acknowledgements** (already used in the Order Service and in our midterm analytics consumer). These protect messages in flight, but they only work if the disk is persistent.

4. **Store credentials in a Kubernetes Secret** instead of plain text in the YAML (`myuser` / `mypassword` are visible to anyone who reads the file). This is not about data loss, but it is another weakness of this configuration.

5. **Use a managed message broker**, such as **Azure Service Bus** (see below).

### Does Azure Service Bus solve the problem?

**Mostly yes, for the data-loss problem.**

- Azure Service Bus is a **fully managed** message broker (PaaS). Azure stores the messages on durable, replicated storage. There is no Pod, disk or StatefulSet for us to manage, so deleting or restarting a Pod in our cluster cannot delete the messages.
- It provides queues, at-least-once delivery, dead-letter queues and message locks. Microsoft handles patching and availability (with an SLA), and the Premium tier adds more isolation and geo-disaster recovery.

**But it is not a drop-in replacement:**

- **Code changes:** Service Bus uses **AMQP 1.0** and the Azure SDK. RabbitMQ uses AMQP 0-9-1 (the `amqplib` and `pika` libraries in our services). The Order Service and any consumer must be rewritten to use the Service Bus SDK or another AMQP 1.0 client.
- **Cost:** you pay for each operation or for the tier. It is not free like a container we run ourselves.
- **Vendor lock-in:** the application now depends on Azure. It cannot run the same way on a laptop or another cloud.
- **Local development:** we cannot simply run it in Docker Compose like RabbitMQ (we would need a cloud namespace or an emulator).

**Conclusion:** Azure Service Bus solves the persistence and availability problems by moving the "state" to a managed service. If we want to keep RabbitMQ (portable, free, same code), the correct fix is a StatefulSet with a persistent volume, or the RabbitMQ Operator.

## Lessons learned

- A Deployment is good for **stateless** services. **Stateful** services (brokers, databases) need persistent storage and a stable identity (StatefulSet + PVC) or a managed service.
- Self-healing creates a **new** Pod. It does not bring back the old Pod's data.
- `durable` and `persistent` in RabbitMQ only help if the disk itself survives.
- Git Bash on Windows converts Linux paths in `kubectl exec`; `MSYS_NO_PATHCONV=1` fixes it.
- To control cost, I used a single node and deleted the resource group right after recording.

## References

- Kubernetes. (n.d.). *StatefulSets*. https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/
- Kubernetes. (n.d.). *Persistent volumes*. https://kubernetes.io/docs/concepts/storage/persistent-volumes/
- RabbitMQ. (n.d.). *Persistence configuration*. https://www.rabbitmq.com/docs/persistence-conf
- RabbitMQ. (n.d.). *RabbitMQ Cluster Kubernetes Operator*. https://www.rabbitmq.com/kubernetes/operator/operator-overview
- Microsoft. (n.d.). *Storage options for applications in Azure Kubernetes Service (AKS)*. https://learn.microsoft.com/azure/aks/concepts-storage
- Microsoft. (n.d.). *What is Azure Service Bus?* https://learn.microsoft.com/azure/service-bus-messaging/service-bus-messaging-overview
