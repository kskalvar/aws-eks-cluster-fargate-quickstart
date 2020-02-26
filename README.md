AWS Elastic Kubernetes Service (EKS) Fargate QuickStart  
=======================================================
Abstract:
```
Although AWS EKS has been available for quite sometime and AWS EKS Fargate has finally been released, 
there are still some manual configuration steps required to get the Fargate worker nodes to work with 
the cluster.  In this quickstart we'll step you throught the entire process of deploying a simple web 
application and give a simple demo of how fargate can scale nodes dynamically.
```
This solution shows how to create an AWS EKS Cluster with Fargate support and deploy a simple web application with an external Application Load Balancer.  
```
Note: This how-to assumes you are creating the eks cluster in us-east-1, you have access to your AWS Root
Account, and you can login to an EC2 Instance remotely.
```
Steps:  
* [Create an EC2 Instance](#create-an-ec2-instance)
* [Create EKS Cluster IAM Security and ALB Ingress Controller](#create-eks-cluster-iam-security-and-alb-ingress-controller)  
* [Deploy Simple WebApp to Your Cluster](#deploy-simple-webapp-to-your-cluster)
* [Configure the Kubernetes Dashboard (Optional)](#configure-the-kubernetes-dashboard-optional)  
* [Remove AWS EKS Cluster](#remove-aws-eks-cluster)  
* [References](#references)   

To make this first microservice easy to deploy we'll use a docker image located in DockerHub at kskalvar/web.  This image is nothing more than a simple webapp that returns the current ip address of the container it's running in.  We'll create an external AWS Application Load Balancer and you should be able to see a unique ip address as it is load balanced across containers.

The project also includes the Dockerfile for those interested in the configuration of the actual application or to build your own and deploy using ECR.

## Create an EC2 Instance 
We'll using an EC2 instance to install kubectl, eksctl, so we can create the EKS Cluster, and worker nodes.  This is a step by step process.

### AWS EC2 Dashboard
Using AWS Managment Console goto the AWS EC2 Dashboard  
Click on "Launch Instance"  

Choose AMI  
```
Amazon Linux 2 AMI (HVM), SSD Volume Type  
```  
Click on "Select"

Choose Instance Type
```
t2.micro
```
Click on "Next"

Configure Instance  
Click on "Advanced Details
```
User data
Select "As file"
Click on "Choose File" and Select "cloud-init/cloud-init" from github project directory 
```  
Next

Add Storage  
Next

Click on "Add Tag"
```
Key: Name
Value: eks_cloud_shell
```
Next  

Configure Security Group  
Select "Select an existing security group"  
Select "default"

Review and Launch  
Click on "Launch"

### Check to insure cloud-init has completed
Log into your EC2 instance above and check the contents of "/tmp/install-eks-support" it should say "installation complete".

### Configure AWS CLI
Use the AWS CLI to set Access Key, Secret Key, and Region Name
```
aws configure
AWS Access Key ID []: <Your Access Key ID>
AWS Secret Access Key []: <Your Secret Access Key>
Default region name []: us-east-1
```
Test aws cli
```
aws s3 ls
```
## Create EKS Cluster, IAM Security and ALB Ingress Controller
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.  
  
We will be using eksctl to create the cluster and the iam security provider.  The cloud-init script  
also installed this project from github and copied installation scripts we will use into /home/ec2-user.  
### Create Cluster
Using eksctl create the cluster
```
eksctl create cluster --name eks-cluster --zones=us-east-1c,us-east-1b,us-east-1a --version 1.14 --fargate
```
Wait till cluster creation has completed before proceeding.  

### Test Cluster
Using kubectl test the cluster status
```
kubectl get svc 
```
### Test Cluster Nodes
Use kubectl test status of cluster nodes
```
kubectl get nodes
```
### Configure the Ingress Controller
Configure the ingress controller using the provide script
```
NOTE: There is a script called "configure-alb-ingress-controller" in /home/ec2-user to configure the ingress
controller for kubernetes and as well as associated AWS IAM Policies.  Using this script will automate the steps
required so you won't have to do them manually. 

./configure-alb-ingress-controller
```
Wait till deployment rollout is complete.
## Deploy Simple WebApp to Your Cluster
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.  

### Create a AWS EKS Fargate Profile
Will use eksctl to create a unique Fargate Profile for this webapp.  This also assoicates a namespace with the profile
which we will use when deploying the webapp below.
```
eksctl create fargateprofile --name web --namespace web-namespace --cluster eks-cluster
```
Using AWS Managment Console goto the AWS EKS Dashboard and insure your cluster Fargate Profile is "Active"  
### Configure the web ingress
You will use the following script to configure the web ingress
```
NOTE: There is a script called "configure-web-ingress" in /home/ec2-user to configure the web-ingress.yaml.  
It requires the AWS VPC Public Subnets used by the cluster and can only be known after the cluster is created.  
Using this script will pre-populate it so you don't need to edit manually.  

./configure-web-ingress
```
### Create the Web Service
Use kubectl to create the web service  
```
kubectl apply -f web-namespace.yaml
kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f web-ingress.yaml
```
### Show Pods Running
Use kubectl to display pods
```
kubectl get pods -n web-namespace --output wide
```
Wait till you see all pods appear in "STATUS Running"
### Get AWS External Application Load Balancer Address
Capture ADDRESS for use below
```
kubectl get ingress/web-ingress -n web-namespace
```
### Test from browser
Using your client-side browser enter the following URL. NOTE: It takes a few minute for ALB to be provisioned!  You  
can check using the AWS Management Console by goint to the EC2 Dashboard/Load Balancing
```
http://<ADDRESS>
```
### Test Scaling
Use kubectl to test scaling of the application.  You should see a new node for each container you create
```
kubectl scale deployment web-container-ip --replicas=4 -n web-namespace
kubectl get pods,nodes -n web-namespace --output wide
```

### Delete Deployment, Service, Namespace
Use kubectl and eksctl to delete application.
```
kubectl delete -f web-ingress.yaml
kubectl delete -f web-service.yaml
kubectl delete -f web-deployment.yaml
kubectl delete -f web-namespace.yaml
```
Wait till the delete has completed before proceeding
### Delete Fargate Profile
Use eksctl to delete profile.
```
eksctl delete fargateprofile --name web --cluster eks-cluster
```
Wait till the fargate profile has completed before proceeding

## Configure the Kubernetes Dashboard (Optional)
You will need to configure the dashboard from the AWS EC2 Instance you created as well.  Use ssh to create a tunnel on port 8001 from your local machine.  This is a step by step process.

### configure-kube-dashboard
Configure Kubernetes Dashboard 
```
NOTE:  There is a script in /home/ec2-user called "configure-kube-dashboard".  
       You may run this script to automate the installation of the dashboard components into the cluster,
       configure the service role, and start the kubectl proxy.
       
./configure-kube-dashboard
```
### Connect to EC2 Instance redirecting port 8001 Locally
Using ssh from your local machine, open a tunnel to your AWS EC2 Instance
```
ssh -i <AWS EC2 Private Key> ec2-user@<AWS EC2 Instance IP Address> -L 8001:localhost:8001
```
### Test from Local Browser
Using your local client-side browser enter the following URL. The configure-kube-dashboard script
also generated a "Security Token" required to login to the dashboard.
```
http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```
## Remove AWS EKS Cluster
Use eksctl to delete all resources used by the AWS EKS Cluster
```
Note: Before proceeding be sure you delete the ingress, service, deployment, and namespace as instructed above.
```
### Delete EKS Cluster
Delete the EKS Cluster using eksctl
```
eksctl delete cluster --name eks-cluster 
```
Wait till completed before proceeding.  

### Delete IAM Policy
Delete the IAM Policy we created initially.
```
alb_ingress_controller=`aws iam list-policies \
                        --query 'Policies[?PolicyName==\`alb-ingress-controller\`].Arn' \
                        --output text`
aws iam delete-policy --policy-arn ${alb_ingress_controller}
```
### Delete EC2 Instance
#### AWS EC2 Dashboard
Terminate "eks_cloud_shell" Instance  

## References
AWS EKS Fargate QuickStart  
https://github.com/kskalvar/aws-eks-cluster-fargate-quickstart

ALB Ingress Controller on Amazon EKS  
https://docs.aws.amazon.com/eks/latest/userguide/alb-ingress.html



