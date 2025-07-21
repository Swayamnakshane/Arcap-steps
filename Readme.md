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
