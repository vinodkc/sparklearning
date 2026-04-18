# Spark on Kubernetes: Pods, Namespaces, and Ephemeral Executors

This is the story of how Spark runs on Kubernetes—the container orchestration platform that has become the dominant infrastructure layer in modern cloud deployments. Unlike YARN, which was built alongside Hadoop with Spark as a primary tenant, Kubernetes is a general-purpose container scheduler that didn't originally know anything about Spark. The native Kubernetes support added to Spark 2.3 changed this, making Kubernetes a first-class cluster manager. Understanding how Spark uses Kubernetes—pods for drivers and executors, namespaces for isolation, resource limits, and the challenges of ephemeral executor storage—equips you to run and debug Spark in any cloud-native environment.

---

## The Kubernetes resource model

In Kubernetes, the fundamental execution unit is a **Pod**: one or more containers that share a network namespace and (optionally) storage volumes, scheduled together on the same physical node. A pod is ephemeral: it runs until it completes or fails, and Kubernetes may reschedule it on a different node if the node fails.

Kubernetes organizes resources into **namespaces**: isolated environments within a cluster, each with its own resource quotas, RBAC policies, and networking. A `data-engineering` namespace might have a quota of 200 cores and 800GB RAM; applications in that namespace cannot exceed these limits.

> **Kubernetes is like a container port with a port authority.** Each ship (pod) carries containers (Docker containers) and docks at whatever berth (node) the port authority (scheduler) assigns. Ships are ephemeral—they load, operate, and leave. The port is divided into zones (namespaces) with different rules and capacity allocations. The port authority tracks every ship in every zone and enforces the zone's rules.

---

## How Spark submits to Kubernetes

With `--master k8s://https://kubernetes-api-server:443`, spark-submit talks directly to the Kubernetes API Server:

```bash
spark-submit \
  --master k8s://https://k8s-api:443 \
  --deploy-mode cluster \
  --name my-spark-app \
  --conf spark.kubernetes.namespace=data-engineering \
  --conf spark.kubernetes.container.image=my-registry/spark:3.5.0 \
  --conf spark.executor.instances=10 \
  local:///opt/app/my-job.jar
```

spark-submit uses the Kubernetes client library to submit a **driver pod spec** to the Kubernetes API Server. Kubernetes schedules the driver pod onto a node. Once the driver pod is running:

1. The driver JVM starts, `SparkContext` initializes.
2. The Spark Kubernetes `SchedulerBackend` creates **executor pod specs** and submits them to the Kubernetes API Server.
3. Kubernetes schedules executor pods onto available nodes (subject to namespace quotas, node selectors, and affinity rules).
4. Each executor pod starts, the JVM comes up, and the executor registers with the driver.

---

## Driver and executor pods

