##### Build the image
 
 docker build . -t pscode/webapp:v1


#####Run the image

docker run -d -p 80:80 --name webapp pscode/webapp:v1

#####Enter the image

docker exec -i -t webapp sh


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
minikube service webapp --url


##### Verify if pod is alive webapp
kubectl get po
kubectl get deployments

##### Update Version of the app
kubectl set image deployment webapp webapp=pscode/webapp:v2

### Access the app in minikube
http://minikubewebapp
