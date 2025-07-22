
# 🚀 Create an EKS Cluster on AWS with ArgoCD & HTTPS Support

This guide walks you through setting up a **production-ready EKS cluster** on AWS using `eksctl`, `kubectl`, `ArgoCD`, `Nginx Ingress`, and `Cert-Manager` with HTTPS and DNS support.

---

## 📋 Prerequisites

- ✅ IAM user with **Access Key** and **Secret Access Key**
- ✅ AWS CLI configured
- ✅ Ubuntu or compatible Linux environment

---

## ⚙️ Step 1: Install & Configure AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip
unzip awscliv2.zip
sudo ./aws/install
aws configure
````

---

## 🧰 Step 2: Install kubectl

```bash
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.19.6/2021-01-05/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin
kubectl version --short --client
```

---

## 📦 Step 3: Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## ☁️ Step 4: Create the EKS Cluster

```bash
eksctl create cluster \
  --name=arcap \
  --region=us-east-2 \
  --version=1.33 \
  --without-nodegroup
```

---

## 🔗 Step 5: Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-2 \
  --cluster arcap \
  --approve
```

---

## 🧱 Step 6: Create Node Group

```bash
eksctl create nodegroup \
  --cluster=arcap \
  --region=us-east-2 \
  --name=arcap \
  --node-type=t2.large \
  --nodes=3 \
  --nodes-min=2 \
  --nodes-max=3 \
  --node-volume-size=30 \
  --ssh-access \
  --ssh-public-key="your-public-key-name"
```

---

## 🚦 Step 7: Install & Configure ArgoCD

### 1️⃣ Create Namespace & Apply Manifest

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 2️⃣ Watch Pod Status

```bash
watch kubectl get pods -n argocd
```

### 3️⃣ Install ArgoCD CLI

```bash
curl --silent --location -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.7/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

### 4️⃣ Expose ArgoCD Server

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n argocd
```

Access ArgoCD at:

```text
http://<EC2 Public IP>:<NodePort>
```

Fetch initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Login:

```bash
argocd login <EC2-IP>:<NodePort>
```

---

## 📊 Step 8: (Optional) Install Metrics Server

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl -n kube-system edit deployment metrics-server
```

Add under `container.args`:

```yaml
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP,Hostname,ExternalIP
```

Then restart:

```bash
kubectl -n kube-system rollout restart deployment metrics-server
kubectl top nodes
```

---

## 🌐 Step 9: Install NGINX Ingress Controller

```bash
kubectl create namespace ingress-nginx
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install nginx-ingress ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --set controller.service.type=LoadBalancer

kubectl get svc -n ingress-nginx
```

---

## 🔐 Step 10: Install Cert-Manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0 \
  --set installCRDs=true

kubectl get pods -n cert-manager
```

---

## 🌍 Step 11: Setup DNS

1. Get DNS from LoadBalancer:

```bash
kubectl get svc nginx-ingress-ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

2. Create a **CNAME** record in GoDaddy or your DNS provider and map to the hostname.

---

## 🔒 Step 12: Enable HTTPS with Cert-Manager


### 2. Example Ingress (10-ingress.yaml)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: easyshop-ingress
  namespace: exam
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - asses.arcap.com
    secretName: easyshop-tls
  rules:
  - host: asses.arcap.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-name
            port:
              number: 80
```

### 3. Apply HTTPS Configurations

```bash
kubectl apply -f 00-cluster-issuer.yaml
kubectl apply -f 04-configmap.yaml
kubectl apply -f 10-ingress.yaml
```

### 4. Check Status

```bash
kubectl get certificate -n exam
kubectl describe certificate examportal-tls -n exm
kubectl logs -n cert-manager -l app=cert-manager
kubectl get challenges -n exam
kubectl describe challenges -n exam
```
https://asses.arcapreit.com/
---

## ✅ Final Checklist

* [x] EKS cluster created and running
* [x] Nodegroup attached
* [x] ArgoCD installed & accessible
* [x] Ingress Controller working with LoadBalancer
* [x] HTTPS via Cert-Manager enabled
* [x] DNS properly mapped to Ingress hostname

---

## 📘 References

* [EKS Documentation](https://docs.aws.amazon.com/eks/)
* [ArgoCD Official](https://argo-cd.readthedocs.io/)
* [Cert-Manager Docs](https://cert-manager.io/)
* [Ingress-NGINX](https://kubernetes.github.io/ingress-nginx/)

```

to stop nodegroups

aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name eks-arcap-52cc0c78-43b4-1b69-66f9-50ab3a8bfd46 \
  --min-size 0 \
  --max-size 3 \
  --desired-capacity 0 \
  --region us-east-2

to start nodegroup


aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name eks-arcap-52cc0c78-43b4-1b69-66f9-50ab3a8bfd46 \
  --min-size 2 \
  --max-size 3 \
  --desired-capacity 2 \
  --region us-east-2


to check nodegroups 

aws ec2 describe-instances \
  --filters "Name=tag:eks:nodegroup-name,Values=arcap" \
  --region us-east-2 \
  --query "Reservations[].Instances[].State.Name"

