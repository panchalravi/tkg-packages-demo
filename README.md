# Pre-requisites
- Access to tanzu workload cluster with kubeconfig file
- Tanzu CLI  
- kubectl CLI

# Install Cert-Manager (with self-signed Certificate)
```sh
#Generate your own CA certificate
openssl genrsa -des3 -out localRootCA.key 2048
openssl req -x509 -new -nodes -key localRootCA.key -reqexts v3_req -extensions v3_ca -sha256 -days 1825 -out localRootCA.pem

#Create unprotected key from password protected key as it is required to work with cert-manager
openssl rsa -in localRootCA.key -out unprotected-localRootCA.key

#Create TLS secret  
kubectl create secret tls my-ca-secret --key unprotected-localRootCA.key --cert localRootCA.pem -n cert-manager

#Install cert-manager package
tanzu package install cert-manager --package-name cert-manager.tanzu.vmware.com --version 1.1.0+vmware.1-tkg.2 --namespace my-packages --create-namespace
```

# Contour
- Prepare "contour-package-config.yaml" with following data. This configures Contour service to be exposed as LoadBalancer type and also to use cert-manafer to auto-generate TLS certificates for ingress resources based on domain names.
```yaml
---
envoy:
  service:
    type: LoadBalancer
certificates:
  useCertManager: true
```
- Install Contour package
```sh
tanzu package install contour -p contour.tanzu.vmware.com --version 1.17.1+vmware.1-tkg.1 --values-file contour-package-config.yaml --namespace my-packages
```

# External DNS (Microsoft AD, insecure)
- Prepare 'external-dns-data-values.yaml' with following data. This configures external-dns to auto-create DNS Type A records in the DNS server for ingress resources.Here, the provider is Bind server such as Microsoft AD.
```yaml
#external-dns-data-values.yaml
---
# Namespace in which to deploy ExternalDNS.
namespace: tanzu-system-service-discovery

# Deployment-related configuration.
deployment:
 args:
   - --provider=rfc2136
   - --source=service
   - --source=ingress
   - --source=contour-httpproxy # Provide this to enable Contour HTTPProxy support. Must have Contour installed or ExternalDNS will fail.
   - --rfc2136-host=192.168.110.10
   - --rfc2136-port=53
   - --rfc2136-zone=corp.local
   - --rfc2136-insecure
   - --rfc2136-tsig-axfr
   - --domain-filter=corp.local
 env: []
 securityContext: {}
 volumeMounts: []
 volumes: []
 ```
 - Install external-dns package
 ```sh
tanzu package install external-dns --package-name external-dns.tanzu.vmware.com --version 0.8.0+vmware.1-tkg.1 --values-file external-dns-data-values.yaml --namespace my-packages
```

# Test the setup with sample kuard app 
- This configuration should expose kuard service via Ingress resource. 
- TLS certificate should be auto-generated for the ingress host "kuard.corp.local" by cert-manager.
- DNS Type A record entry for the domain name "kuard.corp.local" should be auto-created by external-dns. The IP should be that of Envoy service LoadBalancer IP.
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  replicas: 2
  selector:
    matchLabels:
      app: kuard
  template:
    metadata:
      labels:
        app: kuard
    spec:
      containers:
      - image: gcr.io/kuar-demo/kuard-amd64:1
        name: kuard
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: kuard
  name: kuard
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: kuard
  sessionAffinity: None
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: kuard
  annotations:
    kubernetes.io/ingress.class: "contour"
    cert-manager.io/cluster-issuer: "ca-issuer"
spec:
  defaultBackend:
    service:
      name: kuard
      port:
        number: 80
  rules:
  - host: kuard.corp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: kuard
            port:
              number: 80
  tls:
  - hosts:
    - kuard.corp.local
    secretName: kuard-cert
```
