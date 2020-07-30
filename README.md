# Free IBM Cloud Kubernetes hosting for beginners

How to use IBM Cloud Kubernetes(30days Free), Cloudflare DNS(and flarectl), Let's Encrypt(and cert-manager), and Isito Ingress Gateway(and VirtualService) for Powershell windows user.

## Objectives

Ensure to deploy the Bookinfo Istio example application with the below public URL.  
https://sample.example.com:${node_port}/productpage

> ⚠️ This document is written for the Windows Powershell users.
You only need the Windows to run it. because you may not have the WSL environment.

## Prerequisites

* IBM Cloud Account (Free)  
Go to https://cloud.ibm.com/ and sign up for a free account. Your credit card information is needed for launch the Kubernetes cluster. However, you will not be charged for this tutorial.  

> ⚠️ Please set spending notification to avoid anormaly billings. https://cloud.ibm.com/billing/spending-notifications

* Your domain name ($8.03/year- )  
If you don't have domains. Go to https://cloudflare.com/ and buy a cheap domain name ($8.03/year for .com)

* Cloudflare Account (Free)  
Go to https://cloudflare.com/ and sign up free account. Please create the zone as your domain name, then update your domain registrar's Name server 1,2 to Cloudflare's them.  

*OR*

* AWS Account for Route53  


## Setup IBM Kubernetes cluster

### Value definitions explanation

#### Values for tools

| Key  |  Sample value | Description |
| - | - |- |
| $home | | User home folder |
| $env:userprofile/go | | Go language |
| $env:userprofile/scoop | | scoop |
| $env:KUBECONFIG | $home/.kube/config-cluster-01.yaml | kubeconfig file |
| $home/.kube | | Kubernetes configuration folder |
| $letsenctypt_email | yourmailaddress@example.com | Let's Encrypt email address |
| $node_ip | 123.123.123.123 | Node IP of Istio Ingress Gateway IP |
| $node_port | 30861 | HTTPS port of Istio Ingress Gateway |
| $region | eu-de | IBM Cloud Kuberbetes Cluster Region |
| $domain | example.com | Your domain |
| $subdomain | sample | Created subdomain |

#### Values (Cloudflare)

| Key  |  Sample value | Description |
| - | - |- |
| $env:CF_API_EMAIL | yourmailaddress@example.com | Cloudflare Email address|
| $env:CF_API_KEY | ****** | Cloudflare API Key |
| $cloudflare_email | yourmailaddress@example.com | |

#### Values (AWS Route53)

| Key  |  Sample value | Description |
| - | - |- |
| $env:AWS_ACCESS_KEY_ID | ****** | AWS Access Key |
| $env:AWS_SECRET_ACCESS_KEY | ****** | AWS Secret Access Key |
| $env:AWS_REGION | us-east-2 | AWS Region |

#### Files

| File  | Description |
| - | - |
| bookinfo/ | BookInfo application |
| bookinfo/overlays/bookinfo-gateway-patch.yaml | Overwright to bookinfo/base/bookinfo-gateway.yaml with your domain |
| $home/.kube/cluster-01-apikey.json | IBM Cloud API Key |
| aws-route53.json | "A" record add/modify to AWS Route53  |
| ${env:KUBECONFIG}.bak | Backup of kubeconfig file |
| clusterissuer-letsencrypt-prod.yaml | Your cert-manager ClusterIssuer |
| pilot-k8s.yaml | Istio install parameters, reduced request cpu and memory |

### Create the API key for CLI

The API key is used by the CLI login.  
Top Menu -> Manage/Access(IAM)/API keys -> Create IBM Cloud API key(e.g. cluster-01) and download key as json file.

> ⚠️ Rename and move apikey.json to a safe place (e.g. $home/kube/cluster-01-apikey.json)

> ℹ️ Created secret files will be located on the $home/.kube directory.


```
move ~/Downloads/apikey.json $home/.kube/cluster-01-apikey.json
```

### Create the Kubernetes (30days Free) cluster

