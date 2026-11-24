# CKS Reference: Paths, API Server Flags, Kubelet Config

Collected from the hands-on CKS sets in `examples/CKS` (Kubernetes 1.34, kubeadm, containerd).

---

## 1. Paths

### Control plane (kubeadm)

| Path | What it is |
|---|---|
| `/etc/kubernetes/manifests/` | Static Pod manifests (kubelet `staticPodPath`). Edit a file here and the kubelet recreates the Pod. |
| `/etc/kubernetes/manifests/kube-apiserver.yaml` | API server static Pod: where you add or change all API server flags, volumes and volumeMounts. |
| `/etc/kubernetes/manifests/etcd.yaml` | etcd static Pod (data dir, cert flags). |
| `/etc/kubernetes/manifests/kube-controller-manager.yaml`, `kube-scheduler.yaml` | Other control-plane static Pods (`--profiling`, `--bind-address`). |
| `/etc/kubernetes/admin.conf` | Admin kubeconfig. Since 1.29 it uses the group `kubeadm:cluster-admins` (a normal RBAC binding). Copy it to `~/.kube/config` after a cert renewal. |
| `/etc/kubernetes/super-admin.conf` | Break-glass kubeconfig in `system:masters` (bypasses RBAC). |
| `/etc/kubernetes/*.conf` | kubeconfigs (`admin`, `controller-manager`, `scheduler`, `kubelet`) with embedded client certs. |
| `/etc/kubernetes/tmp` | kubeadm temp/backup directory used during upgrades. |
| `/var/lib/etcd` | etcd data directory (should be 700, `etcd:etcd`). |
| `/opt/backup/etcd-pre-upgrade.db` | Example `etcdctl snapshot save` target before an upgrade. |

### PKI

| Path | What it is |
|---|---|
| `/etc/kubernetes/pki/` | Cluster PKI directory, mounted into control-plane Pods as a hostPath. |
| `/etc/kubernetes/pki/ca.crt`, `ca.key` | Cluster CA. Signs client certs (users, kubelets). It is the `--client-ca-file`. |
| `/etc/kubernetes/pki/apiserver.crt`, `apiserver.key` | API server serving cert/key (`--tls-cert-file`, `--tls-private-key-file`). Check its SANs with `openssl x509 -text`. |
| `/etc/kubernetes/pki/apiserver-kubelet-client.crt/.key` | Client cert the API server uses to call kubelets. |
| `/etc/kubernetes/pki/apiserver-etcd-client.crt/.key` | Client cert the API server uses to call etcd. |
| `/etc/kubernetes/pki/front-proxy-ca.crt`, `front-proxy-client.crt/.key` | Aggregation layer (requestheader) CA and client. |
| `/etc/kubernetes/pki/sa.key`, `sa.pub` | ServiceAccount token signing key pair. |
| `/etc/kubernetes/pki/etcd/` | etcd CA and certs: `ca.crt`, `server.crt/.key`, `peer.*`, `healthcheck-client.*`. Use these with `etcdctl`. |
| `/var/lib/kubelet/pki/` | Kubelet certs: `kubelet-client-current.pem` (symlink to the rotated client cert), `kubelet.crt/.key` (serving). |

### API server add-on config files (your own, mounted into the API server Pod)

| Path | What it is |
|---|---|
| `/etc/kubernetes/audit-policy.yaml` | Audit `Policy` (`--audit-policy-file`). Needs a hostPath volume and volumeMount. |
| `/var/log/kubernetes/audit/` | Audit log directory (hostPath mount, must be writable). |
| `/var/log/kubernetes/audit/audit.log` | Audit log file (`--audit-log-path`). |
| `/etc/kubernetes/enc/enc.yaml` | `EncryptionConfiguration` for Secrets at rest (`--encryption-provider-config`). |
| `/etc/kubernetes/admission/admission-config.yaml` | `AdmissionConfiguration` (`--admission-control-config-file`), e.g. for ImagePolicyWebhook. |
| `/etc/kubernetes/admission/kubeconfig.yaml` | kubeconfig the ImagePolicyWebhook uses to reach the backend (`kubeConfigFile`). |
| `/etc/kubernetes/admission/ca.crt`, `apiserver-client.crt/.key` | Webhook backend CA and the API server's client cert for it, referenced from that kubeconfig. |

### Kubelet and node

| Path | What it is |
|---|---|
| `/var/lib/kubelet/config.yaml` | `KubeletConfiguration` file. Edit it, then `systemctl restart kubelet`. |
| `/var/lib/kubelet/` | Kubelet root dir (`--root-dir`). |
| `/var/lib/kubelet/seccomp/` | Base for `localhostProfile`. `profiles/x.json` in a Pod resolves to `/var/lib/kubelet/seccomp/profiles/x.json`. |
| `/usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf` | kubeadm kubelet drop-in (flags, `--config`). Older distros use `/lib/systemd/...`. Run `systemctl daemon-reload` after editing. |
| `/usr/bin/kubelet` | Kubelet binary. Check it with `sha512sum` against the release checksum. |
| `/etc/containerd/config.toml` | containerd config: runtime handlers (e.g. `runsc` for gVisor) used by RuntimeClass. |
| `/run/containerd/containerd.sock` (= `/var/run/...`) | containerd CRI socket (`containerRuntimeEndpoint`, `crictl`). |
| `/var/run/docker.sock` | Docker socket. Mounting it into a Pod amounts to root on the node. |
| `/etc/cni/net.d` | CNI config. |
| `/var/log/pods/`, `/var/log/containers/` | Container logs on the node, e.g. `kube-system_kube-apiserver-*/kube-apiserver/0.log` when the API server won't start. |
| `/var/log/syslog`, `/var/log/kern.log`, `/var/log/auth.log` | Host logs (AppArmor denials, ssh logins). |
| `/etc/ssh/sshd_config` | SSH hardening (e.g. `PermitRootLogin no`). |
| `/etc/passwd`, `/etc/shadow`, `/etc/sudoers` | Users, password hashes, sudo rules (host hardening, Falco targets). |

### AppArmor

| Path | What it is |
|---|---|
| `/etc/apparmor.d/` | Profile files. Load one with `apparmor_parser -r /etc/apparmor.d/<profile>` on **each node** where the Pod can run. |
| `/sys/kernel/security/apparmor/profiles` | Loaded profiles and their modes (or use `aa-status`). |
| `/proc/1/attr/current` | Read inside a container: the AppArmor profile it actually runs under. |
| `/proc/1/status` | Read inside a container: `Seccomp:` (0 = unconfined, 2 = filter), `NoNewPrivs`, capabilities. |

### Falco

| Path | What it is |
|---|---|
| `/etc/falco/falco.yaml` | Main config: `rules_files`, `file_output`, `json_output`, etc. |
| `/etc/falco/falco_rules.yaml` | Default rules. Never edit this file. |
| `/etc/falco/falco_rules.local.yaml` | Local overrides and custom rules (loaded after the defaults). |
| `/etc/falco/rules.d/` | Drop-in rule files (e.g. `custom.yaml`). |
| `/var/log/falco/alerts.log` | Example `file_output` target. |
| `/usr/lib/systemd/system/falco-modern-bpf.service` | Falco service unit (modern eBPF driver). `journalctl -u falco-modern-bpf`. |

### Inside a Pod

| Path | What it is |
|---|---|
| `/var/run/secrets/kubernetes.io/serviceaccount/` | Mounted SA `token`, `ca.crt`, `namespace` (disable with `automountServiceAccountToken: false`). |

---

## 2. kube-apiserver flags

### Authentication / authorization

| Flag | Notes |
|---|---|
| `--anonymous-auth=false` | Rejects unauthenticated requests (CIS 1.2.1). |
| `--authorization-mode=Node,RBAC` | Never `AlwaysAllow`. `Node` limits kubelets to their own objects. |
| `--client-ca-file` | CA that validates client certs (`/etc/kubernetes/pki/ca.crt`). |
| `--token-auth-file` | Static token file. Must **not** be set (CIS 1.2.2). |
| `--enable-bootstrap-token-auth=true` | Allows bootstrap tokens for kubelet TLS bootstrap. |
| `--service-account-key-file` | Public key that verifies SA tokens (`sa.pub`). |
| `--service-account-signing-key-file` | Private key that signs SA tokens (`sa.key`). |
| `--service-account-issuer` | `iss` claim of SA tokens. |
| `--service-account-lookup=true` | Checks that the SA token still exists in etcd. |
| `--requestheader-client-ca-file`, `--requestheader-allowed-names`, `--requestheader-username-headers`, `--requestheader-group-headers`, `--requestheader-extra-headers-prefix` | Front-proxy / aggregation layer auth. |
| `--proxy-client-cert-file`, `--proxy-client-key-file` | Front-proxy client cert for aggregated APIs. |

### Admission

| Flag | Notes |
|---|---|
| `--enable-admission-plugins=NodeRestriction,...` | Add `ImagePolicyWebhook`, `AlwaysPullImages`, etc. Keep `NodeRestriction`. |
| `--admission-control-config-file` | `AdmissionConfiguration` file (plugin config such as ImagePolicyWebhook). |
| `--allow-privileged` | Whether privileged containers are allowed at all. |

### Audit

| Flag | Notes |
|---|---|
| `--audit-policy-file` | Audit Policy. Without it, nothing is logged. |
| `--audit-log-path` | Log file (`-` = stdout). |
| `--audit-log-maxage` | Days to keep old log files. |
| `--audit-log-maxbackup` | Number of rotated files to keep. |
| `--audit-log-maxsize` | MB before rotation. |

### Encryption at rest

| Flag | Notes |
|---|---|
| `--encryption-provider-config` | `EncryptionConfiguration` path. The first provider encrypts; the others can only decrypt. |
| `--encryption-provider-config-automatic-reload` | Reloads the config file without restarting the API server. |

### TLS, kubelet and etcd connections

| Flag | Notes |
|---|---|
| `--tls-cert-file`, `--tls-private-key-file` | API server serving cert/key. |
| `--kubelet-client-certificate`, `--kubelet-client-key` | Client cert for calls to kubelets. |
| `--kubelet-certificate-authority` | Verifies the kubelets' serving certs (pair it with kubelet `serverTLSBootstrap`). |
| `--kubelet-preferred-address-types` | Order of node addresses used to reach kubelets. |
| `--etcd-servers` | etcd endpoint(s), `https://127.0.0.1:2379`. |
| `--etcd-cafile`, `--etcd-certfile`, `--etcd-keyfile` | TLS to etcd (`pki/etcd/ca.crt`, `apiserver-etcd-client.*`). |

### Network / misc

| Flag | Notes |
|---|---|
| `--profiling=false` | Disables `/debug/pprof` (CIS 1.2.15; also controller-manager and scheduler). |
| `--secure-port` | HTTPS port (6443). |
| `--advertise-address` | IP advertised to cluster members. |
| `--bind-address` | Listen address. |
| `--service-cluster-ip-range` | Service CIDR. |

### Related flags on other components

| Flag | Notes |
|---|---|
| controller-manager `--use-service-account-credentials=true` | Each controller runs with its own SA. |
| controller-manager / scheduler `--bind-address=127.0.0.1`, `--profiling=false` | CIS hardening. |
| controller-manager `--cluster-signing-duration` | Lifetime of certs signed via the CSR API (default 1 year). |
| etcd `--client-cert-auth=true`, `--peer-client-cert-auth=true` | Require client certs. |
| etcd `--auto-tls=false`, `--peer-auto-tls=false` | No self-signed certs. |
| etcd `--trusted-ca-file`, `--data-dir` | Client CA, data directory. |
| kubelet `--config`, `--root-dir` | Config file path, root dir (seccomp profile base). |
| ingress-nginx `--default-ssl-certificate` | Cluster-wide default TLS Secret (not per Ingress). |

---

## 3. Kubelet configuration (`/var/lib/kubelet/config.yaml`)

`apiVersion: kubelet.config.k8s.io/v1beta1`, `kind: KubeletConfiguration`. After editing, run `systemctl restart kubelet`.

| Property | Notes |
|---|---|
| `authentication.anonymous.enabled: false` | No anonymous access to port 10250 (CIS 4.2.1). |
| `authentication.webhook.enabled: true` | Validates bearer tokens with a TokenReview against the API server. |
| `authentication.webhook.cacheTTL` | How long TokenReview results are cached. |
| `authentication.x509.clientCAFile` | CA for client certs (`/etc/kubernetes/pki/ca.crt`). |
| `authorization.mode: Webhook` | Authorizes requests with a SubjectAccessReview, never `AlwaysAllow` (CIS 4.2.2). |
| `authorization.webhook.cacheAuthorizedTTL` / `cacheUnauthorizedTTL` | Cache durations for SAR results. |
| `readOnlyPort: 0` | Disables the unauthenticated port 10255 (CIS 4.2.4). |
| `protectKernelDefaults: true` | Kubelet fails if kernel sysctls differ from its defaults instead of changing them. |
| `seccompDefault: true` | Applies `RuntimeDefault` seccomp to containers that set no profile. |
| `rotateCertificates: true` | Rotates the **client** cert. `kube-apiserver-client-kubelet` CSRs are auto-approved. |
| `serverTLSBootstrap: true` | Gets the **serving** cert via the CSR API (`kubelet-serving`, approved manually). Replaces the kubelet's self-signed cert. |
| `staticPodPath` | Static Pod directory (`/etc/kubernetes/manifests`). |
| `containerRuntimeEndpoint` | CRI socket (`unix:///var/run/containerd/containerd.sock`). |
| `cgroupDriver` | Must match the runtime (`systemd`). |
| `clusterDNS` / `clusterDomain` | DNS for Pods. Listed only to recognize them in the file. |
