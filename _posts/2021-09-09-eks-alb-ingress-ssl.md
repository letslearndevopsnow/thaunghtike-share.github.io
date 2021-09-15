---
layout:     post
title:   "Setting Up TLS Encryption On Amazon EKS With ALB Ingress Controller"
date:       2021-09-15 15:13:18 +0200
image: 30.jpg
tags:
    - Kubernetes
    - EKS
categories: kubernetes
---   

<h2> Introduction </h2>

In this tutorial, I' gonna show you how to set up tls encryption on eks cluster with alb ingress controller. I already wrote a post about ssl nginx controller on my blog. You can read it [here](https://thaunghtike-share.github.io/2021/07/30/letencrypt-ssl-gke). In order for the Ingress resource to work, the cluster must have an ingress controller running. There are a lot of ingress controllers such as nginx, traefik and alb.

<h2> Prerequities </h2>

<ul>
    <li> eksctl </li>
    <li> eks cluster </li>
    <li> public hosted zone </li>
    <li> domain </li>
</ul>

<h2> Install Eksctl Binary </h2>

Before creating an eks cluster, we have to install eksctl binary to perform eks cluster operaions.

```bash
$ curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
$ sudo mv /tmp/eksctl /usr/local/bin
$ eksctl version
```
<h2> Create An EKS Cluster </h2>
 
Firstly we have to create an eks cluster named eksdemo1 in us-east-1. It takes 15-20 minutes depending on your connection. You can go to aws cloudformation stack and you can see what resources are deployed.
 
 ```bash
 $ eksctl create cluster --name=eksdemo1 --region=us-east-1 --zones=us-east-1a,us-east-1b --without-nodegroup
 ```
 <h2> Create & Associate IAM OIDC Provider for our EKS Cluster </h2>
 
 To enable and use AWS IAM roles for Kubernetes service accounts on our EKS cluster, we must create & associate OIDC identity provider. To do so using eksctl we can use the below command. 
 
 ```bash
 $ eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve
```    
Yeah you got an eks cluster. Let's create a node group with two worker nodes to join new created cluster named eksdemo1. It also takes 15-20 minutes to fully create a node group.
 
 ```bash
 $ eksctl create nodegroup --cluster=eksdemo1 --region=us-east-1  --name=eksdemo1-ng-public1 --node-type=t3.medium --nodes=2 --nodes-min=2                       --nodes-max=4 --node-volume-size=20 --ssh-access --ssh-public-key=aws --managed --asg-access --external-dns-access --full-ecr-access                       --appmesh-access
 ```
 Now, you've created an eks cluster. Check with 'eksctl get clusters'. You will see one cluster is running. 
 
 ```bash
 $ kubectl get nodes
NAME                             STATUS   ROLES    AGE    VERSION
ip-192-168-12-223.ec2.internal   Ready    <none>   112s   v1.20.7-eks-135321
ip-192-168-34-20.ec2.internal    Ready    <none>   88s    v1.20.7-eks-135321
```
<h2> Create A Kubernetes Service Account For Alb Ingress Controller</h2>

Then we have to create a k8s service account named alb-ingress-controller in kube-system namespace. This service account will help eks cluster to create and delete elastic load balancers.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/rbac-role.yaml
$ kubectl get sa alb-ingress-controller -n kube-system -o yaml
```
<h2> Create IAM Policy for ALB Ingress Controller </h2>

This IAM policy will allow our ALB Ingress Controller pod to make calls to AWS APIs.

```bash
$ aws iam create-policy \
    --policy-name ALBIngressControllerIAMPolicy \
    --policy-document https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/iam-policy.json
```
There is an issue in this policy. So we need to edit this policy. Go to Services -> IAM -> Policies. Click Edit Policy >> Visual Editor on this policy. Add ELB full access: Click on Add Additional Permissions >> Service: ELB >> Actions: All ELB actions (elasticloadbalancing:*) >> Resources: All Resources. Remove ELB which has warning.Then click on review policy.

<h2> Create an IAM role for the ALB Ingress Controller </h2>

This will create an IAM role for the alb ingress controller using created ALB ingress policy above. And this IAM role attachs to the service account named alb-ingress-controller which you created early on this post. Replace cluster name and policy arn with your cluster name and policy arn.

```bash
$ eksctl create iamserviceaccount \
    --region us-east-1 \
    --name alb-ingress-controller \
    --namespace kube-system \
    --cluster eksdemo1 \
    --attach-policy-arn arn:aws:iam::993450297386:policy/ALBIngressControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```
<h2> Verify using eksctl cli </h2>

```bash
$ eksctl  get iamserviceaccount --cluster eksdemo1
```
<h2> Verify k8s Service Account </h2>

You can see that newly created Role ARN is added in Annotations to k8s SA account. It proves that AWS IAM role bounds to alb-ingress-controller service account.

```bash
$ kubectl get sa alb-ingress-controller -n kube-system -o jsonpath={.metadata.annotations}
```
<h2> Deploy ALB Ingress Controller </h2>

You are ready to deploy alb ingress controller on eks cluster. This alb controller deployment uses alb-ingress-controller SA account to perform ELB operations.

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes-sigs/aws-alb-ingress-controller/master/docs/examples/alb-ingress-controller.yaml
```
You will see alb ingress pod going to be 'crashloopbackoff'. You have to edit this deployment using kubectl. Add your eks cluster name '--cluster-name=eksdemo1' to the container arguments. 