Make sure you have selected a plan "Free" cluster. Usually, the cluster created in the us-south(Dallas) region, however sometimes creates it in another region.  

> ⚠️ The free cluster is automatically forced to delete after 30 days without charges. Free!  
> ⚠️ It's recommended to upgrade to 1.18 for master and worker nodes.

> ℹ️ In this document use the cluster name as 'cluster-01' and Default resource group.

## Install tools

### Install Scoop and jq(1.6), go(1.14.4), kubectl (1.18.5), helm(3.2.4), and IBM Cloud CLI+Plugin

```powershell
iwr -useb get.scoop.sh | iex
scoop bucket add extras
scoop install jq go 7zip kubectl helm ibmcloud-cli
ibmcloud plugin install kubernetes-service
mkdir $home/.kube
```

> ℹ️ If you get an error you might need to change the execution policy (i.e. enable Powershell) with

```powershell
Set-ExecutionPolicy RemoteSigned -scope CurrentUser
```

### Install istioctl (1.6.5)

```powershell
curl -kLO https://github.com/istio/istio/releases/download/1.6.4/istioctl-1.6.5-win.zip
7z x istioctl-1.6.5-win.zip
cp istioctl.exe $env:userprofile\scoop\shims <# Copy to any path on $env:PATH #>
```

### Install flarectl (Cloudflare)

I'm using Cloudflare. flarectl is Cloudflare's official tool. To get flarectl.exe should install go lang.  
Please set PATH for flarectl.exe (e.g. $env:userprofile/go/bin)  
Additinal information is https://github.com/cloudflare/cloudflare-go/tree/master/cmd/flarectl  

