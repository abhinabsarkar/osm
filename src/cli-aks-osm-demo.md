# Open Service Mesh - Demo
* [Demo explanation - Youtube](https://www.youtube.com/watch?v=MaOHyIDsyYE)
* [Demo - Github](https://github.com/MicrosoftDocs/azure-docs/blob/cf692573b67503441d36a6c2f0a5a649b7b46166/articles/aks/open-service-mesh-deploy-new-application.md)

Pre-requisite:
* Install Azure CLI
* Install the osm cli - https://osm-docs.netlify.app/docs/guides/cli/

## Create AKS cluster
```bash
rgName=rg-abhi-aks
location=canadacentral
# Create Azure Resource Group
az group create -n $rgName -l $location -o none
# Create AKS cluster
az aks create --resource-group $rgName --name osm-addon-demo \
    --node-count 2 --generate-ssh-keys \
    --enable-managed-identity \
    --verbose
# Connect to & verify AKS cluster
az aks get-credentials -g $rgName -n osm-addon-demo
# Get the current context config
kubectl config current-context
# List the AKS addons
az aks list -g $rgName | jq -r .[].addonProfiles
```
## Deploy the demo bookstore app
Refer - https://release-v1-1.docs.openservicemesh.io/docs/getting_started/install_apps/

![al txt](/images/bookstore-initial.png)
```bash
# Create namespaces
kubectl create namespace bookstore
kubectl create namespace bookbuyer
kubectl create namespace bookthief
kubectl create namespace bookwarehouse
# List all the namespaces
kubectl get ns

# Create the bookbuyer app
# bookbuyer is an HTTP client making requests to bookstore. This traffic is permitted.
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/release-v1.1/manifests/apps/bookbuyer.yaml
# List all resources created by the deployment in the namespace
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n bookbuyer

# Create the bookthief app
# bookthief is an HTTP client and much like bookbuyer also makes HTTP requests to bookstore. This traffic should be blocked.
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/release-v1.1/manifests/apps/bookthief.yaml
# List all resources created by the deployment in the namespace
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n bookthief

# bookstore is a server, which responds to HTTP requests. It is also a client making requests to the bookwarehouse service. This traffic is permitted
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/release-v1.1/manifests/apps/bookstore.yaml
# List all resources created by the deployment in the namespace
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n bookstore

# bookwarehouse is a server and should respond only to bookstore. Both bookbuyer and bookthief should be blocked.
kubectl apply -f https://raw.githubusercontent.com/openservicemesh/osm-docs/release-v1.1/manifests/apps/bookwarehouse.yaml
# List all resources created by the deployment in the namespace
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n bookwarehouse

# Validate bookbuyer is connecting to the bookstore service
POD="$(kubectl get pods -n bookbuyer --show-labels --selector app=bookbuyer --no-headers | grep -v 'Terminatoing' | awk '{print $1}' | head -n1)"
kubectl logs "${POD}" -n bookbuyer -c bookbuyer --tail=100 -f
```

## Configure OSM service mesh

![alt txt](/images/bookstore-osm-permissivemode.png)
```bash
# Enable Open Service Mesh addon
az aks enable-addons --addons "open-service-mesh" -g $rgName -n osm-addon-demo
# Validate the OSM addon
az aks list -g $rgName | jq -r .[].addonProfiles.openServiceMesh.enabled
# Validate the OSM AKS addon is running
kubectl get pods -n kube-system --selector app=osm-controller
kubectl get services -n kube-system
# Verify enablePermissiveTrafficPolicyMode is true i.e. all traffic is allowed
kubectl get meshconfig osm-mesh-config -n kube-system -o=jsonpath='{$.spec.traffic.enablePermissiveTrafficPolicyMode}'

# Add namespaces to the service mesh OSM
osm namespace add bookstore bookbuyer bookthief bookwarehouse

# Enable proxy sidecar injection for Bookstore Apps by restarting all the deployments in the namespaces
kubectl get pods -n bookbuyer # no sidecar (1/1)
kubectl rollout restart deployment bookbuyer -n bookbuyer
kubectl rollout restart deployment bookstore -n bookstore
kubectl rollout restart deployment bookwarehouse -n bookwarehouse
kubectl rollout restart deployment bookthief -n bookthief
kubectl get pods -n bookbuyer # Envoy service car is injected (2/2+)

# Validate sidecar injection
POD="$(kubectl get pods -n bookbuyer --show-labels --selector app=bookbuyer --no-headers | grep -v 'Terminatoing' | awk '{print $1}' | head -n1)"
kubectl describe pod $POD -n bookbuyer
# Validate bookbuyer is connecting to the bookstore service (in permissive mode)
kubectl logs "${POD}" -n bookbuyer -c bookbuyer --tail=100 -f
```

## Disable permissive traffic mode on the mesh
When permissive traffic mode is enabled, you do not need to define explicit [SMI (Service Mesh Interface)](https://smi-spec.io/) policies for services to communicate with other services in onboarded namespaces. For more information on permissive traffic mode in OSM, see [Permissive Traffic Policy Mode](https://docs.openservicemesh.io/docs/guides/traffic_management/permissive_mode/)

![alt txt](/images/bookstore-osm-permissivemode-false.png)

```bash
# Disable permissive mode for service mesh
kubectl patch meshconfig osm-mesh-config -n kube-system  --type=merge -p '{"spec":{"traffic":{"enablePermissiveTrafficPolicyMode":false}}}'
# Verify enablePermissiveTrafficPolicyMode is false i.e. all traffic is blocked
kubectl get meshconfig osm-mesh-config -n kube-system -o=jsonpath='{$.spec.traffic.enablePermissiveTrafficPolicyMode}'
# Validate bookbuyer is unable to connect to the bookstore service
kubectl logs "${POD}" -n bookbuyer -c bookbuyer --tail=100 -f
```

## Apply an SMI traffic access policy for buying books
![alt txt](/images/bookstore-smi-policy.png)
Create allow-bookbuyer-smi.yaml using the following YAML: [allow-bookbuyer-smi.yaml](/src/allow-bookbuyer-smi.yaml)
```bash
vi allow-bookbuyer-smi.yaml
# Deploy SMI traffic policy
kubectl apply -f allow-bookbuyer-smi.yaml
# Validate bookbuyer is able to connect to the bookstore service
kubectl logs "${POD}" -n bookbuyer -c bookbuyer --tail=100 -f
```

## Apply an SMI traffic split policy for buying books
![alt txt](/images/bookstore-smi-split-policy.png)
Traffic split policies allow you to configure the distribution of communications from one service to multiple services as a backend. This capability can help you test a new version of a backend service by sending a small portion of traffic to it while sending the rest of traffic to the current version of the backend service. This capability can also help progressively transition more traffic to the new version of a service and reduce traffic to the previous version over time.

Create bookbuyer-v2.yaml using the following YAML: [bookbuyer-v2.yaml](/src/bookbuyer-v2.yaml)
```bash
vi bookbuyer-v2.yaml
# Deploy bookstore-v2 service
kubectl apply -f bookbuyer-v2.yaml
```
Create bookbuyer-split-smi.yaml using the following YAML: [bookbuyer-split-smi.yaml](/src/bookbuyer-split-smi.yaml)  
It an SMI policy that splits traffic for the bookstore service. The original or v1 version of bookstore receives 25% of traffic and bookstore-v2 receives 75% of traffic.
```bash
vi bookbuyer-split-smi.yaml
# Deploy SMI traffic split policy
kubectl apply -f bookbuyer-split-smi.yaml
# Validate traffic split. 75% of traffic will be going to v2 version 
kubectl logs "${POD}" -n bookbuyer -c bookbuyer --tail=100 -f | grep 'Identity:'
```

## References
* [OSM demo - youtube](https://www.youtube.com/watch?v=MaOHyIDsyYE)
* [Manage a new application with Open Service Mesh (OSM) on Azure Kubernetes Service (AKS)](https://github.com/MicrosoftDocs/azure-docs/blob/cf692573b67503441d36a6c2f0a5a649b7b46166/articles/aks/open-service-mesh-deploy-new-application.md)
* [Open Service Mesh - Deploy Sample Applications](https://release-v1-1.docs.openservicemesh.io/docs/getting_started/install_apps/)