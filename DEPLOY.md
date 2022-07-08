# Doc for Deploying on SCW

## Requirements

**Important**: You need a Kubernetes Kapsule Cluster deployed with Traefik 2 to follow this doc.
To deploy your cluster with Traefik 2, use the easy deploy feature.

You'll also need to configure your cluster secrets to be able to connect to our container registry (ghcr) hosting our applications images :
```bash
$ kubectl create secret docker-registry ghcr-secret --docker-server=https://ghcr.io --docker-username=gh --docker-password=__TOKEN__ --docker-email=example@email.com
```

And you must configure secrets that are located in the file `secrets.yaml`:
```bash
$ kubectl apply -f secrets.yaml
```

**WARNING : DO NOT PUSH THIS FILE WITH SECRETS CONFIGURED ON THIS REPOSITORY** 

## Deploying a Load Balancer using the Easy Deploy feature
- Click the Easy Deploy tab on your clusters overview page. The Easy Deploy feature displays.
- Click Deploy an Application. The application deployment wizard displays.
- Select Application Library, type Traefik in the search bar and select the Traefik 2 Ingress application.
- Add an argument to enable TLS :
```bash
additionalArguments:
  - "--entryPoints.websecure.http.tls=true"
```
- Enter the name traefik for the application and type the kube-system namespace name.
- Click Deploy an application to deploy the Load Balancer on your cluster.

## Creating a service to deploy a LoadBalancer in front of Traefik 2

Use `kubectl` to deploy the configuration located in `lb.yaml`

```bash
$ kubectl create -f lb.yaml
service/traefik-ingress created
```

Verify that your LoadBalancer has been deployed correctly:

```bash
$ kubectl get svc -n kube-system
traefik-ingress   LoadBalancer   10.37.89.202    195.154.68.108   80:30509/TCP,443:32138/TCP   43s
```

You can see here that the IP address of your LoadBalancer is 195.154.68.108. If you ‘curl’ it you can reach the default backend (saying “404 page not found”) as no ingress objects are created and you are reaching it through the IP address:

```bash
$ curl 195.154.68.108
404 page not found
```

You must point your DNS api.domain.com and domain.com to this IP address.

## Deploying applications

From git, clone all the repositories associated to Good Food and for each project use `kubectl` to create the applications :

```bash
$ kubectl create -f .kube/deployment.yaml
```

## Deploying Cert Manager

Cert-manager is in charge of creating Let’s Encrypt TLS certificates to make your website secure.

- Modify the default Traefik 2 daemonset running on Kapsule to do that, add `--entrypoints.websecure.http.tls` in the cmd stanza.

```bash
$ kubectl edit ds traefik -n kube-system
daemonset.apps/traefik edited
[]
        - --global.checknewversion
        - --global.sendanonymoususage
        - --entryPoints.traefik.address=:9000
        - --entryPoints.web.address=:8000
        - --entryPoints.websecure.address=:8443
        - --entrypoints.websecure.http.tls
        - --api.dashboard=true
        - --ping=true
        - --providers.kubernetescrd
        - --providers.kubernetesingress
[]
```

- Delete the existing Traefik pods in order to get the new arguments.

```bash
$ kubectl -n kube-system delete pod -l app.kubernetes.io/name=traefik
```

- Use the command below to install cert-manager and its needed CRD (Custom Resource Definitions):

```bash
$ kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created
namespace/cert-manager created
serviceaccount/cert-manager-cainjector created
serviceaccount/cert-manager created
serviceaccount/cert-manager-webhook created
clusterrole.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrole.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
clusterrole.rbac.authorization.k8s.io/cert-manager-view created
clusterrole.rbac.authorization.k8s.io/cert-manager-edit created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-cainjector created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-issuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-clusterissuers created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-certificates created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-orders created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-challenges created
clusterrolebinding.rbac.authorization.k8s.io/cert-manager-controller-ingress-shim created
role.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
role.rbac.authorization.k8s.io/cert-manager:leaderelection created
role.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
rolebinding.rbac.authorization.k8s.io/cert-manager-cainjector:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager:leaderelection created
rolebinding.rbac.authorization.k8s.io/cert-manager-webhook:dynamic-serving created
service/cert-manager created
service/cert-manager-webhook created
deployment.apps/cert-manager-cainjector created
deployment.apps/cert-manager created
deployment.apps/cert-manager-webhook created
mutatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created
validatingwebhookconfiguration.admissionregistration.k8s.io/cert-manager-webhook created 
```

## Creating the Let's Encrypt issuer

- Create a cluster issuer that allow you to specify:

  - the Let’s Encrypt server, if you want to replace the production environment with the staging one.
  - the mail used by Let’s Encrypt to warn you about certificate expiration.

The file can be found in `certs/issuer.yaml` and you must use `kubectl` to apply the configuration :

```bash
$ kubectl create -f cluster-issuer.yaml
clusterissuer.cert-manager.io/letsencrypt-prod created
```

## Creating and using a Let's Encrypt certificate to serve apps in HTTPS

For each endpoint you'll need to create let's encrypt certificates :

There are two endpoints actually: one for the whole apis and one for the frontend.

They are stored in the `certs/` directory Apply them using `kubectl`:

```bash
$ kubectl create -f api-cert.yaml
certificate.cert-manager.io/api-cert created
```
```bash
$ kubectl create -f front-cert.yaml
certificate.cert-manager.io/api-cert created
```

## Deploy the ingress controller

Now that SSL is ON, deploy the ingress controllers using `kubectl`:

```bash
$ kubectl create -f ingresses.yaml
ingress.networking.k8s.io/front-ingress created
ingress.networking.k8s.io/api-ingress created
```