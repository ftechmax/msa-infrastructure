# msa-infrastructure


## Deploy

```sh
kubectl apply -k cert-manager/crd
kubectl apply -k cert-manager/overlays/local

kustomize build istio/crd --enable-helm | kubectl create -f - && rm -rf istio/crd/charts
kubectl apply -k istio/overlays/local

kubectl apply -k registry/overlays/local
kubectl apply -k sealed-secrets/overlays/local
kubectl apply -k reloader/overlays/local

kubectl apply -k k3s/crd
kubectl apply -k k3s/overlays/local

kubectl apply -k ferretdb/overlays/local

kubectl apply -k rabbitmq/crd
kubectl apply -k rabbitmq/overlays/local

kubectl apply -k postgres/overlays/local
```

## Get kube.local CA
```sh
kubectl get secret -n cert-manager kube-local-root-ca -o jsonpath='{.data.ca\.crt}' | base64 -d > kube-local-root-ca.crt
```
