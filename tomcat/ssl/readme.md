openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN=minikubetomcat"

kubectl create secret tls minikubetomcat-secret --key tls.key --cert tls.crt