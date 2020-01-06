# Cloudformation templates for EKS running on spot instances 
This repo contains the templates you need to get an EKS cluster up and running and savin some money using spot instances.

## Step 0: Install kubectl and aws-iam-authenticator
Follow the steps in:
https://docs.aws.amazon.com/eks/latest/userguide/managing-auth.html

## Step 1: Create a VPC
Deploy a stack of _cf-templates/vpc-setup.yaml_.

The stack will create a new VPC, with public and private subnets in all three availability zones. 

Specify to create 0, 1 or 3 NAT-Gateways. NAT-Gateways are coupled to availability zones, therefore specify 1 Gateway for develop or UAT environments, and choose 3 for redundancy if the setup is for production.

## Step 2: Create an EKS Cluster
Deploy a stack of _cf-templates/eks-cluster.yaml_.

The stack creates the resources:
* An EKS Cluster with a security group and service role 
* A security group the be used by the nodegroups
* A nodegroup instance role the be used by the nodegroups
* Ingress and egress rules to enable secure traffic between the EKS Cluster and the node group, and to allow the nodes to communicate with each other. 

Update your context by adding the created Cluster:
```
aws eks --region YOUR_REGION update-kubeconfig --name YOUR_CLUSTERNAME
```

Verify you can connect to the created cluster:
```
kubect get svc
```

### Step 3: Configure the Auth ConfigMap
To grant additional AWS users or roles the ability to interact with your cluster, you must edit the aws-auth ConfigMap within the cluster. This includes the roles used by the nodegroups.

Add the IAM Role used by the nodegroups by modifying the _k8s/aws-auth-cm.yaml_ file.
You can find the node instance role arn in the outputs of the deployed eks-cluster stack.

When the _k8s/aws-auth-cm.yaml_ file has been modified with the node instance role arn:
```
kubectl apply -f k8s/aws-auth-cm.yaml
```

## Step 4: Deploy a Load Balancer
Deploy a stack of _cf-templates/load-balancer.yaml_.

The stack creates the resources:
* An Application load balancer with the specified scheme. 
* A security group attached to the ALB that accepts all incoming traffic
* An Ingress rule that is added to the nodegroups security group, to allow all traffic from the load balancer on the specified ingress port.

The internet-facing scheme option will deploy the load balancer in the public subnets, while internal deploys it in the private subnets.

## Step 5: Deploy a Nodegroup
Create a key-pair using the AWS console.

Deploy a stack of _cf-templates/eks-spot-group.yaml_. 

The stack creates the resources:
* A LaunchTemplate
* An Auto scaling group
* Cloudwatch alarms and a scale policy on CPU and Memory. Deploy a cloudwatch agent to utilize the memory alarm. 

_Cloudwatch alarms should not have the main responsibility of scaling the nodegroup, this should be done by the cluster-autoscaler service:_
https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler

The User data script labels the instances with matching labels: _lifecycle=Ec2Spot_ and _lifecycle=OnDemand_.

Verify the nodes have been added to the cluster:
```
kubectl get nodes
```

## Step 6: Test the setup
Deploy both the httpbin deployment and service templates:
```
kubectl apply -f k8s/httpbin-deploy.yaml 
kubectl apply -f k8s/httpbin-svc.yaml 
```

Make a GET request to the Load Balancer URL via CURL or your browser.

