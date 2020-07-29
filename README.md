# Free IBM Cloud Kubernetes hosting for beginner

How to use IBM Cloud Kubernetes(30days Free), Cloudflare DNS(and flarectl), Let's Encrypt(and cert-manager), and Isito Ingress Gateway(and VirtualService) for Powershell windows user.

## Objectives

Ensure to deploy the Bookinfo Istio example application with the below public URL.  
https://sample.example.com:${node_port}/productpage

## Prerequisites

* IBM Cloud Account (Free)  
Go to https://cloud.ibm.com/ and sign up for a free account. Your credit card information is needed for launch the Kubernetes cluster. However, you will not be charged for this tutorial.  
Please set spending notification to avoid anormaly billings. https://cloud.ibm.com/billing/spending-notifications

* Cloudflare Account (Free)  
Go to https://cloudflare.com/ and sign up free account.

* Your domain name ($8.03/year- )  
If you don't have domains. Go to https://cloudflare.com/ and buy a cheap domain name ($8.03/year for .com)

> ℹ️ Created secret files will be located on the ~/.kube directory.

## Setup IBM Kubernetes cluster

### Create the API key for CLI

The API key is used by the CLI login.  
Top Menu -> Manage/Access(IAM)/API keys -> Create IBM Cloud API key(e.g. cluster-01) and download key as json file.

> ⚠️ Rename and move apikey.json to a safe place (e.g. ~/kube/cluster-01-apikey.json)

```
move ~/Downloads/apikey.json ~/.kube/cluster-01-apikey.json
```

### Create the Kubernetes (30days Free) cluster

Make sure you have selected a plan "Free" cluster. Usually, the cluster created in the us-south(Dallas) region, however sometimes creates it in another region.  

> ⚠️ The free cluster is automatically forced to delete after 30 days without charges. Free!  
> ⚠️ It's recommended to upgrade to 1.18 for master and worker nodes.

> ℹ️ In this document use the cluster name as 'cluster-01'  

## Install tools

### Install Scoop and jq(1.6), go(1.14.4), kubectl (1.18.5), helm(3.2.4)

```powershell
iwr -useb get.scoop.sh | iex
scoop bucket add extras
scoop install jq go 7zip kubectl helm ibmcloud-cli
```

> ℹ️ if you get an error you might need to change the execution policy (i.e. enable Powershell) with

```powershell
Set-ExecutionPolicy RemoteSigned -scope CurrentUser
```

### Install istioctl (1.6.5)

```powershell
curl -kLO https://github.com/istio/istio/releases/download/1.6.4/istioctl-1.6.5-win.zip
7z x istioctl-1.6.5-win.zip
cp istioctl.exe $env:userprofile\scoop\shims <# Copy to any path on $env:PATH #>
```

### Install flarectl (optional)

I'm using Cloudflare. flarectl is Cloudflare's official tool. To get flarectl.exe should install go lang.  
Please set PATH for flarectl.exe (e.g. $env:userprofile/go/bin)  
Additinal information is https://github.com/cloudflare/cloudflare-go/tree/master/cmd/flarectl  

> ⚠️ Please set environment values:  
> - $env:CF_API_EMAIL="******" # Cloudflare login  
> - $env:CF_API_KEY="4a83******9106" # Cloudflare Global API Key  
> Please access https://dash.cloudflare.com/profile/api-tokens to get Cloudflare Global API Key.

```powershell
$env:CF_API_EMAIL="******"
$env:CF_API_KEY="4a83******9106"
go get -u -v github.com/cloudflare/cloudflare-go
go get -u -v github.com/cloudflare/cloudflare-go/cmd/flarectl
flarectl zone list
```

## IBM Cloud Login

### Login

Please check the cluster which region running on. If the cluster is running on us-south, replace to $region="us-south".

```powershell
$region="eu-de"
ibmcloud login --apikey $(cat ~/.kube/cluster-01-apikey.json | jq -r .apikey) -r $region -g Default -q
```

> ℹ️ There are many other login methods besides the API Key.

### Set KUBECONFIG

The kubeconfig YAML file will be overwritten by the IBM cloud CLI command.

```powershell
ibmcloud plugin install kubernetes-service
mkdir $home/.kube
$env:KUBECONFIG="$home/.kube/config-cluster-01.yaml"
move $env:KUBECONFIG ${env:KUBECONFIG}.bak
ibmcloud ks cluster config -c cluster-01
```

### Check cluster and public IP address of the node

```powershell
kubectl get no -o wide
```
```text
NAME            STATUS   ROLES    AGE   VERSION       INTERNAL-IP     EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
10.144.183.23   Ready    <none>   86m   v1.18.6+IKS   10.144.183.23   169.51.203.243   Ubuntu 16.04.6 LTS   4.4.0-185-generic   containerd://1.3.4
$node_ip=$(ibmcloud ks workers -c cluster-01 --worker-pool default --json | jq -r .[0].publicIP)
```

#### Assign URL to node_ip address

