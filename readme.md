# Kubernetes user creation

<p>This guide helps you create and configure kubernetes users on both client and admin side. To enhance more security with RBAC and gain more leverage please check out <a href="https://github.com/ashokjjk/gatekeeper-k8s-library">gatekeeper</a> At the end of this demo you will be in a position to add and manage users to your kubernetes cluster</p> 
<p>A complete sample data is provided along with this repository. Instead you can create your own by following the below steps</p>
<u>Captions</u>
Client - dev, operations, user
Admin - cluster administrator

## Steps

1. Client has to use openssl to generate certificate signing request (csr) and key
2. Once the client received the key and csr from openssl, submit the csr in a k8s object to admin for authorization/signing
3. Admin after analysing the csr approves the csr
4. Admin will attach user to relevant roles and rolebinding against the relevant RBAC
5. After successfull attachment admin will issue the crt to the client
6. Client creates the kubeconfig file with the crt obtained from admin
7. Authenticate the newly created kubeconfig file, client certificates with the cluster

## How to

<b> Replace the <> with suitable texts </b>

```
openssl genrsa -out <username>.key 2048
openssl req -new -key myuser.key -out <username>.csr

cat <username>.csr | base64 | tr -d "\n"      # Extract the certificate data and encode in base64

```
- Copy the csr encoded content and use it for the request value on below k8s object and name the file as csr.yaml
```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: <username>
spec:
  groups:
  - system:authenticated
  request: <csr-base64-encoded-content>
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```
- It is important to note that kubernetes version 1.17 and below doesn't need to <b>signerName</b> attribute on the above manifest.
- Admin has to apply the csr to the cluster and approve with the below commands
```
kubectl apply -f csr.yaml
kubectl certificate approve <username> 

```
- Finally export the certificate to client
```
kubectl get csr <username> -o jsonpath='{.status.certificate}'| base64 -d > <username>.crt

```
- Admin need to apply the roles and bindings so that user can access the kubernetes resources 
```
kubectl apply -f roles.yaml
kubectl apply -f role-binding.yaml
```
- Check the role-binding.yaml file to update the relevant user parameter on line number 12.
- Also this RBAC is enforced only for the default namespace. Which means the user has freedom to do anything on othe namespaces.
- Admin can verify the user access by simulating api using the below command
```
kubectl auth can-i create deploy -n default --as <username>
```
- Last step is to add the certificate to kubeconfig for the user to access k8s
```
kubectl config set-credentials <username> --client-key=<username>.key --client-certificate=<username>.crt --embed-certs=true
kubectl config set-context <username> --cluster=<cluster-name> --user=<username>
kubectl config use-context <username>

```
## AWS Managed cluster (EKS)
- Client needs their IAM ARN to be configured inside aws-auth configmap that resides within the cluster
- Admin needs to add clients IAM ARN data under the <b>mapUsers</b> attribute
- Below command allows the admin to edit the configmap
```
kubectl edit configmap aws-auth -n kube-system -o yaml
```
- Edit the file with relevance to the below sample information
```
apiVersion: v1
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123422212211:role/sandbox-sit20210524112009155100000007
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
  mapUsers: |
    - userarn: <user-arn>
      username: <username>
      groups:
      - system:basic-user
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
  resourceVersion: "37265"
  selfLink: /api/v1/namespaces/kube-system/configmaps/aws-auth
  uid: 748bfaeb-8537-4742-bd0e-6bcf720e86c1
```
- It is necessary to understand the <b>groups</b> to find more info check <a href="https://kubernetes.io/docs/reference/access-authn-authz/rbac/#default-roles-and-role-bindings">here</a>
- Once the above is done now the client has to configure the AWS cli with their credentials
- Client has to run the below command to update their kubeconfig file with relevant certifates from the cluster
```
aws eks --region <region> update-kubeconfig --name <cluster-name>
```
- After successful configuration now it's time to run kubectl commands from the client side to access the cluster
```
kubectl get pods
kubectl get deployments
kubectl get endpoints
```

### References
1. https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/
2. https://kubernetes.io/docs/reference/access-authn-authz/rbac/#api-overview
3. https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html