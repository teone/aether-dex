# DEX and LDAP integration

To get started you need a `kubernetes` cluster (so far the tests have been performed using `kind`).

The `kind` cluster can be created with:
```
kind create cluster --config aether-cluster.cfg --name aether
```

Add a couple repos to `helm`, if you don't already have them:
```
helm repo add stable https://charts.helm.sh/stable
helm repo add cetic https://cetic.github.io/helm-charts
helm repo update
```

## Install LDAP server

Install the LDAP Server
```
helm install ldap stable/openldap --set env.LDAP_ORGANISATION='ONF' \
  --set env.LDAP_DOMAIN=opennetworking.org \
  --set tls.enabled=false \
  --set persistence.enabled=false \
  --set adminPassword=password,configPassword=password \
  --set logLevel=debug
```

Test LDAP authentication:
```
kubectl port-forward services/ldap-openldap 3389:389
ldapsearch -x -H ldap://localhost:3389 -b dc=opennetworking,dc=org -D "cn=admin,dc=opennetworking,dc=org" -w  password
```

## LDAP Admin

It seems useful to have a GUI to look into LDAP:
```
helm upgrade --install phpldapadmin cetic/phpldapadmin -f ldap-php-admin-values.yaml
```
You can expose the GUI with
```
export POD_NAME=$(kubectl get pods --namespace default -l "app=phpldapadmin,release=phpldapadmin" -o jsonpath="{.items[0].metadata.name}")
echo "Visit http://127.0.0.1:8080 to use your application"
kubectl port-forward $POD_NAME 8080:80
```

Click on `Login` on the left side of the GUI and use the password `password`.
The user is authenticating against LDAP.

## DEX

Install the `dex` server:
```
helm install dex stable/dex -f dex-values.yaml
```

Expose the dex server on port `32000`.
_Note that you need to add `127.0.0.1 dex` to your `/etc/hosts` file so that apps can use it_

```
DEX_POD_NAME=$(kubectl get pods -l "app.kubernetes.io/name=dex,app.kubernetes.io/instance=dex" -o jsonpath="{.items[0].metadata.name}") &&
kubectl port-forward $DEX_POD_NAME 32000:5556
```

Get the DEX client app to run a test:

```
git clone https://github.com/dexidp/dex.git
make bin/example-app
./bin/example-app --issuer http://dex:32000
```
