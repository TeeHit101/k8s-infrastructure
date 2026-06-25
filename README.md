# Kubernetes GitOps Infrastructure for k3s Homelab

Detta GitOps-arkiv är utformat för att hantera din k3s-hemlabbmiljö (med 1 Master-nod och 2 Worker-noder) med hjälp av **FluxCD** och **Kustomize**. Den här strukturen är renodlad för **infrastruktur** ("infrastructure-only repo"), precis som i stora organisationer. Applikationer körs och underhålls i sina egna separata Git-arkiv.

## Arkitektur & Mappstruktur

Alla baskomponenter är helt fristående ("self-contained"). Varje modulmapp innehåller sitt eget `Namespace`, sin egen `HelmRepository` (källa) samt sin `HelmRelease` (installation) eller manifest:

- **`clusters/base/`**: Innehåller de gemensamma modulerna.
  - **`ingress/`**: Nginx Ingress Controller för extern trafik.
  - **`metallb/`**: MetalLB L2 LoadBalancer för lokala IP-adresser.
  - **`cert-manager/`**: Certifikathantering med Let's Encrypt Staging och Production `ClusterIssuers`.
  - **`external-secrets/`**: Synkronisering av externa secrets (från t.ex. Vault, 1Password).
  - **`kyverno/`**: Säkerhetspolicyer. Innehåller [policies.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/kyverno/policies.yaml) som validerar och blockerar privilegierade containrar samt granskar körning som `root` (Pod Security Standards).
  - **`longhorn/`**: Distribuerad persistent lagring optimerad för 2 worker-noder (`defaultReplicaCount: 2`).
  - **`monitoring/`**: Kube-Prometheus-Stack (Prometheus & Grafana) för övervakning.
  - **`logging/`**: **Grafana Loki & Promtail** för centraliserad logghantering. Promtail läser loggar från noderna och skickar dem till Loki, som sparar dem persistent i Longhorn.
  - **`network-policies/`**: **Nätverksisolering**. Innehåller [namespace-isolation.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/network-policies/namespace-isolation.yaml) som implementerar en "Default Deny Ingress" (stoppar all inkommande trafik till poddar som standard) och tillåter trafik enbart från `ingress-nginx`.
  - **`system-upgrade-controller/`**: **Automatiska klusteruppgraderingar**. Innehåller planer ([plans.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/system-upgrade-controller/plans.yaml)) för att utföra rullande uppgraderingar av k3s på dina master- och worker-noder.
- **`clusters/staging/`**, **`clusters/prod/`**, **`clusters/test/`**: Miljöspecifika overlays som ärver från `base` och lägger till miljöspecifika konfigurationer. Mapparna innehåller även `tenant-sync.yaml` som driftsätter alla namespaces och RBAC-roller för dina utvecklingsteam.
- **`tenant/`**:
  - **`tenant-a/`**: En aktiv demo av hur du förbereder multi-tenant isolering för dina externa applikationer. Innehåller [namespace.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/tenant/tenant-a/namespace.yaml) och [rbac.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/tenant/tenant-a/rbac.yaml) som skapar en isolerad miljö där en tenant enbart har admin-rättigheter inom sitt eget namnutrymme.
- **`AGENTS.md`**: Innehåller regler ([AGENTS.md](file:///mnt/c/Users/TeeHit/k8s-infrastructure/AGENTS.md)) för framtida AI-kodningsassistenter som arbetar med detta projekt för att säkerställa att arkitekturen och säkerhetsreglerna bevaras.

---

## Förberedelser innan bootstrap

1. **E-post för Let's Encrypt**:
   Uppdatera din e-postadress i [issuers.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/cert-manager/issuers.yaml) så att du kan ta emot certifikatnotiser från Let's Encrypt.

2. **MetalLB IP-pooler**:
   Kontrollera och justera IP-intervallen för ditt hemnätverkssubnät i:
   - Staging: [metallb-config.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/staging/infrastructure/metallb-config.yaml)
   - Prod: [metallb-config.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/prod/infrastructure/metallb-config.yaml)
   - Test: [metallb-config.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/test/infrastructure/metallb-config.yaml)

---

## Bootstrapa klustret med FluxCD

När ditt Git-arkiv har pushats till din Git-leverantör (t.ex. GitHub), kan du bootstrapa Flux på ditt kluster.

### Exempel: Bootstrap av Staging-miljön

Kör följande kommando på din master-nod (eller maskin med access till ditt k3s-kluster):

```bash
flux bootstrap github \
  --owner=<ditt-github-användarnamn> \
  --repository=k8s-infrastructure \
  --branch=main \
  --path=./clusters/staging \
  --personal
```

### Vad händer under bootstrap?
1. Flux installerar sina egna controller-resurser i namnutrymmet `flux-system`.
2. Flux skapar filerna i `clusters/staging/flux-system/` i ditt repo för att bibehålla sync-statusen.
3. Flux börjar läsa `clusters/staging/kustomization.yaml`, vilket i sin ur:
   - Startar synkroniseringen av **Infrastruktur** (`infra-sync.yaml` -> `./clusters/staging/infrastructure`).
   - Startar synkroniseringen av **Tenants** (`tenant-sync.yaml` -> `./tenant`) så fort infrastrukturen är redo. Det sätter upp alla namespaces och RBAC-rättigheter.
