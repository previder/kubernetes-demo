# Kubernetes Demo
## Kubernetes Demo manifests
This repo contains different example manifests to use on a Kubernetes cluster to get used to kubectl and helm.  
Starting at 01 with deploying a simple app on the prepared systems which will become reachable on your VIP and dedicated port.  
The further you progress, the more complex the solutions.  
All example modules will expect you to have deleted the service or entire namespace as described at the end of each module.

## Kubernetes resources
Many of the resources used today are default Kubernetes Resources which are documented on the [Kubernetes docs](https://kubernetes.io/docs/concepts/)

## Prerequisites
All module expect your cluster config setup on the ubuntu server. Kubectl and Helm have already been installed and prepared.  
Download your kubeconfig from the Previder Portal by going into the details of your cluster in the list view and click on the Endpoints tab.

![](images/base_endpoints.png)

Open the downloaded file using your favorite notepad and add the following line at the top
```shell
cat > ~/.kube/config << EOF
```
End add this as the last line
```shell
EOF
```

It should look like this, but the contents will differ:
```shell
cat > ~/.kube/config << EOF
---
apiVersion: v1
clusters:
- cluster:
    server: https://<ip address>:6443
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS...
  name: 000005
contexts:
- context:
    cluster: clustername
    user: admin-name
  name: admin-name@clustername
kind: Config
preferences: {}
users:
- name: adminname
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tL...
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSU...
current-context: admin-name@clustername
EOF
```
---
Execute the following command in your homedirectory of the server to create the base folder
```shell
mkdir ~/.kube
```
---
Copy and paste the contents of the entire file including the commands we added and paste it into the server.  
This will create the config file, check the creation of this file by executing the next command, it should output the contents of the file.
```shell
cat ~/.kube/config
```


# Modules
- ## [01 deployment](01_deployment.md)

