AWS Elastic Kubernetes Service (EKS) Fargate QuickStart  
=======================================================
Abstract:
```
Although AWS EKS has been available for quite a while and AWS EKS Fargate has finally been released, there is
some configuration still required to get the Fargate worker nodes to work with the cluster.  In this QuickStart
will use the eksctl utility AWS has provided to create the EKS Cluster.  
```
This solution shows how to create an AWS EKS Cluster with Fargate support and deploy a simple web application with an external Application Load Balancer.  This readme updates an article "Getting Started with Amazon EKS" referenced below and provides a more basic step by step process.  It uses CloudFormation and cloud-init scripts we created to do more of the heavy lifting required to setup the cluster.  
```
Note:  This how-to assumes you are creating the eks cluster in us-east-1, you have access to your AWS Root
Account, and you can login to an EC2 Instance remotely.
```
Steps:  
* [Create an EC2 Instance](#create-an-ec2-instance)
* [Create EKS Cluster IAM Security and ALB Ingress Controller](#create-eks-cluster-iam-security-and-alb-ingress-controller)  
* [Deploy WebApp to Your Cluster](#deploy-webapp-to-your-cluster)
* [Configure the Kubernetes Dashboard](#configure-the-kubernetes-dashboard)  
* [Remove AWS EKS Cluster](#remove-aws-eks-cluster)  
* [References](#references)   


To make this first microservice easy to deploy we'll use a docker image located in DockerHub at kskalvar/web.  This image is nothing more than a simple webapp that returns the current ip address of the container it's running in.  We'll create an external AWS Application Load Balancer and you should be able to see a unique ip address as it is load balanced across containers.

The project also includes the Dockerfile for those interested in the configuration of the actual application or to build your own and deploy using ECR.

## Create an EC2 Instance 
We'll using an EC2 instance to install kubectl, eksctl, create the EKS Cluster, and worker nodes.  This is a step by step process.

### AWS EC2 Dashboard

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

### Connect to EC2 Instance
Using ssh from your local machine, connect to your AWS EC2 Instance
```
ssh -i <AWS EC2 Private Key> ec2-user@<AWS EC2 Instance IP Address>
```

### Check to insure cloud-init has completed
See contents of "/tmp/install-eks-support" it should say "installation complete".

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
Will be using eksctl to create the cluster and the iam security provider.  Additionally the cloud-init script  
install this project from github and copied the scripts will use into the /home/ec2-user.  
```
eksctl create cluster --name eks-cluster --zones=us-east-1c,us-east-1b,us-east-1a --version 1.14 --fargate
eksctl utils associate-iam-oidc-provider --cluster eks-cluster --approve
./configure-alb-ingress-controller
```

### Test Cluster
Using kubectl test the cluster status
```
kubectl get svc 
```
### Test Cluster Nodes
Use kubectl to test status of cluster nodes
```
kubectl get nodes
```

## Deploy WebApp to Your Cluster
You will need to ssh into the AWS EC2 Instance you created above. This is a step by step process.  

### Create a AWS EKS Fargate Profile
Will use eksctl to create a unique Fargate Profile for this webapp.  This also assoicates a namespace with the profile
which we will use when deploying the webapp below.    
```
eksctl create fargateprofile --name web --namespace web-namespace --cluster eks-cluster
```

### Deploy Web App
Deploy the webapp to the cluster using the Fargate Profile and Namespace we created above.
```
NOTE: There is also a script called "configure-web-ingress" in /home/ec2-user to configure the web-ingress.yaml.
It requires the AWS VPC Public Subnets used by the cluster and can only be known after the cluster is created.

Additionall the Application Load Balancer may take a few minutes to provision.
```
Use kubectl to create the web service
```
kubectl apply -f web-namespace.yaml

kubectl apply -f web-deployment.yaml
kubectl apply -f web-service.yaml

./configure-web-ingress
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
Using your client-side browser enter the following URL
```
http://<ADDRESS>
```

### Delete Deployment, Service, Namespace
Use kubectl and eksctl to delete application.
```
kubectl delete -f web-ingress.yaml
kubectl delete -f web-service.yaml
kubectl delete -f web-deployment.yaml
kubectl delete -f web-namespace.yaml
```
### Delete Fargate Profile
Use eksctl to delete profile.
```
eksctl delete fargateprofile --name web --cluster eks-cluster
```

## Configure the Kubernetes Dashboard
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
alb_ingress_controller=`aws iam list-policies --query 'Policies[?PolicyName==\`alb-ingress-controller\`].Arn' --output text`
aws iam delete-policy --policy-arn ${alb_ingress_controller}
```

### Delete EC2 Instance
#### AWS EC2 Dashboard
Terminate "eks_cloud_shell" Instance  


## References
AWS EKS Fargate QuickStart  
https://github.com/kskalvar/aws-eks-cluster-fargate-quickstart  

AWS Summit Slides for EKS  
https://www.slideshare.net/AmazonWebServices/srv318-running-kubernetes-with-amazon-eks  

Kubernetes  
https://kubernetes.io  

AWS EKS Getting Started  
https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html  


How do I set up the ALB Ingress Controller on an Amazon EKS cluster for Fargate?  
https://aws.amazon.com/premiumsupport/knowledge-center/eks-alb-ingress-controller-fargate/


