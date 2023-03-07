# Cert-manager

Cert-manager adds certificates and certificate issuers as resource types in Kubernetes clusters. It 
simplifies the process of obtaining, renewing and using those certificates.

Cert-manager issues certificates from a variety of supported sources, including Let's Encrypt, HashiCorp Vault & Venafi 
as well as private PKI.

![cert-manager img](https://cert-manager.io/images/high-level-overview.svg)

Official doc: https://cert-manager.io/docs/

# Hands-on

## Install cert-manager

First of all, use `kind` to create a new Kubernetes cluster with one master and one worker node. Apply the 
`config.yaml` manifest:

```bash
kind create cluster --config=config.yaml
```

Then, add the Helm repository required to install cert-manager:

```bash
helm repo add jetstack https://charts.jetstack.io
```

After that, update Helm's cache:

```bash
helm repo update
```

Finally, install cert-manager and its required Custom Resource Definition (CRD):
    
```bash
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0 \
  --set installCRDs=true
```

## Verify the installation

Manual verification:

```bash
kubectl get pods --namespace cert-manager
```

This should be the expected output:

    NAME                                       READY   STATUS    RESTARTS   AGE
    cert-manager-5c6866597-zw7kh               1/1     Running   0          2m
    cert-manager-cainjector-577f6d9fd7-tr77l   1/1     Running   0          2m
    cert-manager-webhook-787858fcdb-nlzsq      1/1     Running   0          2m

## Generate a test certificate

To verify the webhook is working ok, generate a new certificate using ``Issuer`` resource. 

First, create the required manifests:

```bash
cat <<EOF > test-resources.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager-test
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

Then apply them:

```bash
kubectl apply -f test-resources.yaml
```

Finally, check the status of the new certificate:

```bash
kubectl describe certificate -n cert-manager-test
```

Delete the test certificate with:

```bash
kubectl delete -f test-resources.yaml
```

# Real world implementation 

To finalize this implementation, we'll create a real world implementation using: GKE + Ingress + Lets Encrypt

So, having a new account and project is mandatory before executing these steps.

## 1 - GKE creation

### Pre-requisites

First, authenticate against GCP:

```bash
gcloud auth login
```

Enable required services:
```bash
gcloud services enable servicemanagement.googleapis.com servicecontrol.googleapis.com cloudresourcemanager.googleapis.com compute.googleapis.com container.googleapis.com containerregistry.googleapis.com cloudbuild.googleapis.com
```

### Cluster creation

Define region and cluster name:

```bash
export REGION=us-east1
export CLUSTER=cert-manager-cluster
```

Create a kubernetes cluster, it will take ~5 minutes:

```bash
gcloud container clusters create $CLUSTER --preemptible --num-nodes=1 --region=$REGION
```

Set up GKE auth plugin for kubectl:

```bash
gcloud components install gke-gcloud-auth-plugin
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
gcloud container clusters get-credentials $CLUSTER --region=$REGION
```

Check connections has been established successfully:

```bash
kubectl get nodes -o wide
```

## 2- Web-server deployment

Create a sample web server that will answer HTTP requests with *hello-world!*:
```bash
kubectl expose deployment web --port=8080
```

Expose the  web-server on port 8080:
```bash
kubectl expose deployment web --port=8080
```

## 3 - Create a static IP address

To be reachable from outside the GKE cluster, we'll need to have a static IP address that will be, later on, mapped to 
domain name.

Create the static IP address and assign the name `web-ip`:
```bash
gcloud compute addresses create web-ip --global
```

List the created IP:

```bash
gcloud compute addresses list
```

Store the IP address in an environmental variable so it can be referenced in future commands:

```bash
export IP_ADDRESS=$(gcloud compute addresses describe web-ip --format='value(address)' --global)
```

## 4 - Register a domain for the website

Usually, a regular Internet user does not remember the IP address of every service they consume but they remember their 
domain name. 

So, we'll buy and register a domain name in order to map it with the `web-ip` previously created.

## 5 - Create an Ingress

To expose the web-server to the internet, a Kubernetes Ingress object must be created. This will trigger the creation 
of several services in Google Cloud Platform to allow clients reaching it.

Initially, the Ingress will only handle HTTP requests, and then we'll add the SSL layer with cert-manager.

First, execute this command to declare an Ingress YAML manifest:
```bash
cat <<EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    # This tells Google Cloud to create an External Load Balancer to realize this Ingress
    kubernetes.io/ingress.class: gce
    # This enables HTTP connections from Internet clients
    kubernetes.io/ingress.allow-http: "true"
    # This tells Google Cloud to associate the External Load Balancer with the static IP which we created earlier
    kubernetes.io/ingress.global-static-ip-name: web-ip
spec:
  defaultBackend:
    service:
      name: web
      port:
        number: 8080
EOF
```
---
### Ingress annotations

#### kubernetes.io/ingress.class: gce
Instructs GCP to create an external HTTP Load Balancer that operates at an application level and routes based on host 
paths.

#### kubernetes.io/ingress.allow-http: "true"`
Instructs the ALB to accept Internet clients => HTTP protocol.

#### kubernetes.io/ingress.global-static-ip-name: web-ip
Instructs GCP to assign the ALB our previously created static IP (whose name was `web-ip`).

---

Then, we can apply the Ingress manifest:
```bash
kubectl apply -f ingress.yaml
```
Inspect the progress with this command:

```bash
kubectl describe ingress web-ingress
```

After ~5 minutes, our Ingress's GCP components should be properly configured. This is the expected outcome from the 
previous command:

```bash
Name:             web-ingress
Labels:           <none>
Namespace:        default
Address:          34.98.96.51
Ingress Class:    <none>
Default backend:  web:8080 (10.84.0.6:8080)
Rules:
  Host        Path  Backends
  ----        ----  --------
  *           *     web:8080 (10.84.0.6:8080)
Annotations:  ingress.kubernetes.io/backends: {"k8s1-f7d34f99-default-web-8080-c1c8d423":"Unknown"}
              ingress.kubernetes.io/forwarding-rule: k8s2-fr-xlnnyd6i-default-web-ingress-4vzrw7ji
              ingress.kubernetes.io/target-proxy: k8s2-tp-xlnnyd6i-default-web-ingress-4vzrw7ji
              ingress.kubernetes.io/url-map: k8s2-um-xlnnyd6i-default-web-ingress-4vzrw7ji
              kubernetes.io/ingress.allow-http: true
              kubernetes.io/ingress.class: gce
              kubernetes.io/ingress.global-static-ip-name: web-ip
Events:
  Type    Reason     Age                    From                     Message
  ----    ------     ----                   ----                     -------
  Normal  Sync       2m36s                  loadbalancer-controller  UrlMap "k8s2-um-xlnnyd6i-default-web-ingress-4vzrw7ji" created
  Normal  Sync       2m34s                  loadbalancer-controller  TargetProxy "k8s2-tp-xlnnyd6i-default-web-ingress-4vzrw7ji" created
  Normal  Sync       2m27s                  loadbalancer-controller  ForwardingRule "k8s2-fr-xlnnyd6i-default-web-ingress-4vzrw7ji" created
  Normal  IPChanged  2m27s                  loadbalancer-controller  IP is now 34.98.96.51
  Normal  Sync       2m26s (x4 over 3m18s)  loadbalancer-controller  Scheduled for sync
```

Finally, the web server should be reachable through insecure HTTP requests like this one:
```bash
curl http://$DOMAIN_NAME
```

This is the expected outcome from previous command:
```bash
Hello, world!
Version: 1.0.0
Hostname: web-79d88c97d6-t8hj2
```

## 6 - Install cert-manager

To issue a self-signed certificate, first we'll have to install `cert-manager`in the created cluster.

We'll do it via Helm again. So we'll use the first section as reference.

## 7 - Create an Issuer for Let's Encrypt in Staging


## 8 - Re-configure the Ingress for SSL


## 9 - Create a production ready SSL certificate


## 10 - Clean up


# Uninstall cert-manager

First, it's required to uninstall the tool itself using:

```bash
helm --namespace cert-manager delete cert-manager
```
    

Then, delete the namespace:
    
```bash
kubectl delete namespace cert-manager
```

And finally, uninstall the CRD. Make sure to reference the deployed version to properly remove it:

```bash    
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
```