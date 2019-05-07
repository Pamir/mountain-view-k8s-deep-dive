# Quotas

## Exercise 01: limit the number of pods running in the namespace

You can limit the number of objects user can create in the namespace. For instance, in this excercise you will limit the number of running pods to 2.

1. Create test namespace for this excercise

```
kubectl create namespace quota-01
```

2. Create template quota-pod.yaml

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: pod-demo
spec:
  hard:
    pods: "2"
```

3. Create the resource quota

```
kubectl apply -f quota-pod.yaml --namespace=quota-01
```

4. Get information about created quota

```
kubectl get resourcequota pod-demo --namespace=quota-01 --output=yaml
```

5. Create a deployment with three replicas

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-quota-demo
spec:
  selector:
    matchLabels:
      purpose: quota-demo
  replicas: 3
  template:
    metadata:
      labels:
        purpose: quota-demo
    spec:
      containers:
      - name: pod-quota-demo
        image: nginx
```

```
kubectl apply -f quota-deployment.yaml --namespace=quota-01
```

6. Now check the status of the Deployment

```
kubectl get deployment pod-quota-demo --namespace=quota-01 --output=yaml
```

You will see that there were only 2 replicas out of 3 created

```
spec:
  replicas: 3
status:
  availableReplicas: 2
```

7. Delete the deployment

```
kubectl delete deployment pod-quota-demo --namespace=quota-01
```

## Exercise 02: limit the CPU & memory available for a namespace

1. Create ResourceQuota template in quota-mem-cpu.yaml

```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mem-cpu-demo
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

```
kubectl apply -f quota-mem-cpu.yaml --namespace=quota-01
```
2. Every container must have a memory request, memory limit, cpu request, and cpu limit. Try to create a pod without these specs and see the error.

```file=quota-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
```

```
kubectl apply -f quota-pod.yaml --namespace=quota-01
```

3. Now let's specify the limits for the pod and try to create it again

```file=quota-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
```

```
kubectl apply -f quota-pod.yaml --namespace=quota-01
```

The pod is created.

4. See the resource usage in the namespace

```
kubectl get resourcequota mem-cpu-demo --namespace=quota-01 --output=yaml

status:
  hard:
    limits.cpu: "2"
    limits.memory: 2Gi
    requests.cpu: "1"
    requests.memory: 1Gi
  used:
    limits.cpu: 800m
    limits.memory: 800Mi
    requests.cpu: 400m
    requests.memory: 600Mi
```

5. Try to create the second pod replicas. This will exceed memory quota and throw an error.

```file=quota-pod2.yaml
apiVersion: v1
kind: Pod
metadata:
  name: quota-mem-cpu-demo-2
spec:
  containers:
  - name: quota-mem-cpu-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "800Mi"
        cpu: "800m"
      requests:
        memory: "600Mi"
        cpu: "400m"
```

```
kubectl apply -f quota-pod2.yaml --namespace=quota-01
```

6. Delete all running pods in the namespace

## Exercise 03 (optional): set the default request and limit for a namespace

1. Create LimitRange object

```file=limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      cpu: 1
      memory: "800Mi"
    defaultRequest:
      cpu: 0.5
      memory: "600Mi"
    type: Container
```

```
kubectl apply -f limit-range.yaml --namespace=quota-01
```

2. Create a pod without specifying limits and requests

```file=pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-demo
spec:
  containers:
  - name: default-demo-ctr
    image: nginx
```

```
kubectl apply -f pod.yaml --namespace=quota-01
```

3. Check the limits for created pod

```
kubectl get pod default-demo --output=yaml --namespace=quota-01
```
