# What is the difference between Role, RoleBinding, ClusterRole, and ClusterRoleBinding in terms of namespace?

Role: Namespace-level resource (namespace = true)
RoleBinding: Namespace-level resource (namespace = true)
ClusterRole: Cluster-level resource (namespace = false)
ClusterRoleBinding: Cluster-level resource (namespace = false)


# What are the steps to create and use RBAC resources in Kubernetes?
We will follow structure for project.
First create namespace expense.

# k8-admin\namespace\expense\expense.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: expense
  labels:
    name: expense
    environment: dev
    

# k8-admin\namespace\expense\rbac.yaml

First we will give read access to pod

creating a Role called expense-pod-reader i.e Role
attaching this role to someone like harish   i.e RoleBinding


apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: expense
  name: expense-pod-reader-role
rules:
- apiGroups: [""] #" indicates the core API group"
  resources: ["pods"]
  verbs: ["get","watch","list"]

---
apiVersion: rbac.authorization.k8s.io/v1
#This role binding allows "jane" to read pods in the "default" namespace.
#You need to already have a Role named "pod-reader" in that namespace.
kind: RoleBinding
metadata:
  name: expense-pod-reader-rolebinding
  namespace: expense
subjects:
# You can specify more than one "subject"
- kind: User
  name: harish #"name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
 # 'roleRef' specifies the binding to a Role/ClusterRole
 kind: Role #this must be Role or ClusterRole
 name: expense-pod-reader-role #this must match the name of the Role or ClusterRole you wish
 apiGroup: rbac.authorization.k8s.io


Note: How EKS knows to check AWS for Role and RoleBinding Authorization.
EKS should connect to AWS IAM

First we need to Authentication for EKS.
https://docs.aws.amazon.com/eks/latest/userguide/auth-configmap.html


# k8-admin\namespace\expense\aws-auth.yaml
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::730335449147:role/eksctl-expense-1-nodegroup-spot-NodeInstanceRole-x53FJRVzJ6KD
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - groups:
      - expense-pod-reader-role
      userarn: arn:aws:iam::730335449147:user/harish
      username: harish
kind: ConfigMap
metadata:
  creationTimestamp: "2025-01-25T14:14:00Z"
  name: aws-auth
  namespace: kube-system
  uid: 34e2effb-2213-4446-b35d-cd9a4ab383b0




kubectl get configmap aws-auth -n kube-system


kubectl get configmap aws-auth -n kube-system -o yaml



apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::730335449147:role/eksctl-expense-1-nodegroup-spot-NodeInstanceRole-x53FJRVzJ6KD
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - groups:
      - expense-pod-reader-role
      userarn: arn:aws:iam::730335449147:user/harish
      username: harish
kind: ConfigMap
metadata:
  creationTimestamp: "2025-01-25T14:14:00Z"
  name: aws-auth
  namespace: kube-system
  uid: 34e2effb-2213-4446-b35d-cd9a4ab383b0


To replace specific user arn, go to AWS IAM --> users-->harish --> get harish arn value

Now, Admin will email project team that namespace and access is given. Then


The project team has their own server like expense servers like creating ec2 instance.
Login into ec2-user  as harish user.

Harish wants to login AWS using 'aws configure'
Access Key:
Secret Access Key:
Region:

He can check his login status:
$aws sts get-caller-identity

Create and attach ClusterDecribe permmision in AWS IAM harish user.
o create eks:DescribeCluster permission for an IAM user in AWS, you need to create and attach an IAM policy with that permission.

Here’s a step-by-step guide using both AWS Console and AWS CLI:

✅ Option 1: Using the AWS Console
Sign in to the AWS Management Console as a user with permission to manage IAM.

Navigate to IAM > Policies.

Click Create policy.

Choose the JSON tab and paste the following:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster"
      ],
      "Resource": "arn:aws:eks:us-east-1:484907532817:cluster/expense"
    }
  ]
}

Click Next: Tags (optional), then Next: Review.

Name it something like: EKSDescribeClusterPolicy

Click Create policy.

Then:
Go to IAM > Users > harish.

Click the Permissions tab.

Click Add permissions > Attach policies directly.

Search for EKSDescribeClusterPolicy, select it, and click Next > Add permissions.


We have cluster describe access, so he can get kubeconfig file access.
$aws eks update-kubeconfig --region us-east-1 --name expense

kubectl get nodes

need to install kubectl here.
---------------------------------------
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.30.0/2024-05-12/bin/linux/amd64/kubectl 
sudo chmod +x ./kubectl 
sudo mv kubectl  /usr/local/bin/ 
kubectl version --client 

$kubect get nodes

Error from server (Forbidden): nodes is forbidden: User "harish" cannot list resource "nodes" in API group "" at the cluster scope.

Note: if we observe above error, Authentication worked here, need to work on authorization by giving role access to him

$kubectl get pods -n expense
No resources found in expense namespace.

We have given access to harish only on expense namespace, not on default namespace.

Upto now Harish got access namespace level to access pods.

Now We will give cluster access to Harish by creating cluster and assigning using ClusterBinding.

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
# 'namespace' omitted since ClusterRole are not namespaced
  name: expense-cluster-reader-role
rules:
- apiGroups: [""]
  #
  # at the HTTP level, the name of the resource for accessing Secret
  #objects in "secrets"
  resources: ["secrets","persistentvolumes","nodes"]
  verbs: ["get","watch","list"]

---
apiVersion: rbac.authorization.k8s.io/v1
#This cluster role binding allows anyone in the "manager" group to read secrets in any namespace
kind: ClusterRoleBinding
metadata:
  name: expense-cluster-reader-rolebinding
subjects:
- kind: User
  name: harish #Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: expense-cluster-reader-role
  apiGroup: rbac.authorization.k8s.io


$kubectl get nodes
$kubectl get secrets
$kubectl get pv
$kubectl get pvc   --> this is in default namespace, so not accessible.

namespace is true, resource is namespace level
namespace is false, then resource is cluster level.

1. harish has access to eks describe cluster or not
2. get aws-auth configmap
3. user harish authenticated
4. check role and role bindings

Note: In a cluster, there are multiple namespaces. if a user gets one cluster access, then he can able to access all its namespaces.





