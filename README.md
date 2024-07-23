# Hi-sujung k8s-hpa-scaler
In order to schedule Pods in K8s, the Microservice within the Pod must have sufficient resources. From the Kubernetes perspective, you can know the amount of resources needed in advance and deploy Pods on nodes as many resources as are available. Hisujung service set the CPU availability range by considering the number of nodes and target CPU usage rate, and proceeded with the process of optimizing it through testing.

## Implementation
### Cluster Specifications
Our Kubernetes cluster is configured with the following specifications:

- vCPUs per Node: 2
- Memory per vCPU: 3 GB
- Total Nodes: 6
- Boot Disk per Node: 30 GB

### Setting up the Metrics Server
The Kubernetes Metrics Server is an aggregator of resource usage data in the cluster.
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```
Verify that the Metrics Server is working:
```
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

