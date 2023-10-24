# Traefik for Kubernetes Cluster on Hetzner with cert-manager

## Prerequisites
1. Have kubernetes cluster
2. kubectl (v1.28.2 client and server)
3. Helm (v3.13.0, GoVersion:go1.21.1)
4. Install Hetzner Cloud Manager (follow this [here](https://github.com/Laurin-Notemann/kubernetes_init_master) if you haven't done this yet)
5. Have your own domain

## Install Traefik and cert-manager

1. Add Traefik with helm 
```bash
helm repo add traefik https://traefik.github.io/charts
```

2. Change actual-traefik.yaml (password and hostname for the dashboard) 
```yaml
ingressRoute:
  dashboard:
    enabled: true
    matchRule: Host(`dashboard.example.com`) <------ !!!
    entryPoints: ["websecure"]
    middlewares:
      - name: traefik-dashboard-auth

extraObjects:
  - apiVersion: v1
    kind: Secret
    metadata:
      name: traefik-dashboard-auth-secret
    type: kubernetes.io/basic-auth
    stringData:
      username: admin
      password: <change password> <------------ !!!
```

#### Consider changing othe values in the annotations as well, depending on your server setup (for example region and datacenter)

3. Install Traefik with values from actual-traefik.yml
```bash
helm install -f traefik-values.yaml traefik traefik/traefik --namespace traefik --create-namespace
```
If you have installed the hcloud manager correctly this should have created a new LoadBalancer. Now you need to set up A and AAAA Records with the IP-Address of your LoadBalancer that you can find in your hetzer cloud console.
The dashboard will be exposed on the domain you provide (not https but password protected)

4. Install cert-manager CustomResourceDefinitions
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.crds.yaml
```

5. Add Jetsack with helm
```bash
helm repo add jetstack https://charts.jetstack.io
```

6. Install cert-manager with helm
```bash
helm install cert-manager --namespace cert-manager --version v1.13.1 jetstack/cert-manager --create-namespace
```

## Example of certification
Whoami service example

### Staging

1. Create Whoami Namespace, Deployment and Service
```bash
kubectl apply -f whoami.yml
```

2. Change email in `staging-cert-manager.yml`
```yaml
email: your@email.com
```

3. Create staging cert-manager to test the issueing
```bash
kubectl apply -f staging-cert-manager.yml
```

4. Change issuer in `ingress-whoami.yml`
```bash
cert-manager.io/issuer: "letsencrypt-staging"
```

5. Add correct domain address in `ingress-whoami.yml`
```yaml
spec:
  tls:
  - hosts: 
    - your.example.com
    secretName: tls-whoami-ingress-http
  rules:
  - host: your.example.com
```

6. Start ingress for whoami
```bash
kubectl apply -f ingress-whoami.yml
```

7. Now wait for 30-60 secs and run this to check if the certificate is valid
```bash
kubectl get certificateS,challenge,order,pods,services,issuer,deployments,ingress,certificaterequests,secrets,configmaps -n whoami
```

8. If everything worked you can safely delete the ingress (this will delete all the belonging cert-manager actions as well)
```bash
kubectl delete ingress -n whoami whoami-ingress
```

#### IMPORTANT!! This will not issue a certificate that you can use in the browser, its only for testing purposes to check if the validation works procceed to the production example!!

### Prod
Skipping Steps here that where done during staging

2. Change email in `prod-cert-manager.yml`
```yaml
email: your@email.com
```

3. Create staging cert-manager to test the issueing
```bash
kubectl apply -f prod-cert-manager.yml
```

4. Change issuer in `ingress-whoami.yml`
```bash
cert-manager.io/issuer: "letsencrypt-prod"
```

6. Start ingress for whoami
```bash
kubectl apply -f ingress-whoami.yml
```

7. Now wait for 30-60 secs and run this to check if the certificate is valid
```bash
kubectl get certificateS,challenge,order,pods,services,issuer,deployments,ingress,certificaterequests,secrets,configmaps -n whoami
```

## Delete 

1. Delete cert-manager
```bash
helm delete cert-manager -n cert-manager
```

2. Delete cert-manager CRD
```bash
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.crds.yaml
```

3. Delete Traefik
```bash
helm delete traefik -n traefik
```

