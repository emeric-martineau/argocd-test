# Conjur

## Install Conjur

```sh
(
  export DATA_KEY="$(docker container run --rm cyberark/conjur:1.20.1-4405 data-key generate)"
  envsubst < applications/conjur/argocd.yaml > conjur-argocd.yaml
) && kubectl apply -f conjur-argocd.yaml

# Change type of service
kubectl --namespace conjur get service conjur-conjur-oss -o yaml | yq '.spec.ports = .spec.ports + {"name": "http", "port": 80, "protocol": "TCP", "targetPort": "http"} | del(.spec.clusterIP) | del(.spec.clusterIPs) | del(.spec.externalTrafficPolicy) | del(.spec.internalTrafficPolicy) | del(.spec.ipFamilies) | del(.spec.ipFamilyPolicy) | .spec.type = "ClusterIP"' > conjur-conjur-oss-service.yaml
kubectl apply -f conjur-conjur-oss-service.yaml

# Create ingress
kubectl apply -f applications/conjur/conjur-oss-ingress.yaml

# Change Nginx configuration to remove http redirect to https
export CONJUR_SITE="$(cat applications/conjur/conjur_site.conf)"
kubectl --namespace conjur get configmap conjur-conjur-nginx-configmap -o yaml | yq '.data.conjur_site = strenv(CONJUR_SITE)'

# Kill pod to restart
POD_NAME="$(kubectl --namespace conjur get pods --selector="app.kubernetes.io/name: conjur" -o jsonpath='{.items[0].metadata.name}')"
kubectl --namespace conjur delete pod "${POD_NAME}"
```

Try to display hello page by using `http://localhost:8080/conjur/`.

Now, you must create admin account:
```sh
kubectl --namespace conjur exec --stdin --tty "${CONJUR_POD_NAME}" --container conjur-oss -- conjurctl account create myConjurAccount

Token-Signing Public Key: -----BEGIN PUBLIC KEY-----
...
-----END PUBLIC KEY-----
API key for admin: ...
```

Copy/Paste output in a file.

Install Conjur CLI under Windows. Download at https://github.com/cyberark/cyberark-conjur-cli/releases but be careful, version 7.1.0 doesn't work (see [issue 409](https://github.com/cyberark/cyberark-conjur-cli/issues/409) posted on May 20, 2022!)

## Create connection and login

Run a pod in interactive mode on same namespace and cluster that Conjur:
```sh
kubectl --namespace conjur run -it --restart='Never' --rm --image=cyberark/conjur-cli:8 --leave-stdin-open=true --command -- test sh
```

First create connection:
```sh
conjur init -u https://conjur-conjur-oss -a myConjurAccount --self-signed

Using self-signed certificates is not recommended and could lead to exposure of sensitive data.
 Continue? yes/no (Default: no): y

The Conjur server's certificate SHA-1 fingerprint is:

To verify this certificate, we recommend running the following command on the Conjur server:
openssl x509 -fingerprint -noout -in ~conjur/etc/ssl/conjur.pem
Trust this certificate? yes/no (Default: no): yes
Certificate written to ~\conjur-server.pem

Successfully initialized the Conjur CLI
To start using the Conjur CLI, log in to the Conjur server by running `conjur login`
```

Then login:
```sh
conjur login -i admin -p <admin_api>
```

## Create policy and add secret

Now, create policy for user and variable:
```sh
cat > BotApp.yml <<EOF
- !policy
  id: BotApp
  body:
    # Define a human user, a non-human identity that represents an application, and a secret
  - !user Dave
  - !host myDemoApp
  - !variable secretVar
  - !permit
    # Give permissions to the human user to update the secret and fetch the secret.
    role: !user Dave
    privileges: [read, update, execute]
    resource: !variable secretVar
  - !permit
    # Give permissions to the non-human identity to fetch the secret.
    role: !host myDemoApp
    privileges: [read, execute]
    resource: !variable secretVar
EOF

conjur policy load -b root -f BotApp.yml
```

Save output:
```json
{
  "created_roles": {
    "myConjurAccount:host:BotApp/myDemoApp": {
      "id": "myConjurAccount:host:BotApp/myDemoApp",
      "api_key": "..."
    },
    "myConjurAccount:user:Dave@BotApp": {
      "id": "myConjurAccount:user:Dave@BotApp",
      "api_key": "..."
    }
  },
  "version": 1
}
```

Set secret:
```sh
conjur variable set -i BotApp/secretVar -v 12345678
```
