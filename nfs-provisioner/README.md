# Set the subject of the RBAC objects to the current namespace where the provisioner is being deployed
$ NS=$(kubectl config get-contexts|grep -e "^\*" |awk '{print $5}')
$ NAMESPACE=${NS:-default}
$ sed -i'' "s/namespace:.*/namespace: $NAMESPACE/g" ./deploy/rbac.yaml ./deploy/deployment.yaml
$ 

kubectl create -f templates/rbac.yaml

Edit deploy/deployment.yaml and replace the two occurences of with your server's hostname.

kubectl create -f templates/deployment.yaml
kubectl create -f templates/class.yaml

to test the provisioner:

kubectl create -f test/test-claim.yaml -f test/test-pod.yaml