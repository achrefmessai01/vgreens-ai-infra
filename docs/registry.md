# Private LAN image registry

Model images are large (~7 GB PyTorch/CUDA). Pushing them to gitlab.com from the
on-prem runner and pulling them back to the GPU worker crosses the internet twice
over a slow uplink — that's the ~28-minute push you hit. This registry keeps images
on the LAN: the CI runner (on sv01) pushes to `localhost`, and the GPU worker pulls
from sv01 over gigabit. gitlab.com is no longer in the image path.

## Architecture

`registry:2` runs on **sv01** bound to `127.0.0.1:5000`, fronted by the existing
Caddy (internal CA) at `https://registry.vgreens.lan:9443`. htpasswd auth (bcrypt)
protects it. Both the runner and the worker trust Caddy's internal root CA via
`/etc/docker/certs.d/registry.vgreens.lan:9443/ca.crt`.

## One-time setup

1. **Add the registry password to the vault:**

   ```
   ansible-vault edit inventories/production/group_vars/vault.yml
   # add:  vault_registry_password: "<a strong password>"
   ```

2. **Deploy the registry + Caddy site + client trust:**

   ```
   ansible-playbook playbooks/registry.yml -K
   ```

   This runs `registry:2` on sv01, re-renders Caddy with the registry site (keeping
   the API/web sites), installs the CA on sv01 + gpu-worker-01, adds a
   `registry.vgreens.lan` hosts entry, and `docker login`s both hosts.

3. **Confirm the worker can reach it.** The one environment-specific value is
   `registry_ip_for_worker` in `playbooks/registry.yml` (default `10.8.0.30`). It
   must be the sv01 address the **GPU worker** can route to. If the worker reaches
   sv01 on its LAN IP instead of the VPN IP, set that. Verify on gpu-worker-01:

   ```
   docker pull registry.vgreens.lan:9443/vgreens-models/vgreens-model-strawberry-disease:latest || true
   ```

   A `not found` (rather than a TLS or connection error) means DNS + TLS + auth are
   all good and only the tag is missing — exactly what we want.

## Verify end to end

On sv01 (or your Mac, tunnelled), push a tiny test image:

```
docker pull hello-world
docker tag hello-world registry.vgreens.lan:9443/test/hello:1
docker login registry.vgreens.lan:9443 -u vgreens-ci   # password from the vault
docker push registry.vgreens.lan:9443/test/hello:1
```

If push + a pull from the worker both succeed, the registry is live.

## Point CI at the LAN registry

Keep gitlab.com as the default; flip to the LAN registry with **group-level CI/CD
variables** (Group → Settings → CI/CD → Variables) so it applies to every model:

- `LAN_REGISTRY` = `registry.vgreens.lan:9443`  (not masked — it's a hostname)
- `VGREENS_REGISTRY_USER` = `vgreens-ci`
- `VGREENS_REGISTRY_PASSWORD` = *(masked)* the vault password
- `VGREENS_REGISTRY_CA` = *(type: File)* the CA PEM. Grab it from a client:
  `cat "/etc/docker/certs.d/registry.vgreens.lan:9443/ca.crt"` — paste the whole
  `-----BEGIN/END CERTIFICATE-----` block.

**Runner DNS.** CI jobs run in a Docker container on sv01 with their own
`/etc/hosts`, so the runner must resolve `registry.vgreens.lan`. Add it to the
runner's docker executor in `/etc/gitlab-runner/config.toml` on sv01:

```toml
[runners.docker]
  extra_hosts = ["registry.vgreens.lan:172.16.20.110"]
```

then `sudo gitlab-runner restart`. (172.16.20.110 = sv01's LAN IP; the container
reaches Caddy there since the registry site binds all interfaces.)

Then, in `vgreens-model-template/ci/model-ci.yml`, prepend this to the `build`
job's `script` (it's a no-op unless `LAN_REGISTRY` is set, so the gitlab.com path
is unaffected):

```yaml
    - |
      if [ -n "$LAN_REGISTRY" ]; then
        export IMAGE="$LAN_REGISTRY/$CI_PROJECT_PATH:$CI_COMMIT_SHORT_SHA"
        export CACHE_REPO="$LAN_REGISTRY/$CI_PROJECT_PATH/cache"
        mkdir -p /kaniko/ssl/certs
        printf '%s\n' "$VGREENS_REGISTRY_CA" >> /kaniko/ssl/certs/additional-ca-cert-bundle.crt
        printf '{"auths":{"%s":{"auth":"%s"}}}' "$LAN_REGISTRY" \
          "$(printf '%s' "$VGREENS_REGISTRY_USER:$VGREENS_REGISTRY_PASSWORD" | base64 | tr -d '\n')" \
          > /kaniko/.docker/config.json
      fi
```

The worker already trusts the CA and is logged in (step 2), so it pulls the new
`registry.vgreens.lan:9443/...` image over the LAN. Submit training exactly as
before — the `docker_image` the CI publishes now points at the LAN registry.

## Maintenance

- **Disk:** images live in `/opt/vgreens-registry/data` on sv01. `REGISTRY_STORAGE_DELETE_ENABLED=true`
  is set, so you can delete tags via the registry API and run `registry garbage-collect`
  inside the container to reclaim space.
- **Backups:** the registry data dir is regenerable (images can be rebuilt from CI),
  so it doesn't need the jobs.db backup treatment — but you may snapshot it if you
  want fast recovery.
