# GCP Configuring Istio on GKE with Cloud Armor

## Prerequisites
A pre-existing GKE Cluster is required in order to run the commands in this README (see the [GCP_Creating_a_GKE_Instance_with_Terraform](https://github.com/rigoford/GCP_Creating_a_GKE_Instance_with_Terraform) repo).

## Usage
### Deploy
Set up a couple of environment variables:
```bash
export REGION="<REGION>"
export PROJECT_ID="<PROJECT_ID>"
export CLUSTER_NAME="<CLUSTER_NAME>"
```

Update firewall rules to open port 15017:
```bash
gcloud compute firewall-rules list \
  --filter="name~gke-cluster-[0-9a-z]*-master" \
  --project $PROJECT_ID

gcloud compute firewall-rules update <FIREWALL_RULE_NAME> \
  --allow tcp:10250,tcp:443,tcp:15017 \
  --project $PROJECT_ID

gcloud container clusters get-credentials $CLUSTER_NAME \
  --project $PROJECT_ID \
  --region $REGION
```

Grant cluster administrator (admin) permissions to the current user:
```bash
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-admin \
  --user=$(gcloud config get-value core/account)
```

Download Istio 1.11.4 locally:
```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.11.4  sh -
pushd istio-1.11.4
export PATH=$PWD/bin:$PATH
popd
```

Create a Cloud Armor Policy, denying all:
```bash
gcloud compute security-policies create my-cloud-armor-policy \
  --description "My Cloud Armor Security Policy" \
  --project $PROJECT_ID

gcloud compute security-policies rules update 2147483647 \
  --security-policy my-cloud-armor-policy \
  --action "deny-404" \
  --project $PROJECT_ID
```

Create the `istio-system` Namespace and define a BackendConfig:
```bash
kubectl apply -f namespace-and-backend-config.yaml
```

Install Istio onto the GKE Cluster, configuring the IstioOperator to use the BackendConfig:
```bash
istioctl install -y -f istio-operator-backend-config.yaml
```
This is the bit that tells Istio to use a Cloud Armor Policy with the Backend of any Load Balancer GCP creates on behalf of Istio.


Add the `istio-injection=enabled` label to the `default` Namespace and deploy the BookInfo application from Istio Samples: 
```bash
kubectl label namespace default istio-injection=enabled

kubectl apply -f istio-1.11.4/samples/bookinfo/platform/kube/bookinfo.yaml
```

Verify that the application is running:
```bash
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

```

Create a Gateway and VirtualService for the BookInfo application, and create an Ingress:
```bash
kubectl apply -f istio-1.11.4/samples/bookinfo/networking/bookinfo-gateway.yaml

kubectl apply -f istio-ingressgateway-ingress.yaml
```
It is the Ingress that uses the Cloud Armor Policy configured in the BackendConfig.

Try accessing the BookInfo application via the Ingress created:
```bash
curl -vvv http://<INGRESS_IP_ADDRESS>/productpage
```
You should receive a `404 Not Found` response, this is the Cloud Armor Policy blocking all requests.

Add a new Rule to the Cloud Armor Policy allowing requests from your IP Address:
```bash
gcloud compute security-policies rules update 1000000 \
  --security-policy my-cloud-armor-policy \
  --src-ip-ranges "$(dig +short myip.opendns.com @resolver1.opendns.com)/32" \
  --action "allow" \
  --project $PROJECT_ID
```
Wait a few minutes for the new rule to take effect and then try accessing the application again.

### Undeploy
```bash
kubectl delete -f istio-ingressgateway-ingress.yaml
kubectl delete -f istio-1.11.4/samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f istio-1.11.4/samples/bookinfo/platform/kube/bookinfo.yaml
istioctl x uninstall --purge
```

## Next steps
A second External (TCP) Load Balancer is still created by the Istio IngressGateway Service, which should obviously be removed.

## References
- https://istio.io/latest/docs/setup/install/istioctl/
- https://istio.io/latest/docs/setup/platform-setup/gke/
- https://istio.io/latest/docs/examples/bookinfo/
- https://alwaysupalwayson.com/posts/2021/04/cloud-armor/