I'm using Cloudflare.

```powershell
$cname="sample"
$cf_zone="example.com" <# Your domain #>
$dns_id=flarectl --json dns list --zone="$cf_zone" --type="A" --name="$cname.$cf_zone" | jq -r .[].ID
if ($dns_id) {
  flarectl dns update --id $dns_id --zone "$cf_zone" --content "$node_ip"
} else {
  flarectl dns create --zone "$cf_zone" --type="A" --name="$cname.$cf_zone" --content "$node_ip"
}
ipconfig /flushdns
ping sample.example.com <# Your domain #>
```

## Install applications

### Install kubed

You can use secrets across the namespace with kubed.

```powershell
helm repo add appscode https://charts.appscode.com/stable/
helm install kubed appscode/kubed --namespace kube-system
```

### Install istio

```powershell
istioctl install -f bookinfo/istio/pilot-k8s.yaml
kubectl get po -A
```
```text
NAMESPACE      NAME                                         READY   STATUS    RESTARTS   AGE
ibm-system     addon-catalog-source-sndz7                   1/1     Running   0          8d
ibm-system     catalog-operator-645796fbdf-b5q5k            1/1     Running   0          8d
ibm-system     olm-operator-7bf4dbc978-lvfz5                1/1     Running   0          8d
istio-system   istio-ingressgateway-cc7f64cb-krfqx          1/1     Running   0          37s
istio-system   istiod-5c8bd665c7-skmk8                      1/1     Running   0          52s
istio-system   prometheus-7db4d5f66-jnpb9                   2/2     Running   0          37s
kube-system    calico-kube-controllers-656c5785dd-twvjc     1/1     Running   0          8d
kube-system    calico-node-zpzjg                            1/1     Running   0          8d
kube-system    coredns-7b56dd58f7-7cmmj                     1/1     Running   0          8d
kube-system    coredns-7b56dd58f7-cf6p8                     1/1     Running   0          8d
kube-system    coredns-7b56dd58f7-ngz89                     1/1     Running   0          8d
kube-system    coredns-autoscaler-777bf994bf-t27k4          1/1     Running   0          8d
kube-system    dashboard-metrics-scraper-76756886dc-5vmzv   1/1     Running   0          8d
kube-system    ibm-keepalived-watcher-rzcfl                 1/1     Running   0          8d
kube-system    ibm-master-proxy-static-10.76.68.138         2/2     Running   0          8d
kube-system    kubed-599cb4b8bc-2fbzk                       1/1     Running   0          85m
kube-system    kubernetes-dashboard-5c898cc4d-bsldd         1/1     Running   2          8d
kube-system    metrics-server-58d6cb57cd-pfbrv              2/2     Running   0          8d
kube-system    vpn-79845b6f9d-mpclq                         1/1     Running   0          8d
```

> ℹ️ Why use the customize parameter in pilot-k8s.yaml? This is to minimize the memory definition required for startup. it's important in the free cloud.

### Install certmanager

```powershell
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.15.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
kubectl create namespace cert-manager
helm install `
  cert-manager jetstack/cert-manager `
  --namespace cert-manager `
  --version v0.15.1
kubectl -n cert-manager get po
```
```text
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-7747db9d88-qll5c             1/1     Running   0          22s
cert-manager-cainjector-87c85c6ff-bvtmx   1/1     Running   0          22s
cert-manager-webhook-64dc9fff44-gc2b5     1/1     Running   0          22s
```

#### Set the Cloudflare API key to Kubernetes secret

```powershell
$env:CF_API_EMAIL="******"
$env:CF_API_KEY="4a83******9106"
$api_key=[convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("$env:CF_API_KEY"))

@"
apiVersion: v1
kind: Secret
metadata:
  name: cloudflare-api-key
  namespace: cert-manager
type: Opaque
data:
  api-key.txt: $env:API_KEY
"@ | Set-Content ~/.kube/cloudflare-api-key.yaml

kubectl apply -f ~/.kube/cloudflare-api-key.yaml
kubrctl -n cert-manager describe secret cloudflare-api-key
```
```text
Name:         cloudflare-api-key
Namespace:    cert-manager
Labels:       <none>
Annotations:
Type:         Opaque

Data
====
api-key.txt:  37 bytes
```

#### Deploy cluster issuer

Replace below 2 fields with your mail address for the Let's Encrypt issuer.

```powershell
@"
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: youremail@example.com # Your enmail address
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        cloudflare:
          email: youremail@example.com # Your enmail address
          apiKeySecretRef:
            name: cloudflare-api-key
            key: api-key.txt
"@ | Set-Content clusterissuer-letsencrypt-prod.yaml
```

- Deploy clster issuer

```powershell
kubectl apply -f clusterissuer-letsencrypt-prod.yaml
kubectl describe clusterissuer letsencrypt-prod
kubectl -n cert-manager logs -l app=cert-manager -c cert-manager
```

#### Deploy certificate request

It will set TXT DNS record on Cloudflare (DNS-01 DNS name ownership check)  
Replace below 3 fields with your domain for the let's encrypt certificate.

```powershell
@"
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  name: istio-ingressgateway-certs
  namespace: istio-system
spec:
  secretName: istio-ingressgateway-certs
  duration: 2160h # 90d
  renewBefore: 360h # 15d
  commonName: '*.example.com' # Your Domain
  dnsNames:
  - 'example.com' # Your Domain
  - '*.example.com' # Your Domain
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
"@ | Set-Content istio-ingressgateway-certs.yaml
```

- Deploy

```powershell
kubectl apply -f istio-ingressgateway-certs.yaml
kubectl -n cert-manager logs -l app=cert-manager -c cert-manager
```
```text
I0706 06:31:20.368769       1 dns.go:133] cert-manager/controller/challenges/Check "msg"="waiting DNS record TTL to allow the DNS01 record to propagate for domain" "dnsName"="lifeinreno.com" "domain"="lifeinreno.com" "resource_kind"="Challenge" "resource_name"="istio-ingressgateway-certs-2862066918-2138811768-146870716" "resource_namespace"="istio-system" "type"="dns-01" "fqdn"="_acme-challenge.lifeinreno.com." "ttl"=60
...
I0706 06:33:45.362372       1 sync.go:102] cert-manager/controller/orders "msg"="Order has already been completed, cleaning up any owned Challenge resources" "resource_kind"="Order" "resource_name"="istio-ingressgateway-certs-2862066918-2138811768" "resource_namespace"="istio-system"
```
```powershell
kubectl -n istio-system describe certificaterequest
```
```text
Events:
  Type    Reason             Age    From          Message
  ----    ------             ----   ----          -------
  Normal  OrderCreated       2m50s  cert-manager  Created Order resource istio-system/istio-ingressgateway-certs-2862066918-2138811768
  Normal  CertificateIssued  12s    cert-manager  Certificate fetched from issuer successfully
---
```

#### Export certificate (optional)

The backup certificate will be used for importing later.

```powershell
kubectl -n istio-system get secret istio-ingressgateway-certs -o yaml > ~/.kube/istio-ingressgateway-certs-exports.yaml
```

#### Import (optional)

If certificate cration is failed, You can import the backup certificate.

```powershell
kubectl apply -f istio-ingressgateway-certs-exports.yaml
```

#### Annotate certificate secret for using it across namespaces

Annotate it fo kubed.

```powershell
kubectl annotate secret istio-ingressgateway-certs -n istio-system kubed.appscode.com/sync="app=kubed"
```

#### Istio Ingress Gateway certificate validation

Validate istio ingress gateway's pre-defined mount point /etc/istio/ingressgateway-certs

```powershell
kubectl exec -it -n istio-system $(k -n istio-system get pods -l istio=ingressgateway -o jsonpath='{.items[0].metadata.name}') -- ls -al /etc/istio/ingressgateway-certs
```
```text
total 4
drwxrwxrwt 3 root root  140 Jul  6 06:34 .
drwxr-xr-x 1 root root 4096 Jul  6 03:33 ..
drwxr-xr-x 2 root root  100 Jul  6 06:34 ..2020_07_06_06_34_02.220388696
lrwxrwxrwx 1 root root   31 Jul  6 06:34 ..data -> ..2020_07_06_06_34_02.220388696
lrwxrwxrwx 1 root root   13 Jul  6 06:31 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root   14 Jul  6 06:31 tls.crt -> ..data/tls.crt
lrwxrwxrwx 1 root root   14 Jul  6 06:31 tls.key -> ..data/tls.key
```

## Istio Ingress Gateway test

### Deploy bookinfo application

- Modify kustomize patch for your domain

```powershell
@"
- op: replace
  path: "/spec/servers/0/hosts/0"
  value: "sample.example.com" # Your Domain
"@ | Set-Content bookinfo/overlays/bookinfo-gateway-patch.yaml
```

- Deploy

```powershell
kubectl apply -k bookinfo/overlays
kubectl -n bookinfo get po -w
```
```text
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-558b8b4b76-k5gsx       1/1     Running   0          69s
productpage-v1-6987489c74-bfh7m   1/1     Running   0          68s
ratings-v1-7dc98c7588-x944t       1/1     Running   0          67s
reviews-v1-7f99cc4496-lhh94       1/1     Running   0          67s
reviews-v2-7d79d5bd5d-75nzz       1/1     Running   0          66s
reviews-v3-7dbcdcbc56-4967d       1/1     Running   0          66s
```

### Check the BookInfo URL

Access URL with the Istio Ingress node port by Web browsers or curl.

```powershell
$node_port=$(kubectl -n istio-system get svc istio-ingressgateway --output jsonpath='{.spec.ports[?(@.name==\"https\")].nodePort}')
curl -v "https://sample.example.com:${node_port}/productpage" <# Your domain #>
```

### Delete bookinfo application

```powershell
kubectl delete -k bookinfo/overlay
```
