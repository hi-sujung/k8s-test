# Hi-sujung k8s-hpa-scaler
In order to schedule Pods in K8s, the Microservice within the Pod must have sufficient resources. From the Kubernetes perspective, you can know the amount of resources needed in advance and deploy Pods on nodes as many resources as are available. `Hi-sujung` service set the CPU availability range by considering the number of nodes and target CPU usage rate, and proceeded with the process of optimizing it through testing.

<br/>

Before you begin you need to clone the [`hisujung-msa-k8s/load-test`](https://github.com/hi-sujung/hisujung-msa-k8s/tree/main/load-test):

```bash
git clone https://github.com/hi-sujung/hisujung-msa-k8s.git
cd hisujung-msa-k8s/load-test/
```

## `Hi-sujung` Service Resource Settings
- Target CPU Utilization: 50%
- Container CPU Request: 100m 

### Microservice CPU Utilization
CPU Utilization for Main Microservice: 500m <br/>
<br/>
CPU Utilization Calculation:
```bash
CPU Utilization = (500m / 100m) * 100 = 500%
```
Required Number of Pods:
```bash
Required Pods = 500% / 50% = 10
```
### Required Number of Nodes
Total CPU Request per Pod:
```bash
Total CPU Request per Pod = 100m * 3 = 300m
```
Total CPU Request for All Pods:
```bash
Total CPU Request = 300m * (10 * 3) = 9000m
```
## Implementation
### Cluster Specifications
Our Kubernetes cluster is configured with the following specifications:

- vCPUs per Node: 2
- Memory per vCPU: 3 GB
- Total Nodes: 6
- Boot Disk per Node: 30 GB

### Setting up the Metrics Server
The Kubernetes Metrics Server is an aggregator of resource usage data in the cluster.<br/> 
<br/>To install the Metrics Server, use the following command:
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Verify that the Metrics Server is working:
```bash
kubectl get apiservices v1beta1.metrics.k8s.io
```
### Auto Scaling based on CPU and memory usage
[Example](https://github.com/hi-sujung/hisujung-msa-k8s/blob/main/load-test/hisujung-hpa.yaml) HPA configuration:
```yaml
# member
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: member-deployment-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: member-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```
### Installing `wrk` for Load Testing
#### Benchmarking Tool Selection (`wrk`)
The benchmarking tool was selected based on the criteria of being quick and simple to use. Rather than opting for overly complex tools, `wrk` was chosen for its ease of execution from the command line and effectiveness in server load testing.

<br/>

Update your package lists and Install `wrk`:
```bash
sudo apt update
sudo apt install wrk
```
### Applying Configuration Files

```bash
kubectl apply -f hisujung-configmap.yaml && \
kubectl apply -f hisujung-secret.yaml && \
kubectl apply -f hisujung-ingress.yaml && \
kubectl apply -f hisujung-test.yaml && \
kubectl apply -f hisujung-hpa.yaml
```
Verify HPA is working (Ensure the pods stabilize (CPU usage below 5% and replicas count at 1)):
```bash
#Check that all resources are created successfully
kubectl get all

#Monitor the HPA
kubectl get hpa
```
## Load Testing
Once the pods are stable, proceed with load testing using `wrk`. By default, `wrk` performs load testing on a single URL. To perform load testing on multiple IPs simultaneously, a parallel script was executed. The `wait` command ensures that all background processes complete before proceeding.
1. Get the External IPs for `member-service`, `notice-service`, and `portfolio-service`:
```bash
kubectl get svc
```
2. Run `wrk` Load Tests:
```bash
wrk -t 10 -c 100 -d 30s http://<member-IP> & \
wrk -t 10 -c 100 -d 30s http://<notice-IP> & \
wrk -t 10 -c 100 -d 30s http://<portfolio-IP> & \
wait
```
3. Monitor the HPA:
```bash
kubectl get hpa
```
Observe the scaling behavior (scale-out and scale-in) of your deployment.
