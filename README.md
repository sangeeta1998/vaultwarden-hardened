# vaultwarden-hardened

A hardened, non-root, distroless build of [Vaultwarden](https://github.com/dani-garcia/vaultwarden)
(an unofficial Bitwarden-compatible server) built from pinned upstream
source and shipped through a CI/CD pipeline that scans, signs, and
attests the image before it's published.

This is **not a fork** and does not modify Vaultwarden's application
code. It adds a different Dockerfile on top of an unmodified, pinned
checkout of the upstream source (AGPL-3.0, see [Licensing](#licensing)
below).

## Why we arehere

The official `vaultwarden/server:*-alpine` image as of the version
checked in this repo (see `docker/Dockerfile.hardened` for the pinned
tag) has three properties that a security-conscious deployment would
want to avoid:

| | Official `vaultwarden/server` (Alpine) | This image |
|---|---|---|
| Runs as | root (no `USER` directive) | non-root, unprivileged (`nonroot:nonroot`, uid 65532) |
| Base OS packages at runtime | `apk`, `curl`, `busybox sh`, `tzdata` | none: [distroless](https://github.com/GoogleContainerTools/distroless) `static-debian12`, no shell, no package manager |
| Bundled DB drivers | mysql + postgresql + sqlite, always, regardless of use | sqlite only by default (configurable via `--build-arg DB=...`) |
| Binary linking | dynamic (musl + system OpenSSL/sqlite via Alpine packages) | fully static (musl target + `vendored_openssl` + bundled sqlite); no runtime library dependencies at all |
| Supply chain | unsigned | signed with [cosign](https://github.com/sigstore/cosign) (keyless, via GitHub OIDC), SBOM generated and attested |

None of this is a criticism of the upstream project.. Alpine + root is
a completely reasonable default for a self-hosted homelab tool. It's
just a gap: as of writing, Vaultwarden isn't in Docker's
[Hardened Images](https://hub.docker.com/hardened-images/catalog)
catalog, and no rootless/distroless variant is officially published.

## This repo

- `docker/Dockerfile.hardened`: the hardened multi-stage build. It is
  built with a **pinned checkout of upstream Vaultwarden source as the
  build context** not this repo (see below); this repo only supplies
  the build definition and pipeline.
- `.github/workflows/build-push.yml`: CI that, on every push to `main`
  or version tag:
  1. clones the pinned upstream source tag
  2. builds `linux/amd64` + `linux/arm64` via Buildx/QEMU
  3. pushes to Docker Hub
  4. scans the pushed image with [Trivy](https://github.com/aquasecurity/trivy)
  5. generates an SPDX SBOM with [Syft](https://github.com/anchore/syft)
  6. signs the image digest with cosign (keyless, no signing key to manage or leak)
  7. attests the SBOM to the image with cosign
- `docker-compose.example.yml`: a runnable example applying the
  runtime-side hardening (`read_only`, `cap_drop: ALL`, `no-new-privileges`)
  that pairs with this image.

## Building locally

```bash
git clone --branch 1.37.3 --depth 1 https://github.com/dani-garcia/vaultwarden.git vw-src
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -f docker/Dockerfile.hardened \
  -t <dockerhub_username>/vaultwarden-hardened:1.37.3 \
  --push \
  ./vw-src
```

To build a different upstream version, change the `--branch` and the
`VW_VERSION` build arg to match.

## Running

```bash
docker run -d \
  --name vaultwarden \
  --read-only \
  --cap-drop=ALL \
  --security-opt no-new-privileges \
  -p 8080:8080 \
  -v vw-data:/data \
  <your-dockerhub-username>/vaultwarden-hardened:1.37.3
```

Or via `docker-compose.example.yml`:

```bash
docker compose -f docker-compose.example.yml up -d
```

### Health checks

This image has no `HEALTHCHECK`: distroless has no shell or `curl` to
run one with. Use orchestrator's native HTTP probe against
`GET /alive` on port 8080 instead (see the example in
`docker-compose.example.yml`, or a Kubernetes `livenessProbe.httpGet`
pointed at the same path).

## Verifying the published image

Every image pushed by CI is signed keylessly. we can verify it with:

```bash
cosign verify \
  --certificate-identity-regexp "https://github.com/<your-github-username>/<this-repo>/.github/workflows/build-push.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  <dockerhub-username>/vaultwarden-hardened:1.37.3
```

The SBOM attestation can be inspected with:

```bash
cosign verify-attestation --type spdxjson \
  --certificate-identity-regexp "https://github.com/<your-github-username>/<this-repo>/.github/workflows/build-push.yml@.*" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  <your-dockerhub-username>/vaultwarden-hardened:1.37.3
```

## limitations and next steps

- The web-vault base image is pinned by tag and not by digest. Upstream's
  own Dockerfile recommends pinning by digest for immutability;we note the same recommendation but not enforcing it yet.
  Resolving a tag to its digest and updating the `WEB_VAULT_TAG`
  build arg accordingly is a natural follow-up.
- Only `linux/amd64` and `linux/arm64` are built. Upstream also
  supports `armv6`/`armv7`; those aren't included here to keep the
  build matrix focused.
- Trivy currently reports findings without failing the build
  (`exit-code: "0"`). Flip it to `"1"` once you've triaged a baseline.


Vaultwarden is licensed under [AGPL-3.0](https://github.com/dani-garcia/vaultwarden/blob/main/LICENSE.txt).
We dont verify Vaultwarden's source.. it only adds a build
definition and pipeline layered on top of an unmodified, publicly
available, pinned upstream checkout, so upstream's own public
repository satisfies AGPL's source-availability requirement. This
repo's own files (Dockerfile, workflow, compose file) are original