**Driver pod:**
- `metadata.name`: `my-spark-app-driver` (or random suffix for uniqueness)
- `spec.containers[0].resources.requests/limits`: sets the driver's CPU and memory
- `spec.serviceAccountName`: the Kubernetes service account with permission to create/delete executor pods
- `spec.restartPolicy: Never`: if the driver pod fails, it's not restarted (unlike a Deployment); Spark on K8s doesn't support driver restart natively (unlike YARN's AM restart).

**Executor pods:**
- Created dynamically by the driver using the Kubernetes API (via the service account)
- Named `my-spark-app-XXXXXXXXXX-exec-N` (one per executor)
- `spec.containers[0].resources.requests.memory`: `spark.executor.memory` + `spark.executor.memoryOverhead`
- `spec.containers[0].resources.limits.memory`: same as requests (Kubernetes enforces the limit strictly)
- `spec.containers[0].resources.requests.cpu`: `spark.executor.cores / spark.kubernetes.executor.request.cores` (can differ from cores to allow oversubscription)

When an executor completes its work or is idle (dynamic allocation), the driver deletes the executor pod via the Kubernetes API. Kubernetes terminates the pod and releases its resources.

> **Executor pods are like temporary contract workers who are hired for a project and let go when the project ends.** The project manager (driver) files the contract with HR (Kubernetes API), the worker shows up (pod starts), works on their assigned tasks, and when the project is complete, the contract is terminated. No long-term employment—pure project-based resource use.

---

## Container images: the key deployment artifact

Unlike YARN (which distributes JARs to a shared HDFS), Kubernetes requires that all dependencies are packaged into a **container image** (Docker image). The Spark container image contains:
- The JVM
- Spark's JARs and scripts
- Your application JAR and dependencies
- Any Python dependencies (for PySpark)
- Configuration files

Spark provides an `entrypoint.sh` that handles both driver and executor startup inside the container. The same image is used for both driver and executor pods; the startup mode is controlled by environment variables injected by Spark's Kubernetes integration.

Building and pushing a custom Spark image:
```bash
./bin/docker-image-tool.sh -r my-registry -t 3.5.0 build
./bin/docker-image-tool.sh -r my-registry -t 3.5.0 push
```

The image is pulled from the registry when each pod starts. Image pull time adds to the startup latency—cached images (already pulled on the node) start much faster.

---

## Ephemeral executors and shuffle data

One of Kubernetes' most significant challenges for Spark is that executor pods are **ephemeral**—they can be evicted, rescheduled, or terminated at any time. Shuffle data (written by map tasks to local disk) is stored in the executor pod's local storage. If an executor pod is evicted mid-job:

1. The shuffle data it wrote is gone (the pod's ephemeral disk is deleted when the pod terminates).
2. Downstream reduce tasks that need to fetch that shuffle data will fail with a `FetchFailedException`.
3. Spark retries the stage, which re-runs the map tasks to regenerate the shuffle data.

This is correct behavior, but it can be expensive: entire stages must re-run to regenerate lost shuffle data. For jobs with large shuffles, executor eviction can dramatically increase job time.

The solution is the **External Shuffle Service (ESS)** for Kubernetes: shuffle data is written to a persistent volume or a dedicated ESS pod rather than the executor's local ephemeral disk. When an executor is evicted, the shuffle data survives, and downstream tasks can still fetch it. Kubernetes ESS support has been available since Spark 3.x but requires additional setup (a DaemonSet or sidecar for the shuffle service).

> **Executor ephemeralness is like taking notes on a whiteboard that disappears when you leave the room.** If you return to the room (retry the stage), you have to rewrite everything from scratch. The External Shuffle Service is a persistent notebook: your notes survive even after you leave the room.

---

## Namespaces, RBAC, and resource quotas

Kubernetes namespaces provide multi-tenancy. A typical setup:
- Each team or environment gets a namespace: `data-eng-prod`, `data-eng-dev`, `ml-team`.
- Each namespace has a **ResourceQuota**: total CPU, memory, and pod count limits.
- The Spark driver's **service account** has RBAC permissions to create/list/delete pods and services within its namespace.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["create", "get", "list", "watch", "delete"]
```

Without the right service account permissions, the driver can't create executor pods, and the job fails immediately with a Kubernetes API authorization error.

---

## Monitoring Spark on Kubernetes

The Spark UI runs on the driver pod at port 4040. To access it:
```bash
kubectl port-forward pod/my-spark-app-driver 4040:4040 -n data-engineering
# Then browse to http://localhost:4040
```

For completed jobs, the History Server must have access to the event log (typically written to S3 or a PVC):
```
spark.eventLog.enabled=true
spark.eventLog.dir=s3a://my-bucket/spark-events/
```

Pod logs are accessible via `kubectl logs pod/my-spark-app-exec-1 -n data-engineering` or through a centralized logging stack (Loki, Elasticsearch, CloudWatch).

---

## Bringing it together

Spark on Kubernetes uses the Kubernetes API as the cluster manager: the driver creates and deletes **executor pods** dynamically, communicating with the Kubernetes API Server. Every component—driver and executors—runs as a **pod** inside a **namespace**, subject to resource quotas and RBAC policies. Dependencies are packaged in a **container image** pulled from a registry. The key challenge is ephemeral executor storage: shuffle data on local pod disk is lost when a pod is evicted, requiring stage re-runs unless an **External Shuffle Service** is configured with persistent storage. Access to the Spark UI is via `kubectl port-forward`; post-completion analysis requires event log storage in durable object storage. So the story of Spark on Kubernetes is: **submit driver pod → driver creates executor pods → executors register → tasks run inside containers → driver deletes executor pods on completion.** Kubernetes provides the container lifecycle management; Spark provides the task scheduling; the two work together to deliver elastic, cloud-native data processing.
