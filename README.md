# google-micro

## Exposing Frontend Service on EKS

### Prerequisites
- EKS cluster running
- kubectl configured
- eksctl installed
- Helm installed

### Steps to Expose Frontend Service

#### 1. Install AWS Load Balancer Controller CRDs
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/config/crd/bases/elbv2.k8s.aws_ingressclassparams.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/config/crd/bases/elbv2.k8s.aws_targetgroupbindings.yaml
```

#### 2. Create IAM Policy
```bash
curl -o /tmp/iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file:///tmp/iam_policy.json
```

#### 3. Create IAM Service Account
```bash
eksctl create iamserviceaccount \
  --cluster=google-microservices \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::954945275811:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=ap-south-1
```

#### 4. Get VPC ID
```bash
aws eks describe-cluster --name google-microservices --region ap-south-1 --query 'cluster.resourcesVpcConfig.vpcId' --output text
```

#### 5. Install AWS Load Balancer Controller
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=google-microservices \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=vpc-02da7bda85c143ac2
```

#### 6. Tag Subnets for Load Balancer Discovery
```bash
aws ec2 create-tags --resources subnet-069dc48efc3f6bbc5 --tags Key=kubernetes.io/role/elb,Value=1 --region ap-south-1
aws ec2 create-tags --resources subnet-029bd96c15cff55ac --tags Key=kubernetes.io/role/elb,Value=1 --region ap-south-1
```

#### 7. Annotate Frontend Service
```bash
kubectl annotate svc frontend-external service.beta.kubernetes.io/aws-load-balancer-scheme=internet-facing --overwrite
```

#### 8. Verify Service
```bash
kubectl get svc frontend-external
```

The service will now have an EXTERNAL-IP with an AWS NLB hostname that customers can use to access the application.
