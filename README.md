# Kubernetes Project on EKS

![new](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/32671b58-d7f9-4d32-8f9d-cd787c46427a)


### Prerequisites

1. **Installing the AWS CLI**:

   - Download and install the AWS CLI. You can find installation instructions for various operating systems [here](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html).

2. **Configuring AWS CLI Credentials**:

     ```
     aws configure
     ```
   - Enter the access key ID and secret access key of the IAM user you created earlier.
   - Choose a default region and output format for AWS CLI commands.

3. **Installing kubectl**:
   - Install kubectl [here](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

4. **Install eksctl**:
    - Install eksctl – A command line tool for working with EKS clusters that automates many individual tasks. For more information, see [Installing or updating]("https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html").

# Install EKS
 
## Install using Fargate

```
eksctl create cluster --name demo-cluster --region ap-south-1 --fargate
```

## Delete the cluster

```
eksctl delete cluster --name demo-cluster --region ap-south-1
```

![Screenshot 2023-10-07 012002](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/d4709f35-8924-49e6-8a05-f624ad6c1773)

![Screenshot 2023-10-07 013623](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/9fafb441-a2f4-46eb-b153-fb87097cf6eb)

### If you navigate to console you can see that cluster is created with Fargate Profile.

![Screenshot 2023-10-07 013707](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/15cacc3f-bb9e-4f47-a2d5-6499dcfb4e8a)

![Screenshot 2023-10-07 013929](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/1f66672b-1785-4b6b-8bd5-fb1347b88151)

## Configuring kubectl for EKS:
   - Once kubectl is installed, you need to configure it to work with your EKS cluster.
   - In the AWS Management Console, go to the EKS service and select your cluster.
   - Click on the "Config" button and follow the instructions to update your kubeconfig file. Alternatively, you can use the AWS CLI to update the kubeconfig file:

     ```
     aws eks update-kubeconfig --name demo-cluster --region ap-south-1
     ```

![Screenshot 2023-10-07 014652](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/ce51f88e-416f-4b99-b24e-75c20037a008)

## Create Fargate profile for 2048 App

```
eksctl create fargateprofile \
    --cluster demo-cluster \  
    --region ap-south-1 \
    --name alb-sample-app \
    --namespace game-2048
```

![Screenshot 2023-10-07 014828](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/6057c58f-ab91-46fc-9308-332b0be251dc)

## Now if you see in console Profile name alb-sample-app with Namespaces game-2048 is created.

![Screenshot 2023-10-07 014910](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/cff5187f-9c5d-4595-97a1-694f19d58dde)

## Deploy the deployment, Service and Ingress

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/examples/2048/2048_full.yaml

kubectl get pods -n game-2048

kubectl get svc -n game-2048

kubectl get ingress -n game-2048
```

![Screenshot 2023-10-07 015118](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/213e1237-cb13-476f-9040-a587da02fee9)

![Screenshot 2023-10-07 015348](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/6744e9d6-94da-48d8-a8da-28983be7f506)


## Commands to configure IAM OIDC provider 

```
export cluster_name=demo-cluster
```

```
oidc_id=$(aws eks describe-cluster --name demo-cluster --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5) 
```

## Check if there is an IAM OIDC provider configured already

- aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n 

If not, run the below command

```
eksctl utils associate-iam-oidc-provider --cluster demo-cluster --approve
```

![Screenshot 2023-10-07 015844](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/6e941d56-5b7a-46b7-9e69-e74e6a3fdc1e)

## How to setup ALB add on IAM policy

**Download IAM policy**

```
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
```

![Screenshot 2023-10-07 015948](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/efceaac7-679e-4921-a0b6-f3fd02daeabd)

**Create IAM Policy**

```
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

![Screenshot 2023-10-07 020042](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/d4a87168-5814-4bbf-9088-ebaf6f0eaee4)

**Create IAM Role**

```
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::914856326061:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

![Screenshot 2023-10-07 020319](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/7c0d1ed1-f347-46cb-81ce-04b68f4db35e)

# Deploy ALB controller


### Install helm

Install Helm from [here](https://helm.sh/docs/intro/install/)

**Add helm repo**

```
helm repo add eks https://aws.github.io/eks-charts
```

**Update the repo**

```
helm repo update eks
```

### Install aws loadbalancer controller

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=vpc-091fe59cc858b80ba
```

![Screenshot 2023-10-07 021134](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/d1b0d579-27d2-44b2-8fc9-6579a259795e)

### Verify that the application load balancer is created and there are at least 2 replicas.

```
kubectl get deployment -n kube-system aws-load-balancer-controller
```

![Screenshot 2023-10-07 021541](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/e29642ce-b09a-4d1f-b690-b7978fa827b7)

### In the below image you can see that aws-load-balancer-controller has created Load Balancer.

![Screenshot 2023-10-07 021847](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/f342de90-f9d2-445a-bd6e-906e5fc5f277)

### In the below image you can see that ingress resource has got Address of Load balancer.

```
kubectl get ingress -n game-2048
```

![Screenshot 2023-10-07 022501](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/c792215f-e2ff-4909-8722-420f3c2e82d1)

### To verify the Address go to concole and check Load Balancer DNS name.

![Screenshot 2023-10-07 022916](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/3cdf3e36-dc84-44d0-be1e-9b1e162073c2)

### Now copy the Address and paste it in the Browser.

![Screenshot 2023-10-07 023210](https://github.com/pradip2994/Kubernetes_project_on_EKS/assets/124191442/a65a132f-1ef0-43a5-8552-e03dddbf1843)

**Thank you for reading!**
