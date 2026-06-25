# Tenant Cluster Entrypoint

Den här mappen (`clusters/tenant/`) representerar kluster-konfigurationen för din tenant-miljö.

## Bootstrap instruktioner

För att bootstrapa det här klustret med FluxCD:

```bash
flux bootstrap github \
  --owner=<github-username> \
  --repository=k8s-infrastructure \
  --branch=main \
  --path=./clusters/tenant \
  --personal
```

Detta kommer att skapa mappen `flux-system` i den här mappen och börja synkronisera de resurser som definieras här.
