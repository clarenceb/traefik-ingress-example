Traefik Ingress
===============

Demo using Traefik ingress in AKS.

Assumptions:
- Path based routing for a single domain (shouldn't be too hard to extend this sample to handle multiple domains)
- Tested with Ubuntu (WSL2 on Windows 10) -- some adjustments to commands may be needed for other platforms

Also shows:
- TLS using Let's Encrypt certificates
- Using BasicAuth middleware to protect a service

Steps
-----

### Create AKS cluster and ACR (optional)

Follow these steps to create an AKS cluster: https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

**TL;DR steps:**

```sh
RESOURCE_GROUP=<myresourcegroup>
LOCATION=australiaeast
ACR_NAME=<myacrname>
CLUSTER_NAME=traefikdemo
AKS_VERSION=1.14.8

az group create -n ${RESOURCE_GROUP} -l ${LOCATION}
az acr create -g ${RESOURCE_GROUP} -n ${ACR_NAME} --sku Basic
ACR_ID=$(az acr show -g ${RESOURCE_GROUP} -n ${ACR_NAME} --query id -o tsv)

az aks create -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME} -c 3 -k ${AKS_VERSION} --vm-set-type VirtualMachineScaleSets --load-balancer-sku standard --attach-acr ${ACR_ID} -a monitoring --generate-ssh-keys

sudo az aks install-cli # if you don't have `kubectl` installed
az aks get-credentials -g ${RESOURCE_GROUP} -n ${CLUSTER_NAME}
```

### Install Traefik via Helm into the cluster

**Install Helm v3**

See: https://v3.helm.sh/docs/intro/install/

```sh
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
chmod 700 get_helm.sh
./get_helm.sh
```

Or to use Helm3 side-by-side with Helm v2:

```sh
# See https://github.com/helm/helm/releases for latest release
curl https://get.helm.sh/helm-v3.0.0-linux-amd64.tar.gz -o helm3.tgz
tar xvf helm3.tgz
sudo mv linux-amd64/helm /usr/local/bin/helm3  # assumes Linux or WSL on Windows
sudo chmod +x /usr/local/bin/helm3
```

Install Traefik Ingress chart:


```sh
helm3 repo add stable https://kubernetes-charts.storage.googleapis.com/
helm3 repo update
# First, update acme.email with a valid email address in traefik-values.yaml
helm3 install traefik-ingress stable/traefik -n kube-system --values traefik-values.yaml
# Wait for the external IP to be assigned by watching the traefik service (CTRL+C to exit)
kubectl get svc traefik -n kube-system -w
# Wait for the traefik pod to be 'Running' (CTRL+C to exit)
kubectl get pod -n kube-system -w
```

If you want to update any of the values for Traefik, edit the file `traefik-values.yaml` and run a helm upgrade:

```sh
helm3 upgrade traefik-ingress stable/traefik -n kube-system --values traefik-values.yaml
```

See the Traefik helm chart [configuration](https://github.com/helm/charts/tree/master/stable/traefik#configuration) for explanation of options.

### Set DNS name for the public IP of the Traefik controller:

Get the IP address of the traefik ingress controller service using kubectl get.

```sh
kubectl get svc traefik -n kube-system
# *** EXTERNAL IP  ***
# *** XXX.XX.XX.XX ***
```

Update the DNS name for the public IP of the Traefik ingress

```sh
# Public IP address
IP="<your_public_ip>"

# Name to associate with public IP address
DNSNAME=<unique-prefix-name> # e.g. traefik7242

# Get the resource-id of the public ip
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

# Update public ip address with dns name
az network public-ip update --ids $PUBLICIPID --dns-name $DNSNAME
```

### Deploy a sample app that uses Traefik ingress

Retrieve FQDN (`<DNSNAME>.<LOCATION>.cloudapp.azure.com`) mapped to the Ingress controller's public IP:

```sh
az network public-ip show --ids $PUBLICIPID --query dnsSettings.fqdn -o tsv
# DNSNAME.LOCATION.cloudapp.azure.com
```

Now deploy the sample app resources:

```sh
kubectl create ns azure-vote
# First, update the host in the Ingress resource of azure-vote-app.yaml to your Traefik public IP FQDN: <DNSNAME>.<LOCATION>.cloudapp.azure.com
kubectl apply -f azure-vote-app.yaml -n azure-vote
```

Wait until all resources have been created:

```sh
kubectl get all -n azure-vote
```

Browse to: https://DNSNAME.LOCATION.cloudapp.azure.com

The TLS certificate should be valid in your browser.azu

### Adding basic auth to the sample app

**Create the basic auth secret:**

```sh
sudo apt install apache2-utils  # Needed for htpasswd tool; otherwise install this another way
htpasswd -c auth myuser
kubectl create secret generic mysecret --from-file=auth -n azure-vote
```

**Redeploy the sample app using basic auth:**

Uncomment the following lines in the `Ingress` resource of `azure-vote-app.yaml` and apply the changes:

```yaml
  annotations:
    kubernetes.io/ingress.class: traefik
    # ingress.kubernetes.io/auth-type: basic
    # ingress.kubernetes.io/auth-secret: mysecret
```

should look like:

```yaml
  annotations:
    kubernetes.io/ingress.class: traefik
    ingress.kubernetes.io/auth-type: basic
    ingress.kubernetes.io/auth-secret: mysecret
```

```sh
kubectl apply -f azure-vote-app.yaml -n azure-vote
```

Reloading the sample app in the browser should now prompt you for a username and password.

### Troubleshooting

* Ensure you updated the placeholder values in any input files
* Ensure you use unique domain names
* Ensure your email for LEt's encrypt is valid
* For Traefik or Let's Encrypt issues, check the logs on your Traefik pod:

```sh
kubectl get pods -n kube-system
# traefik-847d6b54f9-x5btx
kubectl logs traefik-847d6b54f9-x5btx -n kube-system
```

### Resources

* https://github.com/helm/charts/tree/master/stable/traefik
* https://docs.microsoft.com/en-us/azure/dev-spaces/how-to/ingress-https-traefik
* https://letsencrypt.org/docs/challenge-types/
* https://docs.traefik.io/https/acme/#the-different-acme-challenges
* https://docs.traefik.io/middlewares/basicauth/
* https://kubernetes.io/docs/concepts/services-networking/ingress/
* https://docs.traefik.io/v1.5/configuration/backends/kubernetes/#annotations
* https://kubernetes.github.io/ingress-nginx/examples/auth/basic/



