# Cert-manager & Certificates

## Pre-Requesites

```shell
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

## To install locally

To install, using Helm, the Cert-manager application, run the following command:

```shell
kubectl apply -f namespace.yaml
helm upgrade -install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true --set prometheus.enabled=false --set webhook.timeoutSeconds=4
```

Before being able to request certificates, you must create CA resources: Issuer or ClusterIssuer. They are used for signing CSRs (certificate requests).  In our case, we use a ClusterIssuer.

To do so, run the following commands:
```shell
kubectl apply -f local.clusterissuer.yaml
kubectl apply -f local.certificate.yaml
```

To test locally, you can run this command as example, while installing the NGinx web server as a test:

```shell
kubectl apply -f local.test-crt.yaml
curl -k https://docker.internal
```


## To install on sandbox

To install on sandbox, run the following commands:

```shell
kubectl apply -f namespace.yaml --cluster <gcp_sandbox_clustername>
helm upgrade -install cert-manager jetstack/cert-manager --namespace cert-manager --set installCRDs=true --kube-context $BOUCIO_CLSTR_FULLNAME
```

This repo carries no values files. The chart settings are inline in the Flux HelmRelease
(`clusters/base/infrastructure/fluxcd-cert-manager.yaml`), and the command above mirrors them for a
manual install.
Where gcp_sandbox_clustername is pointing towards kubeconfig context name for the kubernetes cluster identified as the sandbox cluster.

To install the cluster issuer and certificate, run the following commands:

```shell
kubectl apply -f sandbox.clusterissuer-prod.yaml --cluster $BOUCIO_CLSTR_FULLNAME
kubectl apply -f sandbox.certificate-prod.yaml --cluster $BOUCIO_CLSTR_FULLNAME
```


### For references: To create the locally, self-signed certificate

Note, the following is automatically created by Cert-Manager and although could be used, it is provided as a reference only.

First, install the "cfssl" tool.

```shell
brew install cfssl
```

Second, generate the local Certificate Authority.

```shell
cfssl gencert -initca local.ca-csr.json | cfssljson -bare ca
```

This command will generate the following files:
- ca.pem
- ca-key.pem
- ca.csr

To test and validate, run the following command:

```shell
openssl x509 -in ca.pem -text -noout
```

Third, generate the certicate by running the following command:

```shell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=local.ca-config.json -profile=boucio local.boucio-csr.json | cfssljson -bare boucio-localhost
```
To test and validate, run the following command:

```shell
openssl x509 -in boucio-localhost.pem -text -noout
```


## FluxCD GitOps Integration (Issuers and Certificates)

This repository is consumed by FluxCD to manage both ClusterIssuers and Certificates in a GitOps workflow. The Flux repo includes this repo as a Git submodule and reconciles the subpaths per environment.

- Repo structure for GitOps
  - `issuers/local/` — local issuer(s) (e.g., self‑signed) with a `kustomization.yaml` referencing `local.clusterissuer.yaml`.
  - `issuers/sandbox/` — sandbox issuer(s) (e.g., ACME staging/prod) with a `kustomization.yaml` referencing `sandbox.clusterissuer-*.yaml`.
  - `certificates/local/` — Certificates for the local ingress gateway (namespace `istio-system`) with its own `kustomization.yaml`.
  - `certificates/sandbox/` — Certificates for the sandbox ingress gateway (namespace `istio-system`) with its own `kustomization.yaml`.

- How Flux consumes this repo
  - The Flux Git repo adds this repository as a Git submodule and enables `recurseSubmodules: true` on its `GitRepository` source.
  - Flux defines Kustomizations per environment that point to the submodule paths:
    - Issuers (local): `./clusters/components/cert-manager/issuers/local`
    - Issuers (sandbox): `./clusters/components/cert-manager/issuers/sandbox`
    - Certificates (local): `./clusters/components/cert-manager/certificates/local`
    - Certificates (sandbox): `./clusters/components/cert-manager/certificates/sandbox`
  - Issuers Kustomizations set `dependsOn` the environment’s infra Kustomization (which installs cert‑manager via Helm and Istio) and use `wait: true`, `prune: true`, and a `timeout`.
  - Certificates Kustomizations set `dependsOn` both infra and issuers to ensure controllers and CRDs exist and issuers are ready before requesting certs.
  - Health checks typically include `HelmRelease/cert-manager` in namespace `cert-manager`. If you use ACME HTTP01 with `ingress.class: istio`, you may also add Istio controller health checks.

- Certificates conventions
  - Place Certificates in `istio-system` so the ingress gateway can mount TLS Secrets without cross‑namespace writes.
  - Use stable `spec.secretName` values and reference them from the Gateway’s `credentialName` (or SDS).
  - Local issuer example: `issuerRef.name: selfsigned-cluster-issuer`.
  - Sandbox issuers: `issuerRef.name: letsencrypt-staging` and `letsencrypt-prod`.
  - `spec.dnsNames` should list the exact hosts routed by your VirtualServices; wildcards can reduce churn.

- Reconcile order (example)
  - `config` → `infra` (cert‑manager + Istio) → `cert-issuers` → `cert-certificates` → `apps`.

- Local validation without touching the cluster
  - Render manifests:
    - `kustomize build issuers/local`
    - `kustomize build certificates/local`
  - Server‑side dry run (requires CRDs on the cluster):
    - `kubectl apply --server-side --dry-run=server -f <(kustomize build issuers/local)`
    - `kubectl apply --server-side --dry-run=server -f <(kustomize build certificates/local)`

- In‑cluster checks (after Flux applies)
  - Issuers: `kubectl get clusterissuer`
  - Certificates: `kubectl get certificate -n istio-system` (check `READY=True`)
  - ACME flow: `kubectl get challenges,orders -A`

- Operating model
  - Changes to issuer/certificate manifests are made in this repo. Promotion happens when the Flux repo updates its submodule pointer to the desired commit (or tracks a branch).
  - For HTTP01 using `ingress.class: istio`, ensure DNS points to the Istio ingress and port 80 is routed to the gateway.
  - If moving to DNS01, store provider credentials via SOPS or ExternalSecrets and reference them in the issuer manifests.


## References

Refer to:
- https://medium.com/flant-com/cert-manager-lets-encrypt-ssl-certs-for-kubernetes-7642e463bbce
- https://cert-manager.io/docs/configuration/acme/http01/
- https://cert-manager.io/docs/configuration/acme/
- https://letsencrypt.org/docs/challenge-types/#http-01-challenge
- https://cert-manager.io/docs/tutorials/acme/http-validation/
- https://cert-manager.io/docs/installation/kubernetes/#installing-with-helm
- https://github.com/jetstack/google-cas-issuer
- https://cert-manager.io/docs/faq/acme/
- https://cert-manager.io/docs/concepts/issuer/
- https://tech.paulcz.net/blog/creating-self-signed-certs-on-kubernetes/
- https://youtu.be/SiLlYU5Ai1Y
- https://istio.io/latest/docs/ops/integrations/certmanager/#istio-gateway
- https://github.com/istio/istio/issues/27643#issuecomment-1588249182
