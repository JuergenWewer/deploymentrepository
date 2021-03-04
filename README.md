# License

Copyright 2020 OPTIMAL SYSTEMS GmbH

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.

# yuuvis api helm charts

## Prerequisites

we use version: v3.5.0
Please use helm version [v3.2.4](https://github.com/helm/helm/releases/tag/v3.2.4), newer versions may not be compatible with some of the helm charts.


## install persistent volumes in kubernetess

check disks:

on master:
lsblk
-> sdb1

list all mounted filesystems:
sudo mount -t ext4

to mount a disk on /dev/sdb1 on the directory /mnt/optimal
sudo mkdir -p /mnt/optimal
sudo mount -t ext4 /dev/sdb1 /mnt/optimal

ansible-playbook -i hosts ./playbooks/nfs/nfs-dependencies.yml -v --extra-vars "ansible_sudo_pass=OSVHyuuvis2021!"

ansible-playbook -i hosts ./playbooks/pvs.yml -v --extra-vars "ansible_sudo_pass=OSVHyuuvis2021!"

## yuuvis installation

First please add your credentials for the docker.yuuvis.org registry in the values yaml files of the helm charts.  For any questions about credentials please contact support@yuuvis.com.

Replace all **changeme** default passwords in the values.yaml of the charts you plan to use.   !! minimum length 8 Character (minio)

### Add required Helm repositorys:

```shell
helm repo add minio https://helm.min.io/
helm repo add bitnami https://charts.bitnami.com/bitnami
```
### Install the infrastructure Helm chart

#### Update infrastructure dependencies

old:
```shell
cd infrastructure
helm dep up
helm repo add stable https://kubernetes-charts.storage.googleapis.com/
cd ..
```

new:
https://kubernetes-charts.storage.googleapis.com/ is no longer available
instead:
helm repo rm stable

```shell
cd infrastructure
helm dep up
helm repo add stable https://charts.helm.sh/stable
cd ..
```

#### Edit the infrastructure values.yaml

* Edit the docker registry credentials. 
* Optionally change passwords
* Optionally change the used storage classes

#### Install infrastructure services

export KUBECONFIG=/Users/wewer/.kube/config/master/etc/kubernetes/admin.conf

```shell
kubectl create namespace infrastructure
helm install infrastructure ./infrastructure --namespace infrastructure
```

wait till jobs are done

```shell
kubectl get jobs -n infrastructure
```

There are 2 jobs that prepare the git server and the keycloak environment that need to be completed.

```shell
NAME                              COMPLETIONS   DURATION   AGE
gogsrepo-init                     1/1           83s        8m4s
keycloak-create-selfsigned-cert   1/1           8m4s       8m4s
```

#### Edit the yuuvis values.yaml

* Edit the docker registry credentials.

check port of keycloak:
look for service:
keycloak-https ....NodePort....443/30111/TCP

check keycloak endpoint: https://<master-ip>:30111

### Install the yuuvis Helm chart

```shell
kubectl create namespace yuuvis
helm install yuuvis ./yuuvis --namespace yuuvis
```

wait till all pods are ready 

```shell
kubectl get po -n yuuvis
```

#### Edit the client values.yaml

* Edit the docker registry credentials.

### Install the yuuvis client Helm chart:

```shell
helm install client ./client --namespace yuuvis
```

wait till all pods are ready 

```shell
kubectl get po -n yuuvis
```

#### Post-install tasks for the client

The client helm chart will change the *systemHookConfiguration.json*.  
Services that use this configuration will only read it once at startup.  
For the changes to be noticeable the corresponding services must be restart.  
The changes in the *systemHookConfiguration.json* affect the api gateway.  
To restart the api gateway:  

```shell
kubectl rollout restart deployment api -n yuuvis
```

A role *YUUVIS_CREATE_OBJECT* must be created and assigned to users who should be able to create objects in the client.  

#### Edit the bpm values.yaml

* Edit the docker registry credentials.


### Install the yuuvis bpm Helm chart

install bpm services with:
```shell
kubectl get po -n yuuvis
helm install bpm ./bpm --namespace yuuvis
```

## Version upgrades

The upgrade of the infrastructure chart is not supported at the moment.

For upgrading the yuuvis components get the new Helm charts version, edit the values.yaml of each chart with your modifications and the upgrade the Helm deployments:

Check version of deployed helm chart

```shell
helm list -n yuuvis 
```


```shell
helm upgrade yuuvis ./yuuvis --namespace yuuvis 
helm upgrade client ./client --namespace yuuvis 
helm upgrade bpm ./bpm --namespace yuuvis 
```
Check version of upgraded helm chart

```shell
helm list -n yuuvis 
```

## Uninstall

```shell
 helm uninstall infrastructure  --namespace infrastructure
 helm uninstall yuuvis  --namespace yuuvis
 helm uninstall client  --namespace yuuvis
 helm uninstall bpm  --namespace yuuvis
```

```shell
kubectl delete statefulset elasticsearch -n infrastructure
kubectl delete statefulset rabbitmq -n infrastructure
kubectl delete jobs keycloakaddrole-yuuvis  -n infrastructure
kubectl delete jobs keycloak-create-selfsigned-cert -n infrastructure
kubectl delete job gogsrepo-init -n infrastructure
kubectl delete pvc gogs -n infrastructure
kubectl delete pv name(replace with pv from gogs --check value with kubectl get pv -n infrastructure) -n infrastructure

kubectl delete pv -l app=pvcs 
```
While deleting persistence volume and persistence claim from gogs, please delete pod, than they will be unbind.


Install nginx-ingress

[comment]: <> (kubectl create namespace nginx-ingress)

[comment]: <> (helm repo add nginx-stable https://helm.nginx.com/stable)

[comment]: <> (helm repo update)

[comment]: <> (helm install nginx-ingress ingress-nginx --namespace nginx-ingress)

helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx

use kubernetes helm repos in ansible
ansible-galaxy collection install community.kubernetes

in playbooks use:
community.kubernetes.helm_repository

cd playbooks
ansible-playbook -i macpro start.yml -v
ansible-playbook -i macpro delete.yml -v
