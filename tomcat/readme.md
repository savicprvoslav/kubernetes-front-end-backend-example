##### Build the image
 
 docker build . -t pscode/tomcat:v1

 docker tag pscode/webapp:v1 localhost:5000/pscode/webapp:v1

#####Run the image

docker run -d -p 80:80 --name webapp pscode/tomcat:v1

#####Enter the image

docker exec -i -t tomcat sh


#### Kubernetes
Note: For minikube enable ingress
 minikube addons enable ingress


##### Initialize 
eval $(minikube docker-env)
docker run -d -p 5000:5000 --restart=always --name registry registry:2


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


        readinessProbe:
          httpGet:
            path: /app
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 1
          periodSeconds: 10
        livenessProbe:
            httpGet:
              path: /app
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 15
            timeoutSeconds: 20