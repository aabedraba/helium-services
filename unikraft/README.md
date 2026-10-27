# Running on Unikraft Cloud

The main stack runs as three instances in the `fra` metro. Each instance uses scale-to-zero.

| Instance | Image source | Exposure | Scale-to-zero |
|---|---|---|---|
| `helium-nginx` | [`unikraft/nginx`](nginx) | Public, `helium-services.fra.unikraft.app` | `on`, 1 s cooldown |
| `helium-ext` | [`svc/extension-proxy`](../svc/extension-proxy) | Private, `helium-ext.internal:8000` | `on`, stateful, 5 s cooldown |
| `helium-ubo` | [`svc/ubo`](../svc/ubo) | Private, `helium-ubo.internal:8000` | `on`, stateful, 5 s cooldown |

Unikraft Cloud terminates TLS, so this setup does not use `acme.sh`. The nginx image is a copy of [`svc/nginx`](../svc/nginx) with these changes:

- It listens on plain HTTP port 8080.
- It sends requests to the `.internal` names of the other instances.
- It bakes `bangs.json` and the spellcheck dictionaries into the image at build time. It does not refresh them in the background. To get a newer `bangs.json`, rebuild the image and replace the instance.

## Build

`unikraft build` needs BuildKit. With OrbStack, point the CLI at its socket:

```sh
export DOCKER_HOST=unix://$HOME/.orbstack/run/docker.sock

unikraft build svc/extension-proxy --output abdallah_unikraft/helium-ext:latest
unikraft build svc/ubo --output abdallah_unikraft/helium-ubo:latest
unikraft build unikraft/nginx --output abdallah_unikraft/helium-nginx:latest
```

## Deploy

The values come from the gitignored `.env` file in the repo root. Start the private services first, because nginx resolves their names when it starts.

```sh
set -a; . ./.env; set +a

unikraft run --metro fra --name helium-ext -m 512MiB --restart=on-failure \
  -e HMAC_SECRET="$HMAC_SECRET" -e PROXY_BASE_URL="$PROXY_BASE_URL" \
  --scale-to-zero policy=on,cooldown-time=5000,stateful=true \
  --image abdallah_unikraft/helium-ext:latest

unikraft run --metro fra --name helium-ubo -m 512MiB --restart=on-failure \
  -e UBO_PROXY_BASE_URL="$UBO_PROXY_BASE_URL" \
  --scale-to-zero policy=on,cooldown-time=5000,stateful=true \
  --image abdallah_unikraft/helium-ubo:latest

unikraft run --metro fra --name helium-nginx -m 256MiB --restart=on-failure \
  -p 443:8080/http+tls -p 80:443/http+redirect --domain helium-services \
  --scale-to-zero policy=on,cooldown-time=1000 \
  --image abdallah_unikraft/helium-nginx:latest
```

To point Helium at this deployment, use `https://helium-services.fra.unikraft.app` as the services host.
