# 02 - STaaS

In this manifest you will create and use:
- A [namespace](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
- A Kubernetes CSI (Container Storage Interface)
- [Storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- A [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Pods](https://kubernetes.io/docs/concepts/workloads/pods/) which run the workload with staas storage linked
- [Persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Persistent volume claims](https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/config-and-storage-resources/persistent-volume-claim-v1/)

## Creating a namespace

Create the namespace named "staas"

### Manifest
```shell
cat > 02_namespace.yaml << EOF
apiVersion: v1
kind: Namespace
metadata:
  labels:
    kubernetes.io/metadata.name: staas
  name: staas
EOF
```
```shell
kubectl apply -f 02_namespace.yaml
```
### CLI
```shell
kubectl create ns staas
```

## Install the NFS subdir provisioner
There are many different storage providers for Kubernetes. To use STaaS as dynamically as possible, we will be installing the [NFS subdir external provisioner](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner) using Helm.  
In the test setup a shared STaaS environment and volume has been made available for all members.  
  
To to install the provisioner, execute the following commands:  
```shell
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
```
Execute the following line as one command.  
NFS is setup on IP address 172.16.0.20 and the name of the volume is "volume01"  
```shell
helm install nfs-subdir-external-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
    --namespace staas \
    --set nfs.server=172.16.0.20 \
    --set nfs.path=/volume01/volume01 \
    --set storageClass.name=previder-staas
```
The output of the command should look similar to this:
```text
LAST DEPLOYED: 
NAMESPACE: staas
STATUS: deployed
REVISION: 1
TEST SUITE: None
```
Check the NFS provisioner installation by running the following command:
```shell
kubectl -n staas get pods
kubectl get storageclass
```
The pod should be running and there should be one storageclass calles "previder-staas".
Using this storage class we can create a volume claim for a deployment.

## Volume claim
Execute the following command to create a Persistent Volume Claim to use in a deployment
```shell
cat > 02_pvc.yaml << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mariadb-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: previder-staas
  resources:
    requests:
      storage: 5Gi
EOF
```
```shell
kubectl -n staas apply -f 02_pvc.yaml
```
The claim should now be created. We can check that by executing:
```shell
kubectl -n staas get pvc
kubectl -n staas describe pvc mariadb-pvc
```

Because the storageClassName equals the name of the storage class we created earlier, the NFS provisioner automatically created a Persistent Volume and a Persistent Volume Claim.  
The NFS share has also been mounted on the ubuntu server for debugging and learning purposes. Check out the folder /mnt/staas with the `ls` command and find your PVC folder.  
The default name format is: `${namespace}-${pvcName}-${pvName}`.

## Deployment
Now we have an option for persistent storage, we will deploy a MariaDB server with the storage on STaaS. If the node or cluster would fail, the storage will be safe and reused to prevent data-loss.

Deploy MariaDB with a volume linked to the PVC using the following command
```shell
cat > 02_mariadb_deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mariadb-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: mariadb-deployment
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mariadb-deployment
    spec:
      containers:
      - name: mariadb
        image: mariadb:latest
        securityContext:
          runAsUser: 1000
          allowPrivilegeEscalation: false
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: password
        - name: MARIADB_USER
          value: mariadbuser
        - name: MARIADB_PASSWORD
          value: mariadbpassword
        - name: MARIADB_DATABASE
          value: database
        ports:
        - containerPort: 3306
          name: sql
        volumeMounts:
        - name: mariadb-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mariadb-storage
        persistentVolumeClaim:
          claimName: mariadb-pvc
EOF
```
```shell
kubectl -n staas apply -f 02_mariadb_deployment.yaml
```
After the pod has been created and has the state "Running", the folder in /mnt/staas should contain the DB files and folders.  

## Creating a Mysql service
We will expose mariadb within the cluster only using a ClusterIP service.
This will allow our database deployment to be used by other pods and applications in the cluster.

```shell
cat > 02_mariadb_service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: mariadb-service
spec:
  ports:
  - port: 3306
  selector:
    app.kubernetes.io/name: mariadb-deployment
EOF
```
```shell
kubectl -n staas apply -f 02_mariadb_service.yaml
```

## Wordpress
The database has to be used, right? So we will deploy a Wordpress which will be connecting to the database service and expose that Wordpress instance via a NodePort type service.

```shell
cat > 02_wordpress_deployment.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: wordpress-service
  labels:
    app.kubenetes.io/name: wordpress-service
spec:
  ports:
    - port: 80
      nodePort: 32000
  selector:
    app.kubenetes.io/name: wordpress-deployment
  type: NodePort
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-deployment
  labels:
    app.kubenetes.io/name: wordpress-deployment
spec:
  selector:
    matchLabels:
      app.kubenetes.io/name: wordpress-deployment
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app.kubenetes.io/name: wordpress-deployment
    spec:
      containers:
      - image: wordpress:latest
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: mariadb-service
        - name: WORDPRESS_DB_PASSWORD
          value: mariadbpassword
        - name: WORDPRESS_DB_USER
          value: mariadbuser
        - name: WORDPRESS_DB_NAME
          value: database
        ports:
        - containerPort: 80
          name: web
EOF
```
```shell
kubectl -n staas apply -f  02_wordpress_deployment.yaml
```

## Check out the result
You just deployed an empty Wordpress instance and exposed it on your cluster and VIP on port 32000.
Check the connectivity of your VIP and cluster using the following command.
```console
# Replace the IP with your dedicated cluster VIP
curl http://172.16.0.50:32000
```

Open the url (http://cluster.kubernetes.at-previder.cloud:33000) and replace the port with your dedicated port to check out your new Wordpress instance.

**Important:** The instance is **NOT** secures with TLS/SSL! Do not send sensitive data you don't want roaming on the internet.  
TLS offloading will be handled in module [03 Ingress and SSL](03_ssl.md)

## Cleanup
To clean up the pods, service and deployment at once, just remove the namespace by executing the following command

```console
kubectl delete ns staas
# or
kubectl delete -f 02_namespace.yaml
```
The cleanup will take a few seconds as it removes all resources created including the NFS provisioner.
