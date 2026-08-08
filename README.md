# Kubernetes & AWS EKS DevOps Deployment Project

A hands-on DevOps/Kubernetes project demonstrating Kubernetes core
resources, container deployment, AWS EKS, Amazon ECR, Jenkins CI/CD, and
AWS Application Load Balancer (ALB) ingress.

## Project Overview

This project started with Kubernetes fundamentals such as Pods,
ReplicaSets, Deployments, and Services, and was extended into a
practical AWS EKS deployment workflow.

The production-style workflow is:

``` text
Developer / Git
      |
      v
   Jenkins
      |
      +--> Docker Build
      |
      +--> Amazon ECR
      |
      v
   Amazon EKS
      |
      v
 Kubernetes Deployment
      |
      +--> Frontend Pod 1
      +--> Frontend Pod 2
      |
      v
 ClusterIP Service
      |
      v
 AWS Load Balancer Controller
      |
      v
 Application Load Balancer
      |
      v
    Internet
```

## Technologies Used

-   Kubernetes
-   Amazon EKS
-   Amazon ECR
-   AWS Application Load Balancer
-   AWS Load Balancer Controller
-   Docker
-   Jenkins
-   AWS CLI
-   kubectl
-   Helm
-   Git / GitHub
-   Nginx

## Repository Kubernetes Examples

The repository contains Kubernetes manifests for learning the main
workload and networking objects:

  -----------------------------------------------------------------------
  File                                Purpose
  ----------------------------------- -----------------------------------
  `pods.yaml`                         Creates a basic Nginx Pod

  `replicaset.yaml`                   Demonstrates ReplicaSet-based pod
                                      management

  `deployment.yaml`                   Creates an Nginx Deployment with
                                      multiple replicas

  `services.yaml`                     Exposes Nginx using a Kubernetes
                                      Service

  `na.yaml`                           Additional Kubernetes practice
                                      manifest
  -----------------------------------------------------------------------

The current repository examples use Nginx to demonstrate Kubernetes
concepts. The EKS deployment described below extends those concepts to a
container image stored in Amazon ECR.

## Kubernetes Concepts Demonstrated

### Pod

A Pod is the smallest deployable Kubernetes object. The project includes
a basic Nginx Pod running on port 80.

``` bash
kubectl apply -f pods.yaml
kubectl get pods
kubectl describe pod nginx-pod
```

### ReplicaSet

ReplicaSets maintain a desired number of identical Pod replicas.

``` bash
kubectl apply -f replicaset.yaml
kubectl get replicasets
```

### Deployment

A Deployment provides declarative application rollout and ReplicaSet
management.

The repository deployment example runs multiple Nginx replicas.

``` bash
kubectl apply -f deployment.yaml
kubectl get deployments
kubectl get pods
```

### Service

The repository includes a NodePort example exposing Nginx on port
`30080`.

``` bash
kubectl apply -f services.yaml
kubectl get svc
```

For the AWS EKS + ALB deployment, a `ClusterIP` Service is preferred
with ALB IP target mode.

## AWS EKS Deployment Architecture

The EKS frontend deployment uses:

-   Namespace: `development`
-   Deployment: `frontend`
-   Replicas: `2`
-   Container port: `80`
-   ECR repository: `frontend`
-   Service: `frontend-service`
-   Service type: `ClusterIP`
-   Ingress class: `alb`
-   ALB scheme: `internet-facing`
-   ALB target type: `ip`
-   AWS region: `ap-south-1`

Example ECR image:

``` text
463605761106.dkr.ecr.ap-south-1.amazonaws.com/frontend:<BUILD_NUMBER>
```

## Example EKS Manifest

``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: development

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: development
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: 463605761106.dkr.ecr.ap-south-1.amazonaws.com/frontend:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 20

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: development
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: frontend-ingress
  namespace: development
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80}]'
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/success-codes: "200-399"
spec:
  ingressClassName: alb
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

## AWS Load Balancer Controller

Add and update the EKS Helm repository:

``` bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

Install the controller after creating the required service account/IAM
permissions:

``` bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=demo-cluster-1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --version 1.14.0
```

Verify:

``` bash
kubectl get deployment aws-load-balancer-controller -n kube-system
kubectl get pods -n kube-system
kubectl get ingressclass
```

## Deploy to EKS

Configure kubectl:

``` bash
aws eks update-kubeconfig --region ap-south-1 --name demo-cluster-1
kubectl get nodes
```

Apply the manifest:

``` bash
kubectl apply -f k8s.yaml
```

Verify:

``` bash
kubectl get pods -n development -o wide
kubectl get svc -n development
kubectl get endpointslices -n development
kubectl get ingress -n development
kubectl describe ingress frontend-ingress -n development
```

A successful Ingress receives an AWS ALB DNS name.

## Jenkins CI/CD Pipeline

The CI/CD pipeline automates:

1.  Checkout source code.
2.  Verify AWS CLI, Docker, and kubectl.
3.  Authenticate with AWS.
4.  Login to Amazon ECR.
5.  Build the Docker image.
6.  Tag the image with the Jenkins build number and `latest`.
7.  Push images to ECR.
8.  Update EKS kubeconfig.
9.  Update the Kubernetes Deployment image.
10. Wait for rollout completion.
11. Verify Pods, Service, Ingress, and Deployment.

Important ECR login pattern:

``` bash
aws ecr get-login-password --region ap-south-1 | \
docker login --username AWS --password-stdin \
463605761106.dkr.ecr.ap-south-1.amazonaws.com
```

Login uses the ECR registry hostname. The `/frontend` repository path is
used for image tags and pushes.

Example deployment command:

``` bash
kubectl set image deployment/frontend \
frontend=463605761106.dkr.ecr.ap-south-1.amazonaws.com/frontend:${BUILD_NUMBER} \
-n development
```

Verify rollout:

``` bash
kubectl rollout status deployment/frontend -n development --timeout=300s
```

## Troubleshooting

### EKS endpoint DNS error

Example:

``` text
dial tcp: lookup <cluster>.eks.amazonaws.com: no such host
```

Check:

``` bash
aws eks describe-cluster --name demo-cluster-1 --region ap-south-1 \
  --query "cluster.{Status:status,Endpoint:endpoint}"

aws eks update-kubeconfig --name demo-cluster-1 --region ap-south-1

kubectl get nodes
```

Also verify whether the EKS API endpoint is public or private.

### Jenkins: `aws: not found`

Verify AWS CLI for the Jenkins user:

``` bash
sudo -u jenkins aws --version
```

If AWS CLI exists in a non-standard location, expose it through the
Jenkins PATH or create an appropriate executable link.

### ECR login non-TTY error

If `aws ecr get-login-password` fails, `docker login --password-stdin`
receives no password. Fix the AWS CLI/credential issue first.

### ALB created but website unavailable

Check:

``` bash
kubectl get pods -n development
kubectl get svc -n development
kubectl get endpointslices -n development
kubectl describe ingress frontend-ingress -n development
```

Then inspect the AWS target group and confirm the Pod IP targets on port
80 are `Healthy`.

### NodePort already allocated

If a manually selected NodePort such as `30080` is already used, either
remove the explicit `nodePort` and let Kubernetes allocate one, or
select an unused port.

## Useful Commands

``` bash
kubectl get nodes
kubectl get pods -A
kubectl get pods -n development -o wide
kubectl get deployment -n development
kubectl get svc -n development
kubectl get ingress -n development
kubectl describe ingress frontend-ingress -n development
kubectl logs -n development deployment/frontend
kubectl rollout status deployment/frontend -n development
kubectl rollout undo deployment/frontend -n development
```

## Project Outcomes

This project demonstrates practical experience with:

-   Writing Kubernetes YAML manifests
-   Pods, ReplicaSets, Deployments, Services, and Ingress
-   Running workloads on Amazon EKS
-   Storing Docker images in Amazon ECR
-   Configuring AWS ALB ingress
-   Using Helm for Kubernetes add-ons
-   Jenkins CI/CD automation
-   Kubernetes rolling updates and rollback
-   Debugging DNS, ECR authentication, Services, endpoints, and ALB
    health

## Future Improvements

-   Add HTTPS using AWS Certificate Manager (ACM)
-   Add a custom domain with Route 53
-   Add Horizontal Pod Autoscaler (HPA)
-   Add CloudWatch/Prometheus monitoring
-   Add separate staging and production namespaces
-   Add Kubernetes ConfigMaps and Secrets
-   Add automated smoke tests after deployment
-   Replace long-lived AWS credentials with an EC2 IAM role where
    appropriate

## Author

**Rohit Kolekar**

DevOps / Cloud / AWS learning project.

## Repository

GitHub repository: `rohitkolekar840-a11y/k8s`
