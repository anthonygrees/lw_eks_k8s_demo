# Lacework EKS K8s Terraform Demo
  
[![CIS](https://app.soluble.cloud/api/v1/public/badges/bff9c38b-b828-453f-b328-042995862b40.svg)](https://app.soluble.cloud/repos/details/github.com/anthonygrees/lw_eks_k8s_demo)
[![HIPAA](https://app.soluble.cloud/api/v1/public/badges/78b5d2b0-2d85-4335-bd1e-9b8f83093b32.svg)](https://app.soluble.cloud/repos/details/github.com/anthonygrees/lw_eks_k8s_demo)
[![IaC](https://app.soluble.cloud/api/v1/public/badges/e9c36f1d-12ab-4e76-b117-d9dc3e100733.svg)](https://app.soluble.cloud/repos/details/github.com/anthonygrees/lw_eks_k8s_demo)
  
### About
Provision an EKS Cluster using Terraform and add the Lacework Agent on Amazon Elastic Kubernetes Service
  
The instructions in this repo will:
 - Provide `terraform` to create an EKS cluster  
 - Deploy K8s Metrics Server
 - Deploy K8s Dashboard
 - Install Lacework Agent

### Before you start
Before you begin, you will need the following:
 - AWS Account
 - AWS IAM Permissions [Example Here](../main/iam_permissions.md)
 - AWS CLI Installed
 - Kubernetes CLI
 - wget installed
 - Terraform 0.14.11
  
### 1. Deploy EKS Cluster
Clone the repo
```bash
git clone git@github.com:anthonygrees/lw_eks_k8s_demo.git
cd lw_eks_k8s_demo
```
  
Initiate the Terraform
```bash
terraform init -update
```
  
Apply the terraform to create the EKS cluster
```bash
terraform apply
```
  
This process should take approximately 10 minutes. Upon successful application, your terminal prints the outputs  
```bash
Apply complete! Resources: 51 added, 0 changed, 0 destroyed.

Outputs:

cluster_endpoint = "https://80CD543ECDB40DCF1AC9xxxxxxx9710F.sk1.eu-north-1.eks.amazonaws.com"
cluster_id = "rees-eks-kXpc1xxxx"
cluster_name = "rees-eks-kXpcxxxx"
cluster_security_group_id = "sg-03614b30a2xxxxxd7"
config_map_aws_auth = [
  {
    "binary_data" = tomap(null) /* of string */
....
....
```
  
### 2. Configure kubectl
Now that you've provisioned your EKS cluster, you need to configure kubectl.
  
Run the following command to retrieve the access credentials for your cluster and automatically configure kubectl.  
  
```bash
aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
```
  

### 3. Deploy Kubernetes Metrics Server (OPTIONAL)
The Kubernetes Metrics Server, used to gather metrics such as cluster CPU and memory usage over time, is not deployed by default in EKS clusters.  
  
Download and unzip the metrics server by running the following command.  
```bash  
wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
```
  
Deploy the metrics server to the cluster by running the following command.
```bash
kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
```
  
Verify that the metrics server has been deployed. If successful, you should see something like this.
```bash
kubectl get deployment metrics-server -n kube-system
```
  
### 4. Deploy Kubernetes Dashboard and Proxy (OPTIONAL)
The following command will schedule the resources necessary for the dashboard.
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```
  
Your output will look like this  
```bash
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created
```
  
Now, create a proxy server that will allow you to navigate to the dashboard from the browser on your local machine. This will continue running until you stop the process by pressing `CTRL + C`.
  
```bash
kubectl proxy
```
  
Your output will be:  
```bash
Starting to serve on 127.0.0.1:8001
```
  
You can reach the Kubernetes dashboard here - http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
  

### 5. Authenticate the dashboard  (New Terminal) (OPTIONAL)
To use the Kubernetes dashboard, you need to create a ClusterRoleBinding and provide an authorization token.
  
In another terminal (do not close the kubectl proxy process), create the ClusterRoleBinding resource.
```bash
kubectl apply -f https://raw.githubusercontent.com/anthonygrees/lw_eks_k8s_demo/master/kubernetes-dashboard-admin.rbac.yaml
```
  
Then, generate the authorization token.
```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```
  
Select "Token" on the Dashboard UI then copy and paste the entire token you receive into the dashboard authentication screen to sign in. You are now signed in to the dashboard for your Kubernetes cluster.
  
Navigate to the "Cluster" page by clicking on "Cluster" in the left navigation bar. You should see a list of nodes in your cluster.
  
![dashboard](/images/dashboard.png)

### 6. Installing the Lacework Agent
1. In the Lacework Console, download the two Kubernetes YAML files. Navigate to `Settings > Agents`. Either use an existing agent access token or create a new agent token by clicking `+ Create New`. Click Install Options. Download Kubernetes Config and Kubernetes Orchestration.  
   
2. Using the kubectl command line interface, add the Lacework configuration file into the cluster.  
  
```bash
kubectl create -f lacework-cfg-k8s.yaml 
```   
  
3. Instruct the Kubernetes orchestrator to deploy an agent using a `DaemonSet` on all nodes in the cluster, including the master.  
  
To change the CPU and memory limits, see Change Agent Resource Installation Limits on K8s Environments.  
  
```bash
kubectl create -f lacework-k8s.yaml   
``` 
  
4. Repeat the above steps for each Kubernetes cluster. The config.json file is embedded in the lacework-cfg-k8s.yaml file. To customize FIM or add tags in a Kubernetes environment, edit the configuration section of the YAML file and push the revised lacework-cfg-k8s.yaml file to the cluster using the following command.  
  
```bash
kubectl replace -f lacework-cfg-k8s.yaml   
``` 
  
### 7. Check that LW Agent is running on K8s
You can check what is running on  the pod with:  
```bash
kubectl get pods
```
  
The response will be as follows:  
```bash
kubectl get pods                                                                                     

NAME                                  READY   STATUS    RESTARTS   AGE
lacework-agent-9gfwq                  1/1     Running   0          17d
lacework-agent-v96s2                  1/1     Running   0          17d
lacework-agent-vds79                  1/1     Running   0          17d
```
  
If you are running more than one K8s cluster, check the config view with:  
  
```bash
kubectl config view
```
  
Change the config with:  
  
```bash
kubectl config use-context <cluster::name>
```
  
### 8. Check the Polygraph in Lacework
In the Lacework UI, you will now see your K8s cluster in the Polygraph.  
  
![polygraph](/images/polygraph.png)
  
  
### 9. AWS EKS Audit Log Integration
Lacework integrates with AWS to analyze EKS Audit Logs for monitoring EKS cluster security and configuration compliance.   
  
Audit logging must be enabled on the cluster(s) that you want to integrate. You can do this via the AWS CLI using the following command:
```bash
aws eks --region <region> update-cluster-config --name <cluster_name> \
--logging '{"clusterLogging":[{"types":["audit"],"enabled":true}]}'
```
  
  
This scenario creates a new Lacework EKS Audit Log integration with a cross-account IAM role to provide Lacework access. This example targets cluster(s) in a single AWS region.  
  
Create a file called `main.tf` with the following:  
```bash
provider "lacework" {}

module "aws_eks_audit_log" {
  source             = "lacework/eks-audit-log/aws"
  version            = "~> 0.2"
  cloudwatch_regions = ["us-west-1"]
  cluster_names      = ["example_cluster"]
}
```
  
Open a Terminal and change directories to the directory that contains the `main.tf` file and run terraform init to initialize the project and download the required modules.  
   
Run `terraform plan to validate` the configuration and review pending changes.  
  
After you review the pending changes, run `terraform apply` to execute changes.   
  
  
To validate the integration using the CLI, open a Terminal and run the command:   
```bash
lacework integrations list
```
  
The integration will be listed as `AWS_EKS_AUDIT`.  
