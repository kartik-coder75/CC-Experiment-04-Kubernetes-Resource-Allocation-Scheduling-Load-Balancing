# Kubernetes Practical Experiment: Resource Allocation, Scheduling & Load Balancing

**Analyze Resource Allocation, Scheduling, and Load Balancing in a Three-Node Cluster**

---

## 1. Aim

To analyze resource allocation, pod scheduling, and load balancing in an existing three-node Kubernetes cluster by deploying applications, configuring CPU and memory requests and limits, observing pod distribution, and testing Service load balancing.

## 2. Requirements

- An existing three-node Kubernetes cluster.
- `kubectl` configured and connected to the cluster.
- Internet connectivity to download container images.
- PowerShell or a terminal.

**Environment used for this run:** a three-node [kind](https://kind.sigs.k8s.io/) cluster (1 control-plane + 2 workers) on Docker/WSL2, Kubernetes `v1.37.0`, containerd `2.3.4`, OS image Debian GNU/Linux 13 (trixie).

## 3. Step-by-Step Procedure

### Step 1: Verify the Kubernetes cluster

```bash
kubectl get nodes -o wide
```

- Displays all nodes available in the cluster.
- Verify that all three nodes are in the `Ready` state.
- Note down the node names.

### Step 2: Analyze available resources

```bash
kubectl top nodes
```

- Displays CPU and memory utilization of each node.
- Compare resource utilization across all three nodes.
- If metrics are unavailable, check whether Metrics Server is installed using the command below.

```bash
kubectl get deployment metrics-server -n kube-system
```

### Step 3: Create a deployment

Create a file named `deployment.yaml` and add the following configuration:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-demo
spec:
  replicas: 6
  selector:
    matchLabels:
      app: resource-demo
  template:
    metadata:
      labels:
        app: resource-demo
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
```

```bash
kubectl apply -f deployment.yaml
```

- Creates a Deployment with six replicas, each running an Nginx container.
- A CPU request of `100m` represents 0.1 CPU core; the memory request is 64 MiB.
- CPU and memory limits define the maximum resources the container may consume.
- The scheduler uses resource requests when deciding where to place pods.

### Step 4: Analyze pod scheduling

```bash
kubectl get pods -o wide
kubectl get events --sort-by=.metadata.creationTimestamp
```

- The `NODE` column identifies the node hosting each pod.
- Observe how the six pods are distributed across the three nodes.
- Scheduling depends on available resources, node conditions, and scheduling constraints.
- Events can provide information about successful scheduling or scheduling failures.

### Step 5: Analyze resource allocation

```bash
kubectl describe nodes
kubectl top pods
```

- Node details show CPU and memory capacity and allocated resources.
- Review the allocated-resource section to understand requested resources on each node.
- The `top` command displays current pod CPU and memory consumption.

### Step 6: Test pod distribution with more replicas

```bash
kubectl scale deployment resource-demo --replicas=12
kubectl get pods -o wide
```

- Increases the number of replicas from six to twelve.
- Kubernetes schedules the additional pods according to available resources and scheduling constraints.
- Compare the new pod distribution with the six-replica deployment.

### Step 7: Create a Kubernetes Service for load balancing

Create a file named `service.yaml` and add the following configuration:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: resource-demo
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

```bash
kubectl apply -f service.yaml
```

- Creates a NodePort Service that selects pods with the label `app: resource-demo`.
- Port `30080` is exposed on cluster nodes, subject to cluster networking and firewall rules.
- The Service forwards connections to eligible backend pod endpoints.
- Kubernetes Service load balancing distributes connections across available endpoints; it does not guarantee equal request counts per pod.

### Step 8: Verify load-balancing endpoints

```bash
kubectl get services
kubectl get endpoints nginx-service
kubectl get endpointslices
```

- Displays the Service and its assigned port.
- Endpoints and EndpointSlices show the pod IP addresses registered as backends.
- Check that the expected healthy application pods are registered.

### Step 9: Test the application

Open a browser and enter the following address, replacing `NODE-IP` with an IP address reachable from your computer:

```
http://NODE-IP:30080
```

- Accesses the Nginx application through the NodePort Service.
- The Service forwards the connection to an available backend pod.
- If permitted by the cluster network, test using the IP addresses of different nodes.
- A successful page response confirms application access through the Service, but does not by itself prove equal traffic distribution.

> In this run the cluster is a kind cluster on Docker, so the application was reached at `http://localhost:30080`.

### Step 10: Observe resource utilization again

```bash
kubectl top nodes
kubectl top pods
```

- Observe CPU and memory utilization after increasing the replicas.
- Compare resource consumption before and after scaling.
- Identify which nodes host the workloads and compare their utilization.
- Remember that pod placement and actual CPU or memory usage are different measurements.

### Step 11: Clean up resources

```bash
kubectl delete service nginx-service
kubectl delete deployment resource-demo
```

- Deletes the Service and Deployment created for this experiment.
- The associated pods are removed.
- The existing three-node cluster remains intact.

---

## 4. Observation Table

| Experiment | Observation to record | Observed result |
|---|---|---|
| Cluster verification | Status of all three nodes | `kind-control-plane` (control-plane, 172.19.0.3), `kind-worker` (172.19.0.4) and `kind-worker2` (172.19.0.2) were all **Ready**, version `v1.37.0`, containerd `2.3.4`. |
| Initial resource utilization | CPU and memory usage of each node | Metrics Server was present in `kube-system` (`1/1` available), so `kubectl top` worked. Baseline node figures were not captured; the post-scaling figures are in the "Final utilization" row. |
| Six replicas | Number of pods scheduled on each node | `resource-demo` pods: **3 on `kind-worker`, 3 on `kind-worker2`, 0 on `kind-control-plane`**. All six were `Running` with 0 restarts. |
| Twelve replicas | Updated pod distribution across nodes | After `kubectl scale --replicas=12`: **6 on `kind-worker`, 6 on `kind-worker2`, 0 on `kind-control-plane`**. The six new pods were split 3 and 3, keeping the spread even. |
| Resource allocation | CPU and memory requests and limits | Per pod: request `100m` CPU / `64Mi` memory, limit `200m` CPU / `128Mi` memory. For 12 pods: total requests = `1200m` CPU / `768Mi`; total limits = `2400m` CPU / `1536Mi`. Each worker carries 6 pods, i.e. `600m` CPU / `384Mi` requested. |
| Load balancing | Service backend endpoints and application response | `nginx-service` is a NodePort Service (ClusterIP `10.96.236.116`, `80:30080/TCP`). Endpoints and the EndpointSlice (`nginx-service-x5c7x`, IPv4, port 80) listed **12 backend pod IPs** (e.g. `10.244.1.10`, `10.244.1.11`, `10.244.1.6`, `10.244.2.9`, `10.244.2.7` ...). `http://localhost:30080` returned the **"Welcome to nginx!"** page. |
| Final utilization | CPU and memory usage after scaling | `kind-control-plane`: 167m CPU (1%), 888Mi memory (23%). `kind-worker`: 42m (0%), 469Mi (12%). `kind-worker2`: 55m (0%), 505Mi (13%). All 12 `resource-demo` pods used `0m` CPU and about 12-16Mi memory each. |

### Additional observations

- The `kubectl get endpoints` command printed a warning that `v1 Endpoints` is deprecated from Kubernetes v1.33+ in favour of `EndpointSlice`.
- A separate `web` Deployment (6 pods, 3 on each worker) already existed in the cluster. Its pods show `1 restart` because the kind node containers were restarted with `docker start` before the experiment.
- Pods were never scheduled on `kind-control-plane`, which carries the default control-plane taint.
- After cleanup, only the pre-existing `web` Deployment (6/6 pods), its ReplicaSet and the default `kubernetes` Service remained.

## 5. Expected Learning Outcomes

- Understand how Kubernetes allocates CPU and memory resources.
- Understand resource requests and limits.
- Observe how the Kubernetes scheduler assigns pods to nodes.
- Analyze how scaling affects pod distribution.
- Understand how Kubernetes Services distribute connections across backend pods.
- Monitor node and pod resource utilization.

## 6. Result

Resource allocation, pod scheduling, and Service-based load balancing were analyzed using the existing three-node Kubernetes cluster.

Cleanup was verified: the `nginx-service` Service and `resource-demo` Deployment were deleted, and the cluster stayed intact.

## 7. Suggested Practical Extension

Increase the CPU request in the Deployment and observe what happens when the cluster does not have enough allocatable CPU to schedule a pod. Use `kubectl get pods -o wide` and `kubectl describe pod POD-NAME` to inspect a Pending pod and its scheduling events. This demonstrates resource-based scheduling constraints.

---

## 8. Conclusion

In this experiment, a three-node Kubernetes cluster (one control-plane and two workers) was used to study resource allocation, scheduling and load balancing.

1. **Resource allocation:** Setting requests (`100m` CPU, `64Mi` memory) and limits (`200m` CPU, `128Mi` memory) gave the scheduler the information it needs to place pods and capped what each container could consume.
2. **Scheduling:** The scheduler spread the workload evenly. Six replicas landed 3 and 3 on the two worker nodes. Scaling to twelve gave a 6 and 6 split. No workload pods were placed on the control-plane node.
3. **Load balancing:** The NodePort Service `nginx-service` registered all 12 pod IPs as endpoints and served the Nginx welcome page on port `30080`. This confirms access through the Service. It does not by itself prove that traffic was shared equally.
4. **Utilization:** Actual usage stayed very low (about 0m CPU and 12-16Mi memory per pod; worker nodes at 12-13% memory). This shows that requested resources (used for scheduling) and real usage (shown by `kubectl top`) are different measurements.
5. **Cleanup:** The Service and Deployment were removed without affecting the cluster.

Overall, the experiment showed how Kubernetes uses resource requests to place pods, how scaling changes pod distribution, and how Services expose and balance connections across backend pods.
