kubectl create configmap webapp-default-conf --from-file=default.conf=default.conf

kubectl create -f webapp-default-conf.yaml

kubectl replace -f webapp-default-conf.yaml
