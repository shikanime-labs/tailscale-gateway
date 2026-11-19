<!-- owner: Automata | zone: technical | purpose: Canonical technical index for tailscale-gateway, the Tailscale Gateway Kubernetes controller -->
# Tailscale Gateway Controller

A Go controller (controller-runtime) that provisions Tailscale-based gateways.
It reconciles Gateway API resources, `Ingress`, and `Service` objects and drives
the Tailscale API to expose cluster workloads onto a tailnet.

## Resources it reconciles

- `Gateway` with class `tailscale` &mdash; the primary resource and owner of all
  generated workloads.
- `Ingress` with class `tailscale` &mdash; translated into a `Gateway` +
  `HTTPRoute` pair that the controller then manages.
- `Service` with `type: LoadBalancer` and `loadBalancerClass: tailscale` &mdash;
  exposed via the
  `tailscale.gateway.shikanime.studio/hostname` annotation; a finalizer cleans
  up the synthetic Gateway API resources and Tailscale device on deletion.

Generated resources carry the
`tailscale.gateway.shikanime.studio/source` annotation and a Gateway owner
reference so deletion cascades from the Gateway side.

## Package layout

- `cmd/tailscale-gateway-controller` &mdash; entry point; wires reconcilers,
  metrics on `:8080`, health probes on `:8081`.
- `internal/controller` &mdash; one reconciler per resource kind (`gateway`,
  `ingress`, `service`) plus constants for the managed classes.
- `internal/tailscale` &mdash; wrapper around
  `tailscale.com/client/tailscale/v2` with `CreateAuthKey` and `DeleteDevice`.
- `internal/applyconfig` &mdash; server-side-apply builders for deployments,
  RBAC, service accounts, and Tailscale pods.
- `internal/config` &mdash; viper-backed environment config; `internal/apiutil`
  &mdash; Secret/metadata helpers; `internal/reconcilerutil` &mdash; shared
  plumbing.

## Auth key flow

1. The Gateway reconciler looks for a Secret named `<gateway-name>` in the
   Gateway namespace.
2. If it contains `authkey`, it is left untouched.
3. Otherwise the controller mints a non-reusable, ephemeral, preauthorized,
   tagged key via OAuth credentials (`all:write`); tags come from
   `TAILSCALE_TAGS` (default `tag:gateway`).

## Installation

Prerequisites: a cluster allowing cluster-scoped resources, Gateway API CRDs
(`gateway.networking.k8s.io/v1`, standard install v1.4.0), and Tailscale OAuth
credentials with key-creation scopes. Create the
`tailscale-gateway-controller` Secret in `tailscale-system`, then apply the
gateway overlay:

```
kubectl apply -k https://github.com/shikanime-studio/tailscale-gateway/manifests/gateway
kubectl apply -k ./manifests/demo
```

## Environment configuration

| Variable | Purpose | Default |
| --- | --- | --- |
| `TAILSCALE_OAUTH_CLIENT_ID` | OAuth client ID | required |
| `TAILSCALE_OAUTH_CLIENT_SECRET` | OAuth client secret | required |
| `TAILSCALE_TAGS` | Comma-separated device tags | `tag:gateway` |
| `TAILSCALE_IMAGE` | Tailscale daemon image | `tailscale/tailscale:latest` |
| `METRICS_BIND_ADDRESS` | Metrics listener | `:8080` |
| `HEALTH_PROBE_BIND_ADDRESS` | Health probe listener | `:8081` |

## E2E tests

`go test -tags e2e ./e2e -count=1` runs against a live cluster with the CRDs,
the controller Secret, and the controller deployed into `tailscale-system`
(deploy locally with `skaffold run`). Tests generate unique resource names per
run.

## Development

CI lives in `.github/workflows/`: `nix.yaml` (flake check + package builds per
matrix system), `skaffold.yaml` (base config validation + profile renders),
plus `integration.yaml` and `release.yaml`. Renovate (`.github/renovate.json`)
enables `gomod` and `nix` managers with post-upgrade
`go mod tidy && go mod vendor && nix fmt`. Go is pinned to 1.27 to match
`go.mod` in both the devenv shell and the `buildGoModule` package.

## Repository layout

- `cmd/`, `internal/` &mdash; controller entry point and packages above
- `manifests/gateway/`, `manifests/demo/` &mdash; deployment and example overlays
- `e2e/` &mdash; e2e suite
- `skaffold.yaml`, `flake.nix` &mdash; local dev/build and Nix shell
- `vendor/` &mdash; vendored Go modules
- `docs/` &mdash; this canonical index

## Canonical sources

- [Repository README](https://github.com/shikanime-labs/tailscale-gateway#readme)
- [Technical wiki](https://github.com/shikanime-labs/tailscale-gateway/wiki)
  ([Architecture](https://github.com/shikanime-labs/tailscale-gateway/wiki/Architecture),
  [Operations](https://github.com/shikanime-labs/tailscale-gateway/wiki/Operations),
  [Development](https://github.com/shikanime-labs/tailscale-gateway/wiki/Development))

In-repo `docs/` is canonical for technical detail; the wiki is a rendered human
index that links here.
