# Multi-Tenant Workloads

Den här mappen (`tenant/`) är avsedd för tenant-specifika applikationer och resurser som kan isoleras med hjälp av Kubernetes namespaces, RBAC och Flux multi-tenancy.

## Rekommenderad struktur

För varje tenant kan du skapa en separat mapp:

```
tenant/
  tenant-a/
    namespaces.yaml      # Definerar namespaces för tenant-a
    rbac.yaml            # ServiceAccount, Role, RoleBinding (för att begränsa tillgång)
    resources/
      kustomization.yaml
      deployment.yaml
      service.yaml
  tenant-b/
    ...
```

I din klusterkonfiguration (t.ex. `clusters/tenant/`) kan du sedan konfigurera Flux Kustomization för respektive tenant att endast läsa från dess egen mapp.
