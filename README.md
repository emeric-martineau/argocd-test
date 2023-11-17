# ArgoCD & Kind

## Install King on WSL2

You must first update your `/etc/wsl.conf`:
```sh
[boot]
systemd=true
```

Restart WSL:
```sh
wsl --shutdown
```

Relaunch your WSL by start your distribution.

Now, install Docker by reading [Official Install Documentation](https://docs.docker.com/engine/install/).

Then [install Kind](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries):
```sh
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64

# Clear proxy configuration
http_proxy=
https_proxy=
kind create cluster --name dev-cluster --config kind/kind-config.yaml
```

Install `kubectl` by following [official documentation](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/).

By default, you sould not have access to Kubernetes Cluster from Windows by using `http://localhost`. To do that, you must install an Ingress:
```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

## Install ArgoCD on Kind

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Get `admin` user password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

You need install [yq](https://mikefarah.gitbook.io/yq/) tool:
```shell
curl -L --output /usr/local/bin/yq https://github.com/mikefarah/yq/releases/download/v4.35.2/yq_darwin_amd64
chmod a+x /usr/local/bin/yq
```

You need disable TLS and add a base href to ArgoCD:
```sh
kubectl --namespace argocd get configmap argocd-cmd-params-cm -o yaml | yq '.data["server.basehref"] = "/argocd" | .data["server.insecure"] = "true"' > argocd-cmd-params-cm.yaml && kubectl apply -f argocd-cmd-params-cm.yaml
POD_NAME="$(kubectl --namespace argocd get pods --selector="app.kubernetes.io/name=argocd-server" -o jsonpath='{.items[0].metadata.name}')"
kubectl --namespace argocd delete pod "${POD_NAME}"
```

Now, we create ingress:
```sh
kubectl apply -f kind/argocd-kind-ingress.yaml
```

Now, you can access to ArgoCD by using `http://localhost:8080/argocd/`.

## Install Conjur

```sh
export DATA_KEY="$(docker container run --rm cyberark/conjur:1.20.1-4405 data-key generate)"
cat applications/conjur/argocd.yaml | envsubst > conjur-argocd.yaml && kubectl apply -f conjur-argocd.yaml
```
