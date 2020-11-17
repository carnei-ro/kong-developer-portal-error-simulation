# Kong Dev Portal error simulation

Required:

- Docker (with privileges to open ports 80 and 443 (setcap))
- KinD
- kubectl


## booting up

```bash
sudo setcap 'cap_net_bind_service=+ep' $(which docker)

kind create cluster --image=kindest/node:v1.16.15

sleep 90

kubectl apply -f ingress.yaml

for port in 80 443; do
  nodeport=$((30000 + port))
  docker run -d --name kind-ingress-${port} \
      --publish 0.0.0.0:${port}:${port} \
      --link kind-control-plane:target \
      alpine/socat -dd \
      tcp-listen:${port},fork,reuseaddr tcp-connect:target:${nodeport}
done

for port in 80 443; do
  nodeport=$((31000 + port))
  port_high=$((8000 + port))
  docker run -d --name kind-proxy-${port_high} \
      --publish 0.0.0.0:${port_high}:${port_high} \
      --link kind-control-plane:target \
      alpine/socat -dd \
      tcp-listen:${port_high},fork,reuseaddr tcp-connect:target:${nodeport}
done
```

Edit files `cpdp/07-kong-cp-deployment.yaml` and `cpdp/11-kong-dp-deployment.yaml`, fill the `value` for environment `KONG_LICENSE_DATA`

```bash
kubectl apply -f cpdp/
```

Open browser at the following links, then accept the certificate:

- https://keycloak-127-0-0-1.nip.io/
- https://kong-admin-127-0-0-1.nip.io/
- https://kong-admin-gui-127-0-0-1.nip.io/
- https://kong-portal-gui-127-0-0-1.nip.io/
- https://kong-portal-api-127-0-0-1.nip.io/

## Configure

- Go to KeyCloak at https://keycloak-127-0-0-1.nip.io/, login to `Administration Console` with credentials `admin/admin`.
- Configure / Clients / `Create` button:
    - Client ID: `kong-portal`
    - Client Protocol: openid-connect
    - SAVE
- At the `kong-portal`:
    - Access Type: `confidential`
    - Valid Redirect URIs:
        - `https://kong-portal-gui-127-0-0-1.nip.io/*`
        - `https://kong-portal-api-127-0-0-1.nip.io/*`
    - Web Origins:
        - `https://kong-portal-gui-127-0-0-1.nip.io`
        - `https://kong-portal-api-127-0-0-1.nip.io`
    - hit `Save` button
    - Scroll all to the top, hit `Credentials` tab
        - Copy the value of `Secret`
        - Go Back to file `cpdp/07-kong-cp-deployment.yaml`, paste the Secret to env: `KONG_PORTAL_AUTH_CONF` at `"client_secret"` (overwrite the empty string).
        - Re-apply the file `kubectl apply -f cpdp/07-kong-cp-deployment.yaml`
- Manage / Users / `Add user` button:
    - Username: `developer1`
    - Email: `dev1@dev.com`
    - Email Verified: `ON`
    - hit `Save` button
    - Go to `Credentials` tab:
        - Password: `Developer1`
        - Password Confirmation: `Developer1`
        - Temporary: `OFF`
        - hit `Set Password` button


- Login to Admin Portal at https://kong-admin-gui-127-0-0-1.nip.io/, username: `kong_admin`, password: `KongSuperAdminPassword`
- `Dev Portals` tab
- Enable dev portal

- Open the Developer Portal at `https://kong-portal-gui-127-0-0-1.nip.io/default`
- Click at `Sign Up` button at the top:
    - Full Name: `developer1`
    - Email: `dev1@dev.com`
    - Hit the `Create Account` button
    - Hit OK at the popup message

- Go to `Kong Admin Gui` again at `https://kong-admin-gui-127-0-0-1.nip.io/default/developers/requested` and `Approve` the developer1

- Go to `https://kong-portal-gui-127-0-0-1.nip.io/default` with other browser or annonymous mode:
    - `Login` button at the top:
        - Green `LOGIN` button at the center of the page
        - Login in the KeyCloak screen with email: `dev1@dev.com` and password: `Developer1`

## Cleaning up

```bash
kubectl delete -f cpdp/

docker rm -vf kind-proxy-8443 kind-proxy-8080 kind-ingress-443 kind-ingress-80

kind delete cluster
```
