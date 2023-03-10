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

So, we should buy and register a domain name in order to map it with the `web-ip` previously created.

Since this is a tutorial, that won't happen. We'll use the IP address raw.


## 5 - Create an Ingress

To expose the web-server to the internet, a Kubernetes Ingress object must be created. This will trigger the creation 
of several services in Google Cloud Platform to allow clients reaching it.

Initially, the Ingress will only handle HTTP requests, and then we'll add the SSL layer with cert-manager.

First, execute this command to declare an Ingress YAML manifest:
```bash
cat << EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: gce                    # Tell GCP to create an External LB to realize this Ingress
    kubernetes.io/ingress.allow-http: "true"            # Enable HTTP connections from Internet clients
    kubernetes.io/ingress.global-static-ip-name: web-ip # Tell GCP to associate the External LB with the static IP
spec:
 rules:
  - host: ${DOMAIN_NAME}  # ! Replace with the registered domain name
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
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

Or if we haven't bought a domain, we'll hit the static IP address:
```bash
IP_ADDRESS=$(gcloud compute addresses describe web-ip --format='value(address)' --global)
curl http://$IP_ADDRESS
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

In order to create a new SSL Certificate, cert-manager needs to be instructed on how to sign it. So, an `Issuer` custom
resource needs to be configured to connect with Let's Encrypt staging server, allowing us to test our setup without 
consuming our Let's Encrypt certificate quota for the domain name.

Generate the `Issuer` manifest, **change the email address env var** before executing it:
```bash
EMAIL_ADDRESS=your_email_here # ! Replace with your email

cat << EOF > issuer-lets-encrypt-staging.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: ${EMAIL_ADDRESS} # ❗ Replace this with your email address
    privateKeySecretRef:
      name: letsencrypt-staging
    solvers:
    - http01:
        ingress:
          name: web-ingress
EOF
```

Then apply it with kubectl:
```bash
kubectl apply -f issuer-lets-encrypt-staging.yaml
```

Verify `Issuer`'s state with:

```bash
kubectl describe issuers.cert-manager.io letsencrypt-staging
```

Expected output is:
```bash
Status:
  Acme:
    Last Registered Email:  firstname.lastname@example.com
    Uri:                    https://acme-staging-v02.api.letsencrypt.org/acme/acct/60706744
  Conditions:
    Last Transition Time:  2022-07-13T16:13:25Z
    Message:               The ACME account was registered with the ACME server
    Observed Generation:   1
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
```

## 8 - Re-configure the Ingress for SSL

It's time to use our new certificate and for that we need to reconfigure the Ingress to accept `HTTPS`.

First, we need to use a workaround for a chicken-and-egg issue where the `ingress-gce` controller won't update its 
forwarding rules unless it can first find the Secret that will eventually contain the SSL certificate. But Let's 
Encrypt won't sign the SSL certificate until it can get the special `.../.well-known/acme-challenge/...` URL which 
cert-manager adds to the Ingress and which must then be translated into Google Cloud forwarding rules, by the 
`ingress-gce` controller.

So, we need to first create an empty secret:
```bash
cat << EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-ssl
type: kubernetes.io/tls
stringData:
  tls.key: ""
  tls.crt: ""
EOF
```
Then apply secret's manifest:
```bash
kubectl apply -f secret.yaml
```

Done with the workaround, it's time to update Ingress' manifest definition to use the staging certificate.

First, define an environmental variable for the registered domain:
```bash
DOMAIN_NAME=your_domain_here
```

Then, update Ingress' definition with `cert-manager`'s annotation & the `tls` spec block:
```bash
cat << EOF > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    kubernetes.io/ingress.class: gce                    # Tell GCP to create an External LB to realize this Ingress
    kubernetes.io/ingress.allow-http: "true"            # Enable HTTP connections from Internet clients
    kubernetes.io/ingress.global-static-ip-name: web-ip # Tell GCP to associate the External LB with the static IP 
    cert-manager.io/issuer: letsencrypt-staging         # Use Let's Encrypt staging server
spec:
 rules:
  - host: ${DOMAIN_NAME}
    http:
      paths:
      - pathType: Prefix
        path: /
        backend:
          service:
            name: web
            port:
              number: 8080
  tls:
    - secretName: web-ssl
      hosts:
       - ${DOMAIN_NAME}  # ! Replace with the registered domain name
EOF
```
---
### Ingress annotations

#### cert-manager.io/issuer: letsencrypt-staging
References the name of an `Issuer` to acquire the certificate required for this Ingress. The `Issuer` must be in the 
same namespace as the Ingress resource.

### Secret linkage
The `tls` spec references the empty secret with the domain name we've just registered, linking the Ingress resource
with it.
```bash
spec:
  tls:
    - secretName: web-ssl
      hosts:
        - <DOMAIN_NAME>
```

---

After updating Ingress' definition, apply it:
```bash
kubectl apply -f ingress.yaml
```

