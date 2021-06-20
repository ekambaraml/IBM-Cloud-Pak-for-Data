### Setting up Secured Local Registry
Generate Certs
./certbot-auto certonly --manual --preferred-challenges dns --server https://acme-v02.api.letsencrypt.org/directory



Run the Docker Registry
```
export DOMAIN="ai.ibmcloudpack.com"
mkdir -p /etc/docker/registry/data
mkdir -p /etc/docker/registry/certs
podman run --privileged -d   --name registry   -p 5000:5000 -v /etc/docker/registry/data:/var/lib/registry -v /etc/docker/registry/certs/fullchain.pem:/certs/fullchain.pem -v /etc/docker/registry/certs/privkey.pem:/certs/privkey.pem -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain.pem -e REGISTRY_HTTP_TLS_KEY=/certs/privkey.pem registry:2

```
