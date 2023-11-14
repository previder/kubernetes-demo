# 01 - Deployment

In this manifest you will create and use:
- A [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- A [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) which run the workload
- A [Service](https://kubernetes.io/docs/concepts/services-networking/service/) to expose the deployment
- Manual pod scaling

These pods will be scaled up and down to show the speed and ease of pod scaling.

## Creating a namespace

Create the namespace named "nginx"

### Manifest
```console
cat > 01_namespace.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: nginx
  name: nginx
EOF
```
```console
kubectl apply -f 01_namespace.yaml
```
### CLI
```console
kubectl create ns nginx
```


You can check the creationg of the namespace by executing the following command:
```console
# To see all namespaces in the cluster
kubectl get ns

# To get the output of the created manifest 
kubectl get ns nginx -o yaml
```

## Creating a deployment
Create the deployment named "nginx-deployment"

```console
cat > 01_deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx-deployment
  replicas: 1
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx-deployment
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
EOF
```
```console
# Apply the manifest in the nginx namespace
kubectl -n nginx apply -f 01_deployment.yaml
```

Check the creation of the deployment and pods by executing the following commands.
```console
kubectl -n nginx get deployment
kubectl -n nginx get pods
```

You should see a pod with a name like following with different unique ids:
```console
NAMESPACE     NAME                                                            READY   STATUS    RESTARTS   AGE
nginx         nginx-deployment-8649599b8f-pzccx                               1/1     Running   0          3m
```

## Creating a service
Create a service named "nginx-service" using the following command.
This service will be exposed on a NodePort 32000 and link to the nginx-deployment port 80 based on the selector.

```console
cat > 01_service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: nginx-deployment
  ports:
    - protocol: TCP
      nodePort: 32000
      targetPort: 80
      port: 80
EOF
```
```console
kubectl -n nginx apply -f 01_service.yaml
```

Check the creation of the service by executing the following command:
```console
# All services in the nginx namespace
kubectl -n nginx get svc
# The new service as yaml output
kubectl -n nginx get svc nginx-service -o yaml
```

## Check out the result
You just deployed a basic Nginx server and exposed it on your cluster and VIP on port 32000.
Check the connectivity of your VIP and cluster using the following command.
```console
curl http://<yourvip>:32000
```
Open the url `http://cluster.kubernetes.at-previder.cloud:<yourport>` and replace the port with your dedicated port to check out your Nginx test page!

## Scaling
Sometimes 1 pod will not do the job because of load or high-availability requirements.
Using a [Topology Spread Constraint](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/), you can make sure your pods are spread evenly over your availability zones.
In this module we will manually scale the deployment using the CLI command or via altering the manifest itself.

### CLI
Many actions can be executed using kubectl, scaling is one of them.
Execute the following command to scale up to 5 pods and check the result.
```console
kubectl -n nginx scale deployment/nginx-deployment --replicas=5
kubectl -n nginx get pods
```
Scale down to 0 and your test page should be offline. Try it out!

### Manifest file
Edit the manifest using "vi" and alter the "replicas" property up or down. Save the file and re-apply the file using the next command:
```console
kubectl -n nginx apply -f 01_deployment.yaml
kubectl -n nginx get pods
```

### Kubernetes manifest
Execute the following command to directly alter the Kubernetes manifest via the API
```console
kubectl -n nginx edit deployment nginx-deployment
```

## Cleanup
To clean up the pods, service and deployment at once, just remove the namespace by executing the following command

```console
kubectl delete ns nginx
# or
kubectl delete -f 01_namespace.yaml
```
