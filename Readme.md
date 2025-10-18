First make sure you have docker desktop, minikube, kubectl and helm installed. 

// Get most recent linux downloads from sites then:
sudo install minikube-linux-amd64 /usr/local/bin/minikube
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

sudo apt install -y conntrack


Minikube + Prometheus + Grafana will run with 2 vCPUs and 5 GB Ram total (if this goes higher things may slow down and pods may be dropped)

minikube config set cpus 2
minikube config set memory 5120
minikube config set driver docker

minikube start

kubectl create namespace base-test

kubectl apply -f nginx-pod.yaml
kubectl apply -f nginx-svc.yaml
// kubectl run nginx --image=nginx --port=80 (if you want to rely implicity on latest image from docker hub)

kubectl get pods -n base-test

kubectl port-forward -n monitoring svc/grafana 3000:80
kubectl port-forward -n monitoring svc/prometheus-server 9090:80


minikube service nginx-service -n base-test

kubectl create namespace monitoring

kubectl apply -f prometheus-config.yaml

kubectl apply -f prometheus-pod.yaml
kubectl get pods -n monitoring

kubectl apply -f prometheus-svc.yaml
minikube service prometheus-service -n monitoring --url
