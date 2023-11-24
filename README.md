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

## Conjur

See [./CONJUR.md](CONJUR.md)

## Add ArgoCD Vault Plugin with Conjur

See https://discuss.cyberarkcommons.org/t/argocd-vault-plug-in-supports-conjur/2046

First, we add two config map:
```sh
kubectl apply -f applications/conjur/argocd-plugin/k8s_cmp-plugin-configmap.yaml

(
  export AVP_CONJUR_API_KEY="$(kubectl --namespace conjur get secret conjur-conjur-data-key -o yaml | yq '.data.key' | base64 -d)"
  export AVP_CONJUR_SSL_CERT="$(kubectl --namespace conjur get secret conjur-conjur-ssl-ca-cert -o yaml | yq '.data["tls.crt"]' | base64 -d)"
  yq '.data.AVP_CONJUR_API_KEY = strenv(AVP_CONJUR_API_KEY) | .data.AVP_CONJUR_SSL_CERT = strenv(AVP_CONJUR_SSL_CERT)' applications/conjur/argocd-plugin/k8s_vault-plugin-cm.yaml > k8s_vault-plugin-cm.yaml
) && kubectl apply -f k8s_vault-plugin-cm.yaml
```

We need add some properties on `argocd-repo-server` deployment:
```sh
# Extract ArgoCD deployment
kubectl --namespace argocd get deployment argocd-repo-server -o yaml > argocd-repo-server-deployment.yaml

# Add volume mounts
yq --inplace '.spec.template.spec.containers[0].volumeMounts += {"mountPath": "/home/argocd/cmp-server/config/plugin.yaml", "name": "cmp-plugin", "subPath": "avp.yaml"}' argocd-repo-server-deployment.yaml
yq --inplace '.spec.template.spec.containers[0].volumeMounts += {"mountPath": "/usr/local/bin/argocd-vault-plugin", "name": "custom-tools", "subPath": "argocd-vault-plugin"}' argocd-repo-server-deployment.yaml

# Add volume
yq --inplace '.spec.template.spec.volumes += {"name": "cmp-plugin", "configMap": {"name": "cmp-plugin", "defaultMode": 420}}' argocd-repo-server-deployment.yaml
yq --inplace '.spec.template.spec.volumes += {"name": "custom-tools", "emptyDir": {}}' argocd-repo-server-deployment.yaml

# Add env var from config map
yq --inplace '.spec.template.spec.containers[0] += {"envFrom": [{"configMapRef": {"name": "vault-plugin-cm"}}]}' argocd-repo-server-deployment.yaml

# Add volume in init container
yq --inplace '.spec.template.spec.initContainers += {"args": ["cp /usr/local/bin/argocd-vault-plugin /custom-tools/"], "command": ["sh", "-c"], "image": "itdistrict/argocd-vault-plugin:1.2", "name": "install-argocd-vault-plugin", "volumeMounts":[{"mountPath": "/custom-tools", "name": "custom-tools"}]}' argocd-repo-server-deployment.yaml
```

## External secrets Kubernetes Operator

See [./EXTERNAL_SECRETS_CONJUR.md](EXTERNAL_SECRETS_CONJUR.md)
