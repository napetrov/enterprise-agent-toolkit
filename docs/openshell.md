# NVIDIA OpenShell (Policy-Enforced Sandboxes)

[NVIDIA OpenShell](https://github.com/NVIDIA/OpenShell) adds a policy layer on top of
[Agent Sandbox](agent-sandbox.md). Its gateway creates sandboxes as `agents.x-k8s.io`
Sandbox CRs, and a supervisor inside every sandbox enforces:

- **Egress default-deny** with per-host / per-binary allow rules and L7 (HTTP method/path) policy
- **Filesystem restrictions** (Landlock) and seccomp
- **Credential isolation** — the sandbox only sees a placeholder; the supervisor injects the
  real credential into requests toward the allowed endpoint

The toolkit registers the **GenAI Gateway** (LiteLLM) as an OpenShell provider, so agents in
OpenShell sandboxes call models through LiteLLM with a dedicated, budgeted virtual key that
never enters the sandbox.

OpenShell is **optional** and **off by default** (`deploy_openshell=off`).

---

## Supported Configuration

| Component | Version |
|---|---|
| OpenShell Helm chart / images | `0.0.116` (`oci://ghcr.io/nvidia/openshell/helm-chart`) |
| OpenShell CLI (used by the deployment) | `v0.0.116` GitHub release |
| Agent Sandbox | `v0.5.0` (`deploy_agent_sandbox=on`) |
| GenAI Gateway | LiteLLM as deployed by `deploy_genai_gateway=on` (optional; provider is skipped if absent) |
| User authentication | Single-admin mode (ClusterIP gateway, access gated by Kubernetes RBAC) |
| CNI | Must enforce `NetworkPolicy` (e.g. Calico) |

---

## Architecture

```
Admin / CI (control plane)
        │  openshell CLI over kubectl port-forward (mTLS client cert)
        ▼
openshell.openshell-system.svc.cluster.local:8080   (OpenShell gateway, StatefulSet)
        │  creates Sandbox CRs
        ▼
agent-sandbox-controller  (from deploy_agent_sandbox)
        │  spawns sandbox pod
        ▼
default--<name>  (pod in openshell-sandboxes)
  supervisor: egress policy, L7 rules, Landlock, credential injection
        │  Authorization: Bearer <placeholder>  →  real LiteLLM virtual key
        ▼
genai-gateway-service.genai-gateway.svc.cluster.local:4000  (LiteLLM)
```

---

## Deployed Components

| Component | Kind | Namespace | Notes |
|---|---|---|---|
| `openshell` | StatefulSet + Service (ClusterIP :8080, mTLS) | `openshell-system` | Upstream Helm chart with `core/helm-charts/openshell/values.yaml` |
| `openshell-client-tls` | Secret | `openshell-system`, `openshell-sandboxes` | Client mTLS bundle; copied into the sandbox namespace (supervisors mount it) |
| `openshell-genai-key` | Secret | `openshell-system` | LiteLLM virtual key minted inside the LiteLLM pod (master key never leaves it) |
| `eat-genai-gateway` | Provider profile | gateway | LiteLLM endpoint, bearer credential, allowed binaries |
| `genai-gateway` | Provider | gateway | Attach to sandboxes with `--provider genai-gateway` |
| `eat-openshell-sandbox-pods` | NetworkPolicy | `openshell-sandboxes` | Pod-level egress/ingress limits for sandbox pods (see below) |

---

## Configuration

### Enable in `core/inventory/agentic-config.cfg`

```ini
deploy_agent_sandbox=on
deploy_openshell=on
```

`deploy_openshell=on` requires `deploy_agent_sandbox=on` in the same run, or Agent Sandbox
CRDs already present on the cluster; otherwise the deployment exits before installing
anything. On re-runs, an existing `openshell` Helm release in `openshell-system` skips the
component (resume mode).

### Version pins in `core/inventory/metadata/agentic-metadata.cfg`

```ini
openshell_chart_version="0.0.116"
openshell_image_tag="0.0.116"
openshell_release_tag="v0.0.116"
```

### Settings in `core/inventory/metadata/vars/inference_openshell.yml`

| Variable | Default | Purpose |
|---|---|---|
| `openshell_release_name` | `openshell` | Helm release and gateway Service name |
| `openshell_namespace` | `openshell-system` | Gateway namespace (holds provider credentials) |
| `openshell_sandbox_namespace` | `openshell-sandboxes` | Sandbox pods |
| `openshell_allow_unauthenticated_clusterip_only` | `false` | Must be set to `true` to accept single-admin mode (see [Authentication](#authentication)) |
| `openshell_oidc_issuer` / `openshell_oidc_audience` | `""` / `openshell-cli` | OIDC user authentication (alternative to single-admin mode) |
| `openshell_genai_provider_enabled` | `true` | Register the GenAI Gateway provider |
| `openshell_genai_key_alias` | `openshell-sandboxes` | LiteLLM virtual key alias (spend tracking) |
| `openshell_genai_key_models` | `[]` | Models the key may use (empty = all) |
| `openshell_genai_key_max_budget` | `50` | LiteLLM budget for the key |
| `openshell_sandbox_network_policy` | `true` | Apply `eat-openshell-sandbox-pods` |
| `openshell_upstream_proxy_cidrs` | RFC 1918 ranges | Where the corporate proxy may live (proxy port only) |
| `openshell_sandbox_egress_cidrs` | `[]` | Extra private CIDRs that sandbox policies may target |

The corporate proxy (`https_proxy` / `no_proxy` from `agentic-config.cfg`) is passed to the
gateway, which chains sandbox egress through it.

### Authentication

OpenShell on Kubernetes authenticates users with OIDC only; mTLS covers transport and
supervisors. Choose one mode explicitly — the playbook fails otherwise:

- **Single-admin (supported)**: set `openshell_allow_unauthenticated_clusterip_only: true`.
  The gateway stays `ClusterIP`; only users who can `kubectl port-forward` in
  `openshell-system` and read `openshell-client-tls` can reach it.
- **OIDC**: set `openshell_oidc_issuer` (for example a Keycloak realm URL, served over HTTPS).
  Not yet validated with the toolkit.

---

## Deployment

```bash
# core/inventory/agentic-config.cfg
deploy_agent_sandbox=on
deploy_openshell=on

# core/inventory/metadata/vars/inference_openshell.yml
openshell_allow_unauthenticated_clusterip_only: true

./deploy-agentic-stack.sh
```

The Ansible playbook (`core/playbooks/deploy-openshell.yml`) runs 4 task groups:

| Task Group | Action |
|---|---|
| 1 | Prerequisites — cluster connectivity, explicit auth mode, Agent Sandbox `v1beta1` API served |
| 2 | Namespaces, Helm install, client TLS copy into the sandbox namespace, sandbox NetworkPolicy, gateway rollout |
| 3 | GenAI Gateway provider — mint virtual key, register profile and provider via a temporary port-forward (CLI credentials removed afterwards) |
| 4 | Summary |

Re-running is safe: the virtual key is minted once, the provider profile is updated in place
(running sandboxes pick up changes), and an existing provider is kept.

---

## Verification

```bash
kubectl get pods -n openshell-system
# NAME          READY   STATUS    RESTARTS
# openshell-0   1/1     Running   0

kubectl get secret openshell-genai-key -n openshell-system
kubectl get networkpolicy -n openshell-sandboxes
```

### Connect the CLI (admin, control plane)

Install the CLI from the [v0.0.116 release](https://github.com/NVIDIA/OpenShell/releases/tag/v0.0.116), then:

```bash
kubectl port-forward -n openshell-system svc/openshell 8080:8080 &
export NO_PROXY=127.0.0.1,localhost no_proxy=127.0.0.1,localhost

MTLS=${XDG_CONFIG_HOME:-$HOME/.config}/openshell/gateways/eat/mtls
mkdir -p "$MTLS" && chmod 700 "$MTLS"
for k in ca.crt tls.crt tls.key; do
  kubectl get secret openshell-client-tls -n openshell-system \
    -o jsonpath="{.data.${k//./\\.}}" | base64 -d > "$MTLS/$k"
done
chmod 600 "$MTLS"/*

openshell gateway add https://127.0.0.1:8080 --local --name eat
openshell provider list
# NAME           TYPE               CREDENTIAL_KEYS  CONFIG_KEYS
# genai-gateway  eat-genai-gateway  1                0
```

> The client certificate grants full control of the gateway. Keep it on the control plane.

### Run a sandbox that calls the GenAI Gateway

```bash
openshell sandbox create --name demo --provider genai-gateway --detach \
  --env OPENAI_BASE_URL=http://genai-gateway-service.genai-gateway.svc.cluster.local:4000/v1

openshell sandbox exec -n demo --no-tty -- sh -c 'echo $OPENAI_API_KEY'
# openshell:resolve:env:..._OPENAI_API_KEY    <- placeholder, not the key

openshell sandbox exec -n demo --no-tty -- sh -c \
  'curl -s $OPENAI_BASE_URL/models -H "Authorization: Bearer $OPENAI_API_KEY"'
# {"data":[{"id":"<model>", ...}]}            <- key injected by the supervisor

openshell sandbox exec -n demo --no-tty -- \
  curl -s -m 10 -o /dev/null -w '%{http_code}\n' https://example.com
# 000                                          <- denied: not allowed by any policy

openshell logs demo --source sandbox | grep DENIED
openshell sandbox delete demo
```

Without `--from`, sandboxes use the OpenShell community base image. Pass
`--from <image>` for your own agent image, and `--policy <file>` to allow additional
endpoints (see the [OpenShell policy reference](https://github.com/NVIDIA/OpenShell/blob/v0.0.116/docs/reference/policy-schema.mdx)).

---

## Sandbox NetworkPolicy

OpenShell `0.0.116` runs each sandbox as a single pod, which the chart's own NetworkPolicy
does not select. The toolkit applies `eat-openshell-sandbox-pods` to these pods as defense in
depth under the OpenShell egress policy:

- **Ingress**: only from `openshell-system`
- **Egress**: kube-dns, `openshell-system`, the GenAI Gateway on port `4000`, the proxy port on
  `openshell_upstream_proxy_cidrs`, `openshell_sandbox_egress_cidrs`, and public addresses
  (all private and link-local ranges excluded)

Pods labelled `openshell.ai/boundary-role` (split supervisor/workload pods of newer charts) are
not selected, so the chart's stricter policies stay in effect.

---

## Known Limitations (0.0.116)

- **Credential placeholder in a request body fails the request.** If an agent copies the
  placeholder into a request body (for example by pasting `env` output into its LLM context),
  the supervisor rejects the request with `credential_injection_failed`. Redact
  `openshell:resolve:env:*` from tool output before sending it to the model.
- **Sandbox pod logs show warnings only.** Policy decisions are written to a file inside the
  pod; read them with `openshell logs <sandbox>`. Fluent Bit does not collect them.
- **`kubectl exec` into a sandbox pod bypasses the sandbox policy.** Restrict `pods/exec` in
  `openshell-sandboxes` with Kubernetes RBAC.
- **One release name per cluster.** The chart creates a cluster-scoped ClusterRole named after
  the release (`openshell-node-reader` for release `openshell`). A second install with the same
  release name in another namespace fails with a Helm ownership error; set
  `openshell_release_name` to a different value.
- **NetworkPolicy enforcement depends on the CNI.** Some CNIs (for example kindnet) do not block
  traffic from a pod to its own node, such as the API server on a single-node cluster.

---

## Troubleshooting

```bash
kubectl get pods -n openshell-system
kubectl logs -n openshell-system statefulset/openshell
kubectl get sandbox,pods -n openshell-sandboxes
openshell logs <sandbox> --source sandbox
```

| Symptom | Cause / Fix |
|---|---|
| `deploy_openshell=on requires deploy_agent_sandbox=on` | Enable Agent Sandbox in the same run |
| `OpenShell on Kubernetes authenticates users via OIDC only` | Choose an auth mode in `inference_openshell.yml` |
| LLM call returns `policy_denied` | The calling binary is not in the provider profile `binaries` list (`core/helm-charts/openshell/genai-gateway-provider.yaml`); re-run the deployment after editing |
| CLI hangs or `transport error` | Port-forward dropped; restart it and make sure `NO_PROXY` includes `127.0.0.1` |

---

## Configuration Files

- **Playbook**: `core/playbooks/deploy-openshell.yml`
- **Wrapper**: `core/lib/components/openshell-controller.sh`
- **Variables**: `core/inventory/metadata/vars/inference_openshell.yml`
- **Helm overlay**: `core/helm-charts/openshell/values.yaml`
- **Provider profile**: `core/helm-charts/openshell/genai-gateway-provider.yaml`

---

## References

- [NVIDIA/OpenShell](https://github.com/NVIDIA/OpenShell)
- [OpenShell v0.0.116 release](https://github.com/NVIDIA/OpenShell/releases/tag/v0.0.116)
- [Agent Sandbox in this toolkit](agent-sandbox.md)