```bash
$  kubectl edit deployment.apps/alb-ingress-controller -n kube-system
```
Wait for a couple of minutes. ALB ingress controller pod is running now.

```bash
$ thaunghtikeoo@thaunghtikeoo:~$ kubectl get all -n kube-system
NAME                                          READY   STATUS    RESTARTS   AGE
pod/alb-ingress-controller-7f699ff874-q5xsq   1/1     Running   0          103s
```
<h2> Verify our ALB Ingress Controller is running </h2>

You can check alb ingress controller is working or not. Check the logs of alb controller pod. Otherwise, if ALB have not created well then you see something is wrong.

```bash
$ kubectl logs -f $(kubectl get po -n kube-system | egrep -o 'alb-ingress-controller-[A-Za-z0-9-]+') -n kube-system

1 leaderelection.go:214] successfully acquired lease kube-system/ingress-controller-leader-alb
I0915 15:12:57.217703       1 controller.go:134] kubebuilder/controller "level"=0 "msg"="Starting Controller"  "controller"="alb-ingress-controller"
I0915 15:12:57.318066       1 controller.go:154] kubebuilder/controller "level"=0 "msg"="Starting workers"  "controller"="alb-ingress-controller" "worker count"=1
```
<h2> Create Nginx Deployment </h2>

You are ready to create ingress routes on this eks cluster. Let's create a sample nginx deployment for this demo using kubectl.

```bash
$ kubectl create deploy nginx --image nginx --port 80
```
make sure nginx deployment is ready.

```bash
$ kubectl get deploy 
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           63s
```
<h2> Expose Nginx Service As NodePort </h2>

This step is quite important. You have to expose nginx svc as NodePort. So, alb can create a listener which will forward to target group with this NodePort.

```bash
$ kubectl expose deploy nginx --type NodePort --target-port 80 --port 80
```

<h2> Create ALB kubernetes Basic Ingress Manifest </h2>

create an ingress route to nginx service. check the following yaml. You can refer [this link](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.2/guide/ingress/annotations) for alb ingress annotations. You can define alb health checks and other alb settings using those annotations.

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
  labels:
    app: nginx
  annotations:
    # Ingress Core Settings
    kubernetes.io/ingress.class: "alb"
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: instance
spec:
  rules:
    - http:
        paths:
          - path: /*
            backend:
              serviceName: nginx
              servicePort: 80
```              
create the ingress manifest.

```bash
$ kubectl apply -f ingress.yaml
Warning: extensions/v1beta1 Ingress is deprecated in v1.14+, unavailable in v1.22+; use networking.k8s.io/v1 Ingress
ingress.extensions/nginx created
```
Wait for some minutes to create an alb. Realize your ingress is not created well after waiting for some minutes.

```bash
$  kubectl get ingress
NAME    CLASS    HOSTS   ADDRESS    PORTS    AGE
nginx   <none>   *                   80     6m10s
```
May be something is going wrong with alb controller. Check the logs again.

```bash
$ kubectl logs -f $(kubectl get po -n kube-system | egrep -o 'alb-ingress-controller-[A-Za-z0-9-]+') -n kube-system

Subnets must contains these tags: 'kubernetes.io/cluster/eksdemo1': ['shared' or 'owned'] and 'kubernetes.io/role/elb': ['' or '1']. See https://kubernetes-sigs.github.io/aws-alb-ingress-controller/guide/controller/config/#subnet-auto-discovery for more details. 
```
Due to above logs, you need to assign tags to two public subnets under eks vpc. After assigning tags to these subnets, your alb controller is working fine.

![subnet_tags]()

check ingress again. 

```bash
$  kubectl get ingress
NAME    CLASS    HOSTS   ADDRESS                                                              PORTS   AGE
nginx   <none>   *       6ad0ef5b-default-nginx-ef8b-1266263549.us-east-1.elb.amazonaws.com   80      6m50s
```
You can see an alb with one listener which forwards traffic to nginx target group is active now. 

![albconsole]()

Also you can verify target group have two registerd instances with port 30866. This is nginx service's NodePort.

![albnodeport]()

Access the alb dns on your browser. Verify nginx is running.

![nginxalb]()