> ⚠️ Please set environment values:  
> - $env:CF_API_EMAIL="******" <- Cloudflare login  
> - $env:CF_API_KEY="4a83******9106" <- Cloudflare Global API Key  
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
ibmcloud login --apikey $(cat $home/.kube/cluster-01-apikey.json | jq -r .apikey) -r $region -g Default -q
```

> ℹ️ There are many other login methods besides the API Key.

### Set KUBECONFIG

The kubeconfig YAML file will be overwritten by the IBM cloud CLI command.

```powershell
$env:KUBECONFIG="$home/.kube/config-cluster-01.yaml"
if (Test-Path $env:KUBECONFIG) {
  move $env:KUBECONFIG "${env:KUBECONFIG}.bak"
}
ibmcloud ks cluster config -c cluster-01
```

### Check cluster and public IP address of the node

```powershell
kubectl get no -o wide
```
```text
NAME            STATUS   ROLES    AGE   VERSION       INTERNAL-IP     EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
10.144.183.23   Ready    <none>   86m   v1.18.6+IKS   10.144.183.23   169.51.203.243   Ubuntu 16.04.6 LTS   4.4.0-185-generic   containerd://1.3.4
$node_ip=ibmcloud ks workers -c cluster-01 --worker-pool default --json | jq -r .[0].publicIP
```

### Set Your subdomain domain to values

```
$domain="example.com" <# Your domain #>
$subdomain="sample"
```

### Assign URL to node_ip address (Cloudflare)

Add "A" record with node IP address to DNS. I'm using Cloudflare. If you use AWS Route53, Plase set via AWS CLI or AWS Console.

```powershell
$dns_id=flarectl --json dns list --zone="$domain" --type="A" --name="$subdomain.$domain" | jq -r .[].ID
if ($dns_id) {
  flarectl dns update --id $dns_id --zone "$domain" --content "$node_ip"
} else {
  flarectl dns create --zone "$domain" --type="A" --name="$subdomain.$domain" --content "$node_ip"
}
```

### Assign URL to node_ip address (AWS Route53)

```
$dns_id=aws route53 list-hosted-zones-by-name --dns-name "$domain" --query "HostedZones[?Name=='$domain.'].Id" --output text
$action = "CREATE"
if ((aws route53 list-resource-record-sets --hosted-zone-id $dns_id `
  --query "ResourceRecordSets[?Name=='$subdomain.$domain.'].ResourceRecords[0].Value" --output text).length -ne 0) {
  $action = "UPSERT"
}
@"
{"Changes": [{
  "Action": "$action",
  "ResourceRecordSet": {
    "Name": "$subdomain.$domain", "Type": "A", "TTL": 300, "ResourceRecords": [{ "Value": "$node_ip" }]
  }
}]}
"@ | Set-Content aws-route53.json

aws route53 change-resource-record-sets --hosted-zone-id $dns_id --change-batch file://aws-route53.json
```

### Ping check

```
ipconfig /flushdns
ping -n 3 "$subdomain.$domain" <# Your subdomain.domain #>
```

## Install applications

### Install kubed

You can use secrets across the namespace with kubed. It will be associated in base/namespace.yaml as the label "app=kubed".

```powershell
helm repo add appscode https://charts.appscode.com/stable/
helm install kubed appscode/kubed --namespace kube-system --wait
```

### Install Istio

```powershell
istioctl install -f pilot-k8s.yaml
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

#### Set the API key to Kubernetes secret (Cloudflare)

```powershell
$env:CF_API_EMAIL="******"
$env:CF_API_KEY="4a83******9106"
$letsenctypt_email="yourmailaddress@example.com"
$cloudflare_email="yourmailaddress@example.com"

kubectl -n cert-manager create secret generic cloudflare-api-key --from-literal=secret-access-key=$env:CF_API_KEY
kubectl -n cert-manager get secret cloudflare-api-key -o yaml     
```
```text
apiVersion: v1
data:
  api-key.txt: NGE******g==
kind: Secret
metadata:
  ...
  name: cloudflare-api-key
  namespace: cert-manager
  ...
```

#### Set the API key to Kubernetes secret (AWS Route53)

```powershell
$env:AWS_ACCESS_KEY_ID="AKI******"
$env:AWS_SECRET_ACCESS_KEY="******"
$env:AWS_REGION="us-east-2" <# Your AWS region #>
$letsenctypt_email="yourmailaddress@example.com"

kubectl -n cert-manager create secret generic prod-route53-credentials-secret --from-literal=secret-access-key=$env:AWS_SECRET_ACCESS_KEY
kubectl -n cert-manager get secret prod-route53-credentials-secret -o yaml
```

```text
apiVersion: v1
data:
  secret-access-key: NGE******g==
kind: Secret
metadata:
  ...
  name: prod-route53-credentials-secret
  namespace: cert-manager
  ...
```

#### Deploy Cluster Issuer (Cloudflare)

```powershell
@"
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $letsenctypt_email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        cloudflare:
          email: $cloudflare_email
          apiKeySecretRef:
            name: cloudflare-api-key
            key: api-key.txt
"@ | Set-Content clusterissuer-letsencrypt-prod.yaml
```

> ⚠️ This manifest is using Let's Encrypt production environment. Please be aware of the rate limits.  
> https://letsencrypt.org/docs/rate-limits/

#### Deploy Cluster Issuer (AWS Route53)

```powershell
@"
apiVersion: cert-manager.io/v1alpha2
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: $letsenctypt_email
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - dns01:
        route53:
          region: $env:AWS_REGION
          accessKeyID: $env:AWS_ACCESS_KEY_ID
          secretAccessKeySecretRef:
            name: prod-route53-credentials-secret
            key: secret-access-key
"@ | Set-Content clusterissuer-letsencrypt-prod.yaml
```

- Deploy

```powershell
kubectl apply -f clusterissuer-letsencrypt-prod.yaml
kubectl describe clusterissuer letsencrypt-prod
```

Check logs (optional)

```powershell
kubectl -n cert-manager logs -l app=cert-manager -c cert-manager
```

#### Deploy Certificate Request

It will add a TXT DNS record to the Cloudflare/Route53 DNS (DNS01 challenge)  

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
  commonName: '*.$domain'
  dnsNames:
  - '$domain'
  - '*.$domain'
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
"@ | Set-Content istio-ingressgateway-certs.yaml
```

- Deploy

```powershell
kubectl apply -f istio-ingressgateway-certs.yaml
```

Check logs (optional)

```
kubectl -n cert-manager logs -l app=cert-manager -c cert-manager
```
```text
I0706 06:31:20.368769       1 dns.go:133] cert-manager/controller/challenges/Check "msg"="waiting DNS record TTL to allow the DNS01 record to propagate for domain" "dnsName"="******.com" "domain"="******.com" "resource_kind"="Challenge" "resource_name"="istio-ingressgateway-certs-2862066918-2138811768-146870716" "resource_namespace"="istio-system" "type"="dns-01" "fqdn"="_acme-challenge.******.com." "ttl"=60
...
I0706 06:33:45.362372       1 sync.go:102] cert-manager/controller/orders "msg"="Order has already been completed, cleaning up any owned Challenge resources" "resource_kind"="Order" "resource_name"="istio-ingressgateway-certs-2862066918-2138811768" "resource_namespace"="istio-system"
```

Check request result.

```powershell
kubectl -n istio-system describe certificaterequest
```
```text
Events:
  Type    Reason             Age    From          Message
  ----    ------             ----   ----          -------
  Normal  OrderCreated       2m50s  cert-manager  Created Order resource istio-system/istio-ingressgateway-certs-2862066918-2138811768
  Normal  CertificateIssued  12s    cert-manager  Certificate fetched from issuer successfully
```

> ℹ️ For AWS Route53, If you got failed results, Please check [IAM policy](https://cert-manager.io/docs/configuration/acme/dns01/route53/).

#### Export certificate (optional)

The backup certificate will be used for importing later.

```powershell
kubectl -n istio-system get secret istio-ingressgateway-certs -o yaml > $home/.kube/istio-ingressgateway-certs-exports.yaml
```

#### Import (optional)

If certificate cration is failed, You can import the backup certificate.

```powershell
kubectl apply -f istio-ingressgateway-certs-exports.yaml
```

#### Annotate Certificate Secret for using it across namespaces

Annotate cert secret for kubed. It will be associated in base/namespace.yaml as the label "app=kubed".

```powershell
kubectl annotate secret istio-ingressgateway-certs -n istio-system kubed.appscode.com/sync="app=kubed"
```

#### Istio Ingress Gateway Certificate validation

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

### Deploy BookInfo application

- Modify kustomize patch for your domain

```powershell
@"
- op: replace
  path: "/spec/servers/0/hosts/0"
  value: "$subdomain.$domain"
"@ | Set-Content bookinfo/overlays/bookinfo-gateway-patch.yaml
```

- Deploy

```powershell
kubectl apply -k bookinfo/overlays
kubectl -n bookinfo get po -w
```
```text
NAME                              READY   STATUS    RESTARTS   AGE
details-v1-558b8b4b76-j2tff       2/2     Running   0          78s
productpage-v1-6987489c74-j4tcg   2/2     Running   0          77s
ratings-v1-7dc98c7588-2l9kq       2/2     Running   0          77s
reviews-v1-7f99cc4496-m7gwq       2/2     Running   0          76s
reviews-v2-7d79d5bd5d-t5gkq       2/2     Running   0          75s
reviews-v3-7dbcdcbc56-txc57       2/2     Running   0          75s
```

> ⚠️ Deploy with overridden file "bookinfo/overlays/bookinfo-gateway-patch.yaml" by your domain. The bookinfo/base directory contain basic setting as template.

### Check the BookInfo URL

Access URL with the Istio Ingress node port by Web browsers or curl.

```powershell
$node_port=$(kubectl -n istio-system get svc istio-ingressgateway --output jsonpath='{.spec.ports[?(@.name==\"https\")].nodePort}')
curl -kv "https://${subdomain}.${domain}:${node_port}/productpage"

start "$env:programfiles (x86)\Google\Chrome\Application\Chrome.exe" "https://${subdomain}.${domain}:${node_port}/productpage"
```

### Delete BookInfo application

```powershell
kubectl delete -k bookinfo/overlay
```
