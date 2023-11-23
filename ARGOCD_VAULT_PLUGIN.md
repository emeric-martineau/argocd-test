# ArgoCD Vault Plugin in k8s

## How it works?

ArgoCD Vault Plugin is binary that must be copied in ArgoCD container.

You setup ArgoCD to run plugin by adding k8s resources (see [offcial example](https://github.com/argoproj-labs/argocd-vault-plugin/blob/main/manifests/cmp-configmap/argocd-cm.yaml)):
```yaml
kind: ConfigMap
metadata:
  name: cmp-plugin
  namespace: argocd
```

ArgoCD parse file or run kustomize/helm then parse result.

In case of use of [Conjur ArgoCD Vault Plugin](https://github.com/itdistrict/argocd-vault-plugin) you must provide only one API key.

After, to use is in kustomize `<path:secrets:`:
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: test-secret
type: Opaque
stringData:
  password: <path:secrets#password>
  username: <path:secrets#username#1>
```

## Pro ArgoCD Vault Plugin with Conjur

- Easy to use in template
- Easy to configure

## Con ArgoCD Vault Plugin with Conjur

- Must patch ArgoCD resource to add vault plugin,
- Only one api key. That mean one user in Conjur must have access to all secrets,
- Must use unofficial fork of ArgoCD Vault Plugin
- If secret change you must redeploy application
