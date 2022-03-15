#deployment repository

deploy the agent to the cluster:
export KUBECONFIG=/Users/wewer/.kube/master/etc/kubernetes/admin.conf
kubectl create -f deploymentagent/deployment.yaml
kubectl delete -f deploymentagent/deployment.yaml