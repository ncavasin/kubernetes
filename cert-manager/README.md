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

## Uninstall cert-manager

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