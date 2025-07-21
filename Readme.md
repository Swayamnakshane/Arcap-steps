Create EKS Cluster on AWS

IAM user with access keys and secret access keys

AWSCLI should be configured (Setup AWSCLI)

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
Install kubectl(Setup kubectl )

curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
Install eksctl(Setup eksctl)

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
Create EKS Cluster

eksctl create cluster --name=arcap \
                    --region=us-east-2 \
                    --version=1.33 \
                    --without-nodegroup
Associate IAM OIDC Provider

eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster arcap \
  --approve
Create Nodegroup

eksctl create nodegroup --cluster=arcap \
                     --region=us-east-2\
                     --name=arcap \
                     --node-type=t2.large \
                     --nodes=3 \
                     --nodes-min=2 \
                     --nodes-max=3 \
                     --node-volume-size=30 \
                     --ssh-access \
                     --ssh-public-key="public key name"

Install and Configure ArgoCD
Create argocd namespace
kubectl create namespace argocd
Apply argocd manifest
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
Make sure all pods are running in argocd namespace
watch kubectl get pods -n argocd
Install argocd CLI
curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
Provide executable permission
chmod +x /usr/local/bin/argocd
Check argocd services
kubectl get svc -n argocd
Change argocd server's service from ClusterIP to NodePort
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
Confirm service is patched or not
kubectl get svc -n argocd
Check the port where ArgoCD server is running and expose it on security groups of a k8s worker node image
Access it on browser, click on advance and proceed with
<public-ip-worker>:<port>
image image image
Fetch the initial password of argocd server
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
Username: admin
Now, go to User Info and update your argocd password



If you are using a Kind cluster install Metrics Server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
Edit the Metrics Server Deployment
kubectl -n kube-system edit deployment metrics-server
Add the security bypass to deployment under container.args
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
Restart the deployment
kubectl -n kube-system rollout restart deployment metrics-server
Verify if the metrics server is running
kubectl get pods -n kube-system
kubectl top nodes



Nginx ingress controller:
Install the Nginx Ingress Controller using Helm:
kubectl create namespace ingress-nginx
Add the Nginx Ingress Controller Helm repository:
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
Install the Nginx Ingress Controller:
helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer
Check the status of the Nginx Ingress Controller:
kubectl get pods -n ingress-nginx
Get the external IP address of the LoadBalancer service:
kubectl get svc -n ingress-nginx
Install Cert-Manager
Jetpack: Add the Jetstack Helm repository:
helm repo add jetstack https://charts.jetstack.io
helm repo update
Cert-Manager: Install the Cert-Manager Helm chart:
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0 \
  --set installCRDs=true
**Check pods:**Check the status of the Cert-Manager pods:
kubectl get pods -n cert-manager
DNS Setup: Find your DNS name from the LoadBalancer service:
kubectl get svc nginx-ingress-ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
Create a DNS record for your domain pointing to the LoadBalancer IP.
Go to your godaddy dashboard and create a new CNAME record and map the DNS just your got in the terminal.
HTTPS:
1. Update your manifests to enable HTTPS:
04-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: easyshop-config
  namespace: easyshop
data:
  MONGODB_URI: "mongodb://mongodb-service:27017/easyshop"
  NODE_ENV: "production"
  NEXT_PUBLIC_API_URL: "https://easyshop.letsdeployit.com/api"
  NEXTAUTH_URL: "https://easyshop.letsdeployit.com/"
  NEXTAUTH_SECRET: "HmaFjYZ2jbUK7Ef+wZrBiJei4ZNGBAJ5IdiOGAyQegw="
  JWT_SECRET: "e5e425764a34a2117ec2028bd53d6f1388e7b90aeae9fa7735f2469ea3a6cc8c"
2. Update your manifests to enable HTTPS:
10-ingress.yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: easyshop-ingress
  namespace: easyshop
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - easyshop.letsdeployit.com
    secretName: easyshop-tls
  rules:
  - host: easyshop.letsdeployit.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: easyshop-service
            port:
              number: 80
3. Apply your manifests:
kubectl apply -f 00-cluster-issuer.yaml
kubectl apply -f 04-configmap.yaml
kubectl apply -f 10-ingress.yaml
4. Commands to check the status:
kubectl get certificate -n easyshop
kubectl describe certificate easyshop-tls -n easyshop
kubectl logs -n cert-manager -l app=cert-manager
kubectl get challenges -n easyshop
kubectl describe challenges -n easyshop
