# Create a role binding for port-forward
https://c6d7-61-245-161-43.ngrok-free.app/buildStatus/icon?job=github
kubectl create clusterrolebinding default-admin --clusterrole=cluster-admin --serviceaccount=default:default
