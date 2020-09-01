# Setup RBAC with Konvoy based k8s cluster

## Prerequisites

### This installation guide was tested with the following components:

- Konvoy 1.4 or higher 
- Install and configure `kubectl` and `kubectx` cli-tools 
  Here is a link for kubectx https://github.com/ahmetb/kubectx
- Access to K8s cluster with Aurhorization mode as `RBAC` find the link below on how to do that.
  

1. Validate that you have admin access to k8s api-server:

```bash
# Validate that you can run kubectl against the api-server.
kubectl get pods -n kube-system

# Check that you have admin access to the k8s cluster.
kubectl config view
```
Output:
```bash
[centos@ip-10-0-1-198 demo-cluster]$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://demo-cluster-9b1f-lb-control-46693284.us-west-2.elb.amazonaws.com:6443
  name: demo-cluster
contexts:
- context:
    cluster: demo-cluster
    user: demo-cluster-admin
  name: demo-cluster-admin@demo-cluster
current-context: demo-cluster-admin@demo-cluster
kind: Config
preferences: {}
users:
- name: demo-cluster-admin
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

2. Provision a service account for a an individual member ex john-smith

```bash
kubectl create serviceaccount john-sa
```
Output:
```bash
centos@ip-10-0-1-198 demo-cluster]$ kubectl create serviceaccount john-sa
serviceaccount/john-sa created
```

3. Bind the service account to the appropriate roles to grant privileges for actions you desire. In this case we are binding user John to an existing cluster role with edit permissions

```bash
kubectl create clusterrolebinding john-sa-binding --clusterrole=edit --serviceaccount=default:john-sa
```
Output:
```bash
[centos@ip-10-0-1-198 demo-cluster]$ kubectl create clusterrolebinding john-sa-binding --clusterrole=edit --serviceaccount=default:john-sa
clusterrolebinding.rbac.authorization.k8s.io/john-sa-binding created
```

#### Note
A. Whenever we create a service account, K8s api server creates a token for this service account by default so that this Service account can authenticate itself to the api-server. We are going to extract this token to access the cluster for external access. 

B. Above example uses default SA use this link to get creative and set up further custome roles as per your needs 
https://kubernetes.io/docs/reference/access-authn-authz/rbac/
`

4. Verify the secrets exist & Retrieve the token
```bash
kubectl get secrets
```
Output:
```bash
[centos@ip-10-0-1-198 demo-cluster]$ kubectl get secrets
NAME                  TYPE                                  DATA   AGE
default-token-x5nqv   kubernetes.io/service-account-token   3      38m
john-sa-token-xbtpv   kubernetes.io/service-account-token   3      4m11s
```

5. Verify the token is generated for service account by describing the secret be sure to use your own secret name
```bash
kubect describe secret <secret-name>
```
Output:

```bash
[centos@ip-10-0-1-198 demo-cluster]$ kubectl describe secret john-sa-token-xbtpv
Name:         john-sa-token-xbtpv
Namespace:    default
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: john-sa
              kubernetes.io/service-account.uid: 6a7a78bf-d347-485f-b715-d24b692f4fd1

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  7 bytes
token:      eyJhbGciOiJSdY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiam9obi1zYSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjZhN2E3OGJmLWQzNDctNDg1Zi1iNzE1LWQyNGI2OTJmNGZkMSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYW . . . [output snipped]
```

6. Add the token as an environmental variable(This would make reduce the hassale of copy pasting it over and over )
```bash
export TOKEN=$(kubectl get secret john-sa-token-rq4ls -o=jsonpath="{.data.token}" | base64 -d -i -)
```

7. Review your kubeconfig file, notice that it has only 1 user and 1 cluster defined in it. 
```bash
cat ~/.kube/config
```
Output:
```bash
[centos@ip-10-0-1-198 demo-cluster]$ cat ~/.kube/config
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBbVFhZ2ovZU1XYz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo= . . . [output snipped]
    server: https://demo-cluster-9b1f-lb-control-46693284.us-west-2.elb.amazonaws.com:6443
  name: demo-cluster
contexts:
- context:
    cluster: demo-cluster
    user: demo-cluster-admin
  name: demo-cluster-admin@demo-cluster
current-context: demo-cluster-admin@demo-cluster
kind: Config
preferences: {}
users:
- name: demo-cluster-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM4akNDBVEUtLS0tLQo= . . . [output snipped]
```

8. Since we are going to be messing with config file that's vital for our line of communication with k8s cluster lets take a backup 
```bash
cp ~/.kube/config ~/.kube/config.old
```

9. Configure your kubeconfig file to include the service accounts you included. Below is the sample and make sure to include the token you generate to the token section. Since we are working with the same cluster you might need to add only new user john and his context like below

