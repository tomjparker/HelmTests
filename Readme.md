First make sure you have docker desktop, minikube, kubectl and helm installed. 

// Get most recent linux downloads from sites then:
sudo install minikube-linux-amd64 /usr/local/bin/minikube
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

sudo apt install -y conntrack


Minikube + Prometheus + Grafana will run with 2 vCPUs and 5 GB Ram total (if this goes higher things may slow down and pods may be dropped)

// Set personal computer limitations for virtual machines to use
minikube config set cpus 2
minikube config set memory 5120
minikube config set driver docker

minikube start
kubectl create namespace base-test

kubectl apply -f nginx-pod.yaml
kubectl apply -f nginx-svc.yaml
// kubectl run nginx --image=nginx --port=80 (if you want to rely implicity on latest image from docker hub)

kubectl get pods -n base-test

// kubectl port-forward -n monitoring svc/grafana 3000:80
k// ubectl port-forward -n monitoring svc/prometheus-server 9090:80


minikube service nginx-service -n base-test

kubectl create namespace monitoring

kubectl apply -f prometheus-config.yaml

kubectl apply -f prometheus-pod.yaml
kubectl get pods -n monitoring

kubectl apply -f prometheus-svc.yaml
minikube service prometheus-service -n monitoring --url

kubectl apply -f grafana-pod.yaml

minikube service grafana-service -n monitoring --url   // go to the url in browser

// Will prompt you for new user and pass, just set
admin1
admin1

add a connection -> data sources -> prometheus -> Url (http://prometheus-service.monitoring.svc.cluster.local:9090)   // These tunnels are temporary, and if you interrupt the tunnel stops

minikube service grafana-service -n monitoring --url
minikube service prometheus-service -n monitoring --url

1860 for node exporter template

node exporter runs as a daemonset, one per k8s node


if you repush a change to a pod, it likely uses the old version of a config until it restarts so you may have to cleanup sometimes
kubectl delete pod -n monitoring -l app=prometheus   // It should auto spin back up after from the replicasets
kubectl logs prometheus -n monitoring | grep "Loaded configuration"

typical cluster setups:
On AWS (EKS cluster):

1. Prometheus & Grafana deployed via Helm (kube-prometheus-stack)
2. Persistent volumes:
    - Prometheus → gp3 EBS volume
    - Grafana → gp3 EBS volume

3. Optional: Backup dashboards/configs to S3
4. External access: via ALB Ingress Controller or AWS Load Balancer Service

On Azure (AKS cluster):
1. Prometheus & Grafana via Helm or Flux
2. Persistent volumes:
    - Prometheus → Azure Managed Disk
    - Grafana → Azure Disk
3. External access: via Azure Application Gateway Ingress Controller
4. Optional: integrate Grafana with Azure AD SSO


# NB: Typically you wouldnt create the helm charts yourself, you would take from a Helm repository, this work is to merely connect the dots and understand fully the repercusions. 

1. deployment.yaml: Defines how Pods are managed and replaced by the `Deployment` controller (e.g., replicas, rolling updates) | Optional for dev (I used static Pods here), Needed for prod     
2. service.yaml: Exposes your Pods internally (ClusterIP) or externally (NodePort/LoadBalancer) | Needed for dev and prod
3. configmap.yaml: Stores config files for Prometheus or app config | Needed for dev and prod
4. serviceaccount.yaml:  Used for RBAC permissions so Pods can access the API or other resources securely | Optional for dev, needed in prod 
5. hpa.yaml: Horizontal Pod Autoscaler — scales replicas based on CPU/memory | Unnecessary for dev, needed for prod
6. ingress.yaml/httproute.yaml: Exposes HTTP traffic through a controller (Nginx, Application Gateway, ALB, etc.) | Optional for dev, needed in prod
7. pvc.yaml/storage.yaml: Persistent storage for Grafana/Prometheus | Optional for dev, needed in prod

# --- Taints --- #

kubectl taint nodes minikube key=infra:NoSchedule   # Don’t schedule any pod on this node unless it tolerates the taint key=infra:NoSchedule

# Sample toleration section (goes under spec in yaml):
tolerations:
    - key: "infra"
      operator: "Equal"
      value: "NoSchedule"
      effect: "NoSchedule"

kubectl taint nodes minikube key=infra:NoSchedule-  # To untaint

# --- Selectors --- # 

kubectl label nodes minikube node-type=frontend     # Basically implies - please only tun on nodes where they have this label

# On a pod yaml itself
spec: 
    nodeSelector:
        node-type: frontend


# --- Affinity --- #

# Sample affinity section
spec:
    affinity:
        nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
                - key: node-type
                    operator: In
                    values:
                    - backend

# Only schedule on nodes labeled node-type=backend. If none exist, pod stays pending.

spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - key: node-type
                operator: In
                values:
                  - backend

# Prefer backend nodes, but schedule elsewhere if needed.

spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - my-app
          topologyKey: "kubernetes.io/hostname"

# Don’t schedule two my-app pods on the same node - used in Deployments with multiple replicas for high availability.