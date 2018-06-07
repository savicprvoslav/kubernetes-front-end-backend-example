##### Build the image
 
 docker build . -t pscode/tomcat:v1

#####Run the image

docker run -d -p 80:80 --name webapp pscode/tomcat:v1

#####Enter the image

docker exec -i -t tomcat sh


#### Kubernetes
Note: For minikube enable ingress
 minikube addons enable ingress


##### Initialize 
eval $(minikube docker-env)

##### Create webapp
kubectl create -f deployment.yaml
kubectl create -f service.yaml
kubectl create -f ingress.yaml


##### Get url of the webapp
minikube service tomcat --url


##### Verify if pod is alive webapp
kubectl get po
kubectl get deployments
kubectl rollout status deployment/tomcat

##### Update Version of the app
kubectl set image deployment webapp webapp=pscode/tomcat:v2

### Access the app in minikube
http://minikubetomcat