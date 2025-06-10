

---

# 🚀 Getting Your App Live on AWS EKS: A Practical Walkthrough

> Deploy the classic 2048 game using Amazon EKS, AWS Fargate, and the AWS Load Balancer Controller.

This walkthrough will guide you through setting up an Amazon EKS cluster, deploying a containerized app, using Fargate for serverless pod hosting, and exposing your application to the internet using an Application Load Balancer (ALB).

> 💡 Inspired by [Abhishek Veeramalla's video walkthrough](https://www.youtube.com/watch?v=RRCrY12VY_s)

---

## 🧰 Prerequisites

Make sure the following CLI tools are installed and configured:

* **kubectl** – Interact with Kubernetes clusters
* **eksctl** – Manage EKS clusters easily
* **AWS CLI** – Interact with AWS services from the command line

Create your EKS Fargate cluster (replace `demo-cluster` and region if needed):

```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate
```

---

## 🎮 Step 1: Deploy the 2048 Game App

Apply the Kubernetes manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/examples/2048/2048_full.yaml
```

> Note: This deploys into the `game-2048` namespace.

---

## 🧱 Step 2: Serverless Pods with AWS Fargate

Create a Fargate profile for the namespace:

```bash
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name game-2048-profile \
    --namespace game-2048
```

Check pod status:

```bash
kubectl get pods -n game-2048
```

---

## 🌐 Step 3: Expose the App with AWS Load Balancer Controller

### 3.1 Set Up IAM OIDC Provider

```bash
export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

if ! aws iam list-open-id-connect-providers | grep -q $oidc_id; then
    eksctl utils associate-iam-oidc-provider --cluster $cluster_name --approve
else
    echo "OIDC provider already exists."
fi
```

### 3.2 Create IAM Policy and Service Account

**a.** Download the policy document:

```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
```

**b.** Add the following permission block to `iam_policy.json` under the `"Statement"` array:

```json
{
  "Effect": "Allow",
  "Action": [
    "elasticloadbalancing:DescribeListenerAttributes",
    "elasticloadbalancing:DescribeListeners",
    "elasticloadbalancing:DescribeLoadBalancers",
    "elasticloadbalancing:DescribeRules",
    "elasticloadbalancing:DescribeTags",
    "elasticloadbalancing:DescribeTargetGroups",
    "elasticloadbalancing:DescribeTargetHealth",
    "elasticloadbalancing:ModifyListener",
    "elasticloadbalancing:ModifyRule",
    "elasticloadbalancing:ModifyTargetGroupAttributes",
    "elasticloadbalancing:ModifyLoadBalancerAttributes"
  ],
  "Resource": "*"
}
```

**c.** Create the IAM policy:

```bash
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json
```

**d.** Create the service account and attach the policy:

```bash
export aws_account_id=$(aws sts get-caller-identity --query Account --output text)

eksctl create iamserviceaccount \
  --cluster=$cluster_name \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$aws_account_id:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

### 3.3 Install the Controller with Helm

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
```

Set your VPC ID (you can find it in the AWS Console or via CLI):

```bash
export vpc_id="vpc-xxxxxxxxxxxxxxxxx"
```

Install the controller:

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$cluster_name \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$region \
  --set vpcId=$vpc_id
```

### 3.4 Verify Controller Status

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

Look for `AVAILABLE` in the output.

---

## 🌍 Step 4: Access the 2048 Game

Once the ALB is provisioned, fetch the external URL:

```bash
kubectl get ingress game-2048 -n game-2048
```

Open the `ADDRESS` value in a web browser and enjoy the game!

---

## ✅ Wrapping Up

You’ve now:

* Created an EKS cluster with Fargate
* Deployed a containerized app
* Exposed it via ALB using the AWS Load Balancer Controller

🎯 From here, you can explore:

* Adding HTTPS with ACM
* Configuring custom domains
* CI/CD integrations
* Monitoring and scaling

---

## 🙌 Contributions & Feedback

Got questions or improvements? Feel free to open an issue or PR — or share your tips via Discussions!

---