```bash
vim  ~/.kube/config
```
Note: You will need to add a context and user at the bottom as we are using the same cluster. Below you can see how it should look 

Output:
```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EY3dPVEl6TWpjd01Wb1hEVE13TURjd056SXpNamN3TVZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTFNUCm1Xdm0wQ2NhbEMyaEw1RGhUVkJvVjZPaGR3d3B6c1BxaUJVcWNmV0UvRzlqTWMzcFVZc1JSMWlNbU0ycWF2VWIKaHZ0VnUxcW1WMXFpK1dLMGRQeEhPOXRwOU5XTm5pYm41Q3MrNVAvN3U2cUhxemlNckNIbFZwYjkrekZhdnRQSgpzWlFINUFUK0YrSzdUYkNpaHNpcG1BczMwclpINFNMYUJXSDF6VzJnMkszaXBNbHMzM2hPS043dkJWNEdoYXNlCnlPbU9xMW1NVHFZak5aSE01Yk5IbXNjdjlIcVQ5YkF4cU9PdEhPVzYrZWtlNkhteU9VdHpVa3UvV3VkaC9XdUIKbTh4RWl2UVBSUHVNak5xR3Z5RlZQTkl0aVFnb0U4QUFXaE1JZENLUC9kT05PVzNZRXgzb1RrN3JkNXhqZy91VQpoRVRuSUVMMWY5cVhIT2pmZ1JjQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFFV1ZKYnEzMkVVc1U2M0lTWUJUQndqd1NyakEKYnYzOFp0dHlpenZYei82SHA2L0picXVFRCtkd1FneEpmYkZVY2FHT1U4eUtEYnM3emtSTlAvV0c0Z0tRb2hCTApOam1IZVVFc1ZXZnFoNEZOOUs5Z1F4WG5NbXo2cTd2TUVOVzhuMGRIU1FHY2wxanJGTGdNN1Z1Q2RPL0dEVTNxCmVEYm5xb2g5NkNMWDZCNUhCRTRBMUw5TDE4RTBYMTl4aTR0bXZiTm5uUFE2UXY2OU4zVGVLRzRrVUx3Zis1M3QKMzFFc0xxMlI2Y3QzRFhBcTRJdSs4R1cxUTdDSVJjVUpZSmtja3Y2YlFkWVVnNUlrVVk3NzVjeEFZbVdMRlRXagpyTHNZa3BEbzVhYXFOQ1dpMG9jNlZFUlFvUmxuOVcwMDZaMk14Zi9TMnhMZnUyMzZ5bVFhZ2ovZU1XYz0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://demo-cluster-9b1f-lb-control-46693284.us-west-2.elb.amazonaws.com:6443
  name: demo-cluster
contexts:
- context:
    cluster: demo-cluster
    user: demo-cluster-admin
  name: demo-cluster-admin@demo-cluster
- context:
    cluster: demo-cluster
    user: john-sa
  name: john-sa@demo-cluster
current-context: john-sa@demo-cluster
kind: Config
preferences: {}
users:
- name: demo-cluster-admin
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk. . . [output snipped]
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcFFJQkFBS0NBUUVBdUtzMl. . . [output snipped]
- name: john-sa
  user:
    token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImFvOXlMTUJvSUEzSjRCeG5BLTNvZk15anB. . . [output snipped]
```
Here another sample which showcases slighty different kind of kubeconfig file which leverages different kinds of authentication methods. 
```bash
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
- cluster:
    insecure-skip-tls-verify: true
    server: https://5.6.7.8
  name: scratch
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: storage
    user: developer
  name: dev-storage
- context:
    cluster: scratch
    namespace: default
    user: experimenter
  name: exp-scratch
current-context: ""
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
- name: experimenter
  user:
    password: some-password
    username: exp
```

10. Once you have updated the kubeconfig file. You can validate list of contexts available to you using `kubectx`
```bash

kubectx
```
Output:
```bash
[centos@ip-10-0-1-198 demo-cluster]$ kubectx
demo-cluster-admin@demo-cluster
john-sa@demo-cluster
```
11. Lets switch the context using kubectx 
```bash
kubectx john-sa@demo-cluster
```
Run the kubectx command again to validate that you are now using John's context. It should be highlighted 
```bash
kubectx
```
Output:
```bash
[centos@ip-10-0-1-198 demo-cluster]$ kubectx
demo-cluster-admin@demo-cluster
john-sa@demo-cluster
```
12. Run commands to validate that you can query the resources

```bash
kubectl auth can-i get deployments

kubectl get pods 
```
13. You can also use the token to make http calls to k8s api
```bash
curl -H "Authorization: Bearer $TOKEN" https://api.cluster-address/api/v1/pods -k
```
