# kubernetes-front-end-backend-example
Demonstration on how to setup ingress, two services ( frontend and backend) with reverse proxy.

## Reverse proxy 
 User will hit the front end on url http://minikubewebapp to get the static pages for example index.html. Now lets assume that in index.html we have a js code that requires the backend to load some data, for example /hello API, what are our options?
 
 One of the options is to have a domain api.minikubewebapp where script can access with proper CORS setup, another option is to hide the domain and allow javascript to call api on url http://minikubewebapp/service/hello. This call will be reverse proxied to the real backend located on http://minikubetomcat. Second approach is a way to go because it allows you to configure what backend is used for front end and front end is not aware of this.
 
## Side car containers
You can read more about sidecar containders but in short is to have one pod with multiple containers that run inside and where each one has specific role. For example one has a role of nginx server, second provides the index.html and other webapp related contents, third can do a log rotate ...

In our case both tomcat and webapp pods will cointain two containers one to provide the war and index.html and other will provide nginx and tomcat. One part is what is run and other is who runs them.

This has multiple benefits, let assume that we want to switch from nginx to apache httpd, or tu upgrade tomcat version how many lines we have to change? Well only one, we just change the deployoment and specify other image name. 

## Tomcat

### Create Docker image

First we need to create a war, for this I have already created a example war downloaded from [tomcat website](https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/) 

Next we need to create a docker image that will contain only this war file. This is super easy using Dockerfile with only three lines of code:
```
FROM busybox:latest
ADD app.war app.war
CMD "tail" "-f" "/dev/null"
```
and running docker build
```
docker build . -t pscode/tomcat:v1
```

### Create kubernets deployment
Kubernetes deployment.yaml located in folder kubernetes is a longest and most complex file, but when split into sections feature by feature it is super easy to understand and follow.

First 20 lines are:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat
  labels:
    app: tomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tomcat
  strategy:
    type: RollingUpdate
    rollingUpdate: 
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
         app: tomcat
```
This lines define the deployment name, labels, replicas, strategy and template and are probably what you will find in every deployment yaml out there.

Next we have two containers one is the war and other is the tomcat, war container is where the war file is located.
```
      - image: pscode/tomcat:v1
        name: war
        lifecycle:
          postStart:
            exec:
              command:
              - "cp"
              - "/app.war"
              - "/app"
        volumeMounts:
        - mountPath: /app
          name: app-volume
```
First we have specified the image tag, and a lifecycle post start action, what does this action do is it copies the app.war into the app folder that is a volume mounted in both containers. 

Next we have tomcat container ( basic part)
```
  - image: tomcat:7
        name: tomcat
        env:
         - name: JAVA_OPTS
           value: -Xmx300m -Xms300m -XX:+PrintGCDetails  -XX:+PrintGCTimeStamps
        volumeMounts:
        - mountPath: /usr/local/tomcat/webapps/
          name: app-volume
        ports:
        - containerPort: 8080
```
We have specified the container port to 8080 and specified the volume app-volume that is mounted in tomcat webapps folder. So when war container starts it copies the file to /app that is then visible by the tomcat in /usr/local/tomcat/webapps/ folder. 

Next we have resources specified for tomcat :

```
        resources:
          limits:
            cpu: 200m
            memory: 400Mi
          requests:
             cpu: 200m
             memory: 400Mi
```

Remember that limits must be equal or higher than requests.

As you know when tomcat starts depending of the size of your app it takes some time while tomcat starts to respond to requests, what is important is that your tomcat does not get any requests before he is actually ready. This is accomplished by readinessProbe configuration.

```
        readinessProbe:
          httpGet:
            path: /app
            port: 8080
          initialDelaySeconds: 10
          timeoutSeconds: 1
          periodSeconds: 10
```

After tomcat is started it can process requests, problem is what happens when for example tomcat stops responding for what ever reason. This detection is done by livenessProbe. When probe starts to fail pod is restarted by kubernetes.

```
         livenessProbe:
            httpGet:
              path: /app
              port: 8080
              scheme: HTTP
            initialDelaySeconds: 20
            timeoutSeconds: 20

```

Last part of deployment is just a volume configuration
```
  volumes:
      - name: app-volume
        emptyDir: {}
```

### Create kubernetes service
Service is super easy as all it has is a selector and a service port
```
apiVersion: v1
kind: Service
metadata:
  name: tomcat
spec:
  type: NodePort
  ports:
  - name: http
    port: 8082
    targetPort: 8080
  selector:
    app: tomcat
```

### Create kubernetes ingress
Ingress is a feature that allows proxing request from outside of the cluster to the services and pods. 

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  name: tomcat
spec:
  tls:
    - hosts:
      - minikubetomcat
      secretName: minikubetomcat-secret
  rules:
  - host: minikubetomcat
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat
          servicePort: 8082
```

What it does every request to https://minikubetomcat it proxies to service tomcat. You will notice that it has tls setup. In order to create a kubernets secret just use files from tomcat/ssl that hold the instruction for creating key, crt and a secret.
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=minikubetomcat"

kubectl create secret tls minikubetomcat-secret --key tls.key --cert tls.crt
```
This is it for the tomcat, once you create deployment, service, secret and ingress you should be able to hit https://minikubetomcat. 

Few things to remember you need to change /etc/hosts and add something like this:
```
 192.168.99.100 minikubewebapp
 192.168.99.100 minikubetomcat
```

## Webapp
For the webapp part I will not go into details this much as everything is mostly the same but I will focus on one important part nginx configuration. 

Webapp is configured to be accesable on address https://minikubewebapp and tomcat is suppossed to be accessable on https://minikubewebapp/service .

Would not it be awesome to decouple nginx configuration and nginx it self ? This is done using kubernets feature called configmaps. Nginx configmap is created to hold the nginx config:
```
server {
    listen       80;
    server_name  localhost;

    location /service/ {
            proxy_pass http://tomcat:8082/app;
    }

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

```

Then we will mount this config map in the /etc/nginx/conf.d folder. 

First we must create configmap for nginx using webapp-default-conf.yaml configuration. 

After we have created the configmap we must mention it in deployment: 
```
       volumeMounts:
        - mountPath: /etc/nginx/conf.d
          name: config-volume
      volumes:
      - name: config-volume
        configMap:
          name: webapp-default-conf
          items:
          - key: default.conf
            path: default.conf
```

This is it, hopefully this will help you get started with kubernetes, front end backend with reverse proxy. 



