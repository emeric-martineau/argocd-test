# External secret for k8s

## Install with ArgoCD

To install External secrets with ArgoCD, just do this:
```sh
kubectl --namespace argocd apply -f applications/external-secrets/argocd.yaml
```

## Get CA of Conjur

Get CA of conjur in base64 and put in `applications/external-secrets/secret-store.yaml` file.
```sh
POD_NAME="$(kubectl --namespace conjur get pods --selector="app=conjur-oss" -o jsonpath='{.items[0].metadata.name}')"
kubectl --namespace conjur exec -it "${POD_NAME}" -c conjur-nginx -- /bin/sh -c "cat /opt/conjur/etc/ssl/ca/tls.crt | base64 --wrap=0"
```
## Create SecretStore

You must create secret with conjur credentials:
```sh
# If hostid is host, must prefix by `host` see https://docs.conjur.org/Latest/en/Content/Developer/Conjur_API_Authenticate.htm#URIParameters
kubectl --namespace test1 create secret generic conjur-creds --from-literal=hostid=host/BotApp/myDemoApp --from-literal=apikey=xwqfgjwd9v5p2x9416827r6agq1an4f3n2j2sr4w29tw1691k67rf2
```

You must create a `SecretStore` in the namespace of application that need get secret:
```yaml
kubectl --namespace test1 apply -f applications/external-secrets/secret-store.yaml
```

Then  create an `ExternalSecret`: 
```yaml
kubectl --namespace test1 apply -f applications/external-secrets/external-secret.yaml
```

cannot setup new Conjur client: Can't append Conjur SSL cert 

## API

### ConjurProvider

See https://external-secrets.io/latest/api/spec/#external-secrets.io/v1beta1.ConjurProvider
```
+------------+------------+-------------+
| Field Name | Type       | Description |
+------------+------------+-------------+
| url        | string     |             |
| caBundle   | string     | (Optional)  |
| caProvider | CAProvider | (Optional)  |
| auth       | ConjurAuth |             |
+------------+------------+-------------+
```

### CAProvider

See https://external-secrets.io/latest/api/spec/#external-secrets.io/v1beta1.CAProvider
```
+------------+----------------+----------------------------------------------------------------------------------------------------------+
| Field Name | Type           | Description                                                                                              |
+------------+----------------+----------------------------------------------------------------------------------------------------------+
| type       | CAProviderType | The type of provider to use such as "Secret", or "ConfigMap".                                            |
| name       | string         | The name of the object located at the provider type.                                                     |
| key        | string         | The key where the CA certificate can be found in the Secret or ConfigMap.                                |
| namespace  | string         | (Optional) The namespace the Provider type is in. Can only be defined when used in a ClusterSecretStore. |
+------------+----------------+----------------------------------------------------------------------------------------------------------+
```

### CAProviderType (string alias)

See https://external-secrets.io/latest/api/spec/#external-secrets.io/v1beta1.CAProvider
```
+-------------+-------------+
| Value       | Description |
+-------------+-------------+
| "ConfigMap" |             |
| "Secret"    |             |
+-------------+-------------+
```

### ConjurAuth

See https://external-secrets.io/latest/api/spec/#external-secrets.io/v1beta1.ConjurAuth

```
+--------+--------------+-------------+
| Field  | Type         | Description |
+--------+--------------+-------------+
| apikey | ConjurApikey | (Optional)  |
| jwt    | ConjurJWT    | (Optional)  |
+--------+--------------+-------------+
```

### ConjurApikey

See https://external-secrets.io/latest/api/spec/#external-secrets.io/v1beta1.ConjurApikey
```
+-----------+--------------------------------------------+-------------+
| Field     | Type                                       | Description |
+-----------+--------------------------------------------+-------------+
| account   | string                                     |             |
| userRef   | External Secrets meta/v1.SecretKeySelector |             |
| apiKeyRef | External Secrets meta/v1.SecretKeySelector |             |
+-----------+--------------------------------------------+-------------+
```

```go
type SecretKeySelector struct {
	// The name of the Secret resource being referred to.
	Name string `json:"name,omitempty"`
	// Namespace of the resource being referred to. Ignored if referent is not cluster-scoped. cluster-scoped defaults
	// to the namespace of the referent.
	// +optional
	Namespace *string `json:"namespace,omitempty"`
	// The key of the entry in the Secret resource's `data` field to be used. Some instances of this field may be
	// defaulted, in others it may be required.
	// +optional
	Key string `json:"key,omitempty"`
}
```

### ConjurJWT

See https://external-secrets.io/latest/api/spec/#external-secrets.io/v1beta1.ConjurJWT
