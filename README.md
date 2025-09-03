INSTALL KUBECTL
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
    chmod +x ./kubectl
    sudo mv ./kubectl /usr/local/bin/kubectl
    kubectl version --client
INSTALL AWS CLI:
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    aws --version
INSTALL HELM
    curl -LO https://get.helm.sh/helm-v3.13.1-linux-amd64.tar.gz
    tar -zxvf helm-v3.13.1-linux-amd64.tar.gz
    sudo mv linux-amd64/helm /usr/local/bin/helm
    helm version

INSTALL EKSCTL
    sudo yum update -y
    sudo yum install -y curl unzip
    curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" -o eksctl.tar.gz
    tar -xzf eksctl.tar.gz
    sudo mv eksctl /usr/local/bin
    eksctl version

AWS CONFIGURE
    With aws access key, secret key, region etc.,

AWS UPDATE KUBECONFIG
    aws eks update-kubeconfig --region <region> --name <cluster-name>

MAKE THE STORAGE CLASS AS DEFAULT
    kubectl patch storageclass gp2 -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}


Step-1: Create an OIDC provider.

cluster_name=my-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)
echo $oidc_id

aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4

eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve


Step 2: Create an IAM role

eksctl create iamserviceaccount \
        --name ebs-csi-controller-sa \
        --namespace kube-system \
        --cluster my-cluster \
        --role-name AmazonEKS_EBS_CSI_DriverRole \
        --role-only \
        --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
        --approve

Step-3: Create add-on.
Note: Replace {my-cluster} with the cluster name,
	      Replace {name-of-addon} with the name of any add-on.
	      Replace {111122223333} with the ID of your account.
	      Replace {role-name} with the role name which you created earlier.
Replace the below values

eksctl create addon --cluster my-cluster --name aws-ebs-csi-driver --version latest --service-account-role-arn arn:aws:iam::730335510642:role/AmazonEKS_EBS_CSI_DriverRole --force

	

step-4: Install Prometheus.

kubectl create namespace prometheus

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm upgrade -i prometheus prometheus-community/prometheus \
    --namespace prometheus \
    --set alertmanager.persistence.storageClass="gp2" \
    --set server.persistentVolume.storageClass="gp2"

Step-4: Set the service type of the Prometheus server from ClusterIP to NodePort with and then Open that node in the security-groups of the cluster worker nodes, because the cluster don't have any control plane like that.
Then you can access it with public-ip of the any worker node with the port.

Step-5:

kubectl create namespace grafana

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update

helm install grafana grafana/grafana \
  --namespace grafana \
  --create-namespace \
  --set adminPassword='admin' \
  --set service.type=NodePort

Step-6:
Default the grafana service will be type of node-port, now just add the inbound traffic to the port of the grafana in the security groups.