This triggers some complex operations that may last up to ~5 minutes and could eventually fail. However, after some 
attempts they should all succeed because cert-manager and Google's `ingress-gce` will periodically re-reconcile.

When all the pieces are set, the web server should be accessible using HTTPS. Verify this with:
```bash
curl -v --insecure https://$DOMAIN_NAME
```

Or if the IP address was used:
```bash
IP_ADDRESS=$(gcloud compute addresses describe web-ip --format='value(address)' --global)
curl -v --insecure https://$IP_ADDRESS
```


## 9 - Create a production ready SSL certificate

Once everything is working fine in Let's Encrypt's staging server, switch to the production and get a trusted SSL 
certificate.

First, a Let's Encrypt production `Issuer` resource must be declared:
```bash
cat << EOF > issuer-lets-encrypt-production.yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${EMAIL_ADDRESS} # ❗ Replace this with your email address
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
    - http01:
        ingress:
          name: web-ingress
EOF
```

Then it must be applied:
```bash
kubectl apply -f issuer-lets-encrypt-production.yaml
```

Verify it's working with:
```bash
kubectl describe issuers.cert-manager.io letsencrypt-production
```

Finally, update Ingress' annotation to use the production server instead of staging:
```bash
kubectl annotate ingress web-ingress cert-manager.io/issuer=letsencrypt-production --overwrite
```

This command will:
- Trigger the generation of a new SSL certificate signed by Let's Encrypt production CA.
- Will store that new SSL certificate in `web-ssl` Secret. 

After ~10 minutes, when the GCP load balancer gets synced with Let's Encrypt cert, the web server will be accessible
via secure HTTPS.

Validate with:
```bash
curl -v https://$DOMAIN_NAME
```

Or if the IP address was used:
```bash
IP_ADDRESS=$(gcloud compute addresses describe web-ip --format='value(address)' --global)
curl -v https://$IP_ADDRESS
```

Either way, expected output is:
```bash
...
* Server certificate:
*  subject: CN=www.example.com
*  start date: Jul 14 09:44:29 2022 GMT
*  expire date: Oct 12 09:44:28 2022 GMT
*  subjectAltName: host "www.example.com" matched cert's "www.example.com"
*  issuer: C=US; O=Let's Encrypt; CN=R3
*  SSL certificate verify ok.
...
Hello, world!
Version: 1.0.0
Hostname: web-79d88c97d6-t8hj2
```

Remember it's also accessible via the browser, without any errors nor warnings, at: 
[https://$DOMAIN_NAME](https://$DOMAIN_NAME)


## 10 - Recap

### Part 1

Set up the environment:

Create cluster -> generate static IP address -> HTTP:8080 Ingress -> register a domain and create an A record with the 
static IP to solve DNS queries.

At this point we expose our web server via HTTP using a domain name with our static IP address.

---
### Part 2

Then, the SSL process begins:

install cert-manager -> create issuer manifest using Let's Encrypt staging server -> define an empty TLS secret -> 
update Ingress config with:
- cert-manager.io/issuer: letsencrypt-staging annotation.
- spec tls: referencing the created 
secret and defining our domain name.

At this point we expose our web server via insecure HTTPS since we're using LE staging server.

#### How does it work?
cert-manager verifies Ingress' endpoint, detects the secret definition and auto-generates a signed certificate for the specified domain using the
`Issuer` custom resource as issuer.  

---

### Part 3

Finally, the SSL process concludes:
create issuer manifest using Let's Encrypt production server (secret will be auto-updated) -> update Ingress config with:
- cert-manager.io/issuer: letsencrypt-production annotation.

At this point we expose our web server via secure HTTP since we're using LE production server. The web server can be 
accessed through the browser.

---

Here's a brief summary of all the operations performed by cert-manager & ingress-gce:

1. cert-manager connects to Let's Encrypt and sends an SSL certificate signing request.
2. Let's Encrypt responds with a "challenge", which is a unique token that cert-manager must make available at a well-known location on the target web site. This proves that you are an administrator of that web site and domain name.
3. cert-manager deploys a Pod containing a temporary web server that serves the Let's Encrypt challenge token.
4. cert-manager reconfigures the Ingress, adding a `rule` to route requests for from Let's Encrypt to that temporary web server.
5. Google Cloud ingress controller reconfigures the external HTTP load balancer with that new rule.
6. Let's Encrypt now connects and receives the expected challenge token and the signs the SSL certificate and returns it to cert-manager.
7. cert-manager stores the signed SSL certificate in the Kubernetes Secret called `web-ssl.
8. Google Cloud ingress controller uploads the signed certificate and associated private key to a Google Cloud certificate.
9. Google Cloud ingress controller reconfigures the external load balancer to serve the uploaded SSL certificate.

---

We: 

1. Created a kubernetes cluster.
2. Deployed a web server internally.
3. Created a static IP address.
4. Created a domain name and its A record with our static IP.
5. Deployed an Ingress and exposed our web server insecurely via HTTP:8080.
6. Installed cert-manager.
7. Created an


## 11 - Clean up


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