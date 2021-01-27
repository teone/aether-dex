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
helm install ldap stable/openldap --set env.LDAP_ORGANISATION='OpenNetworking Foundation' \
  --set env.LDAP_DOMAIN=opennetworking.org \
  --set tls.enabled=false \
  --set persistence.enabled=false \
  --set adminPassword=password,configPassword=password \
  --set logLevel=debug
```

Add top level containers for `users` and `groups`
```
kubectl port-forward services/ldap-openldap 3389:389
ldapadd -H ldap://localhost:3389 -D "cn=admin,dc=opennetworking,dc=org" -w password -f usersAndGroupsContainers.ldif
```

Add a sample user and group
```
kubectl port-forward services/ldap-openldap 3389:389
ldapadd -H ldap://localhost:3389 -D "cn=admin,dc=opennetworking,dc=org" -w password -f usersAndGroupsContainers.ldif
```

Test LDAP authentication:
```
kubectl port-forward services/ldap-openldap 3389:389
ldapsearch -x -H ldap://localhost:3389 -b dc=opennetworking,dc=org -D "cn=admin,dc=opennetworking,dc=org" -w  password "(&(objectClass=inetOrgPerson)(mail=test@opennetworking.org))"
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

To follow the Dex logs
```
kubectl logs --follow $DEX_POD_NAME
```

Get the DEX client app to run a test:

```
git clone https://github.com/dexidp/dex.git && cd dex
make bin/example-app
./bin/example-app --issuer http://dex:32000
```

then browse to [http://localhost:5555/login](http://localhost:5555/login)
and login with "test@opennetworking.org" and password `password`

> The example-app must send "groups" in its "scope" to get groups in the Access Token
> This can be changed by adding it in `dexidp/dex/examples/example-app/main.go` Line 246

This should redirect you to [http://dex:32000](http://dex:32000) to login, and
when successful should redirect you to example-app where you should see
the logged in details.
```
ID Token:

eyJhbGciOiJSUzI1NiIsImtpZCI6ImM3YWFjMWRjNDJlNTkzMjFiMmJkZmQ1MmNlZWMxY2JmOGIxYTUwMGUifQ.eyJpc3MiOiJodHRwOi8vZGV4OjMyMDAwIiwic3ViIjoiQ2dWMGRYTmxjaElFYkdSaGNBIiwiYXVkIjoiZXhhbXBsZS1hcHAiLCJleHAiOjE2MTE3NTY4OTUsImlhdCI6MTYxMTY3MDQ5NSwiYXRfaGFzaCI6IkM5R0d4UkNxT2FaMl82YS1UTklkREEiLCJlbWFpbCI6InRlc3RAb3Blbm5ldHdvcmtpbmcub3JnIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImdyb3VwcyI6WyJ0ZXN0R3JvdXAiXSwibmFtZSI6IlRlc3QgVXNlciJ9.JmK449AlVftDzH2J4pnk08s4dZdgFQq6Iz5mvLltDKc11cNirVf-HrkmpPP4-G-5K2YZF-nEbcIT2LJky4mlsP8NlhYBot8AEyLekIHqeT77npvP82oChWFOSMDx1hBwb4JzVB131dcg8n9AwFULIUWfHwsson6LgrWIlai_Yh20LPeWpIzKB6vFEXiqSgz2JP-qtiqtX4zvaO4cgLbGyh2n0noPeVyYXMwYgr5-pvkCS3KeoEkr4g_7LoS4FVon7434gSlTkTtsFUmei3Wlf7S_oeKYiUVGCucb-KNL8B_W_D3pbsakhQHIfW2m6Xn7ogRvi58devuR2vJpGG9V8g
Access Token:

eyJhbGciOiJSUzI1NiIsImtpZCI6ImM3YWFjMWRjNDJlNTkzMjFiMmJkZmQ1MmNlZWMxY2JmOGIxYTUwMGUifQ.eyJpc3MiOiJodHRwOi8vZGV4OjMyMDAwIiwic3ViIjoiQ2dWMGRYTmxjaElFYkdSaGNBIiwiYXVkIjoiZXhhbXBsZS1hcHAiLCJleHAiOjE2MTE3NTY4OTUsImlhdCI6MTYxMTY3MDQ5NSwiYXRfaGFzaCI6ImdyeFg0TnY3T2I4UktyMlRhcjEtVHciLCJlbWFpbCI6InRlc3RAb3Blbm5ldHdvcmtpbmcub3JnIiwiZW1haWxfdmVyaWZpZWQiOnRydWUsImdyb3VwcyI6WyJ0ZXN0R3JvdXAiXSwibmFtZSI6IlRlc3QgVXNlciJ9.M-ikvqnX1i8KlElSXncPYHIoawBDOV9u0xU_-KfUNGN-ykiudhbsEpWb187gfLjlG_ToSJihUXZsTXESwyaXeK07tOhKLmaskmsnf_iSs8zJjAw78ZKCcRiMv6TRaf1J6u0VxeQ3bIP15hKHqEhkFAUuHKIsas9xf8vPCNxEU0S3AoCY6iuG1g7Oo-pvfF8txQ-SjUzAZTTT6YQihJB_06qc_VbaqSB4VG4iVGTmlx_qv-iWGN6Ekz12ellPC0lDcW4lTQ1CDPxFfWyWA6_q-LJlPH9Oh_jTHb-RPY2sIhcFuu9ChsaTmTbz9QcZzYDRNEh7fC9zxkSZHSCCLG_FRg
Claims:

{
  "iss": "http://dex:32000",
  "sub": "CgV0dXNlchIEbGRhcA",
  "aud": "example-app",
  "exp": 1611756895,
  "iat": 1611670495,
  "at_hash": "C9GGxRCqOaZ2_6a-TNIdDA",
  "email": "test@opennetworking.org",
  "email_verified": true,
  "groups": [
    "testGroup"
  ],
  "name": "Test User"
}
```
> Note: the `groups` entry shows the test user is part of `testGroup` as was configured in LDAP 
