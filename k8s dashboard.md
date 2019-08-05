# Deploy K8s Dashboard:

## Deploy Dashboard:

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/alternative/kubernetes-dashboard-arm.yaml

kubectl proxy --address 0.0.0.0 --accept-hosts '.*' 

## CHANGE TO NODEPORT:

kubectl -n kube-system edit service kubernetes-dashboard

You should see yaml representation of the service. Change type: ClusterIP to type: NodePort and save file. 

Next we need to check port on which Dashboard was exposed.

$ kubectl -n kube-system get service kubernetes-dashboard

## DASHBOARD URL:

http://192.168.8.11:31789/#!/overview?namespace=default

## SERVICE ACCOUNT
Create service account, errors will appear on the dashboard if account is not crated:

kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard





