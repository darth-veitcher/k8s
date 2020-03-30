Install cert-manager with the provided manifest based on version

```bash
export CERT_MANAGER_VERSION=v0.12.0; \
kubectl create -f https://github.com/jetstack/cert-manager/releases/download/${CERT_MANAGER_VERSION}/cert-manager.yaml
```