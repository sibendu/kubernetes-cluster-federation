Provision Clusters
===================
gcloud beta container clusters create asia-east1-b --cluster-version 1.16.13-gke.401 --zone asia-east1-b  --scopes "cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,https://www.googleapis.com/auth/ndev.clouddns.readwrite" 


gcloud beta container clusters create europe-west1-b --cluster-version 1.16.13-gke.401 --zone europe-west1-b  --scopes "cloud-platform,storage-ro,logging-write,monitoring-write,service-control,service-management,https://www.googleapis.com/auth/ndev.clouddns.readwrite" 


for cluster in asia-east1-b europe-west1-b; do
  gcloud container clusters get-credentials ${cluster} --zone ${cluster}
done

GCP_PROJECT=$(gcloud config list --format='value(core.project)')

for cluster in asia-east1-b europe-west1-b; do
  kubectl config set-context ${cluster} --cluster=gke_${GCP_PROJECT}_${cluster}_${cluster} --user=gke_${GCP_PROJECT}_${cluster}_${cluster}
done


HOST_CLUSTER=asia-east1-b


kubectl config set-context host-cluster --cluster=gke_${GCP_PROJECT}_${HOST_CLUSTER}_${HOST_CLUSTER} --user=gke_${GCP_PROJECT}_${HOST_CLUSTER}_${HOST_CLUSTER} --namespace=federation


Create a Google DNS Managed Zone
==================================
gcloud dns managed-zones create federation --description "Kubernetes federation testing" --dns-name federation.sddomain


Provision Federated Control Plane
===================================
kubectl config use-context host-cluster

Federation Namespace:: kubectl create -f ns/federation.yaml

Federated API Server Service:: kubectl create -f services/federation-apiserver.yaml

kubectl get services 
>> federation-apiserver   LoadBalancer   10.15.252.95   35.234.48.135   443:30917/TCP   62s

Create Federation API Servr secret: 
-----------
FEDERATION_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')

cat > known-tokens.csv <<EOF
${FEDERATION_TOKEN},admin,admin
EOF

Create the federation-apiserver-secrets
------------
kubectl create secret generic federation-apiserver-secrets --from-file=known-tokens.csv

kubectl describe secrets federation-apiserver-secrets


Federation API Server Deployment
----------------
kubectl create -f pvc/federation-apiserver-etcd.yaml
kubectl get pvc
kubectl get pv

kubectl create configmap federated-apiserver --from-literal=advertise-address=$(kubectl get services federation-apiserver -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

kubectl get configmap federated-apiserver -o jsonpath='{.data.advertise-address}'


kubectl create -f deployments/federation-apiserver.yaml  (need change in yaml to adjust for k8s 1.16)


Provision Federated Controller Manager
====================================

kubectl config set-cluster federation-cluster --server=https://$(kubectl get services federation-apiserver -o jsonpath='{.status.loadBalancer.ingress[0].ip}') --insecure-skip-tls-verify=true

FEDERATION_TOKEN=$(cut -d"," -f1 known-tokens.csv)

kubectl config set-credentials federation-cluster --token=${FEDERATION_TOKEN}

kubectl config set-context federation-cluster --cluster=federation-cluster --user=federation-cluster

kubectl config use-context federation-cluster

mkdir -p kubeconfigs/federation-apiserver

kubectl config view --flatten --minify > kubeconfigs/federation-apiserver/kubeconfig


Create the Federated API Server Secret
---------------
kubectl config use-context host-cluster

kubectl create secret generic federation-apiserver-kubeconfig --from-file=kubeconfigs/federation-apiserver/kubeconfig

kubectl describe secrets federation-apiserver-kubeconfig


Deploy the Federated Controller Manager
----------------
DNS_ZONE_NAME=$(gcloud dns managed-zones describe federation --format='value(dnsName)')
DNS_ZONE_ID=$(gcloud dns managed-zones describe federation --format='value(id)')

kubectl create configmap federation-controller-manager --from-literal=zone-id=${DNS_ZONE_ID} --from-literal=zone-name=${DNS_ZONE_NAME}

kubectl get configmap federation-controller-manager -o yaml

kubectl create -f deployments/federation-controller-manager.yaml


kubectl get pods  -> federation-controller-manager pod running


Configure kube-dns with federated DNS support
---------------
kubectl config use-context federation-cluster

cat > configmaps/kube-dns.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-dns
  namespace: kube-system
data:
  federations: federation=${DNS_ZONE_NAME}
EOF

kubectl create -f configmaps/kube-dns.yaml 


Add Clusters to Federation
==========================
kubectl config use-context host-cluster


CLUSTERS="asia-east1-b europe-west1-b"














