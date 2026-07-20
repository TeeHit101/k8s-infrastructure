# Kubernetes GitOps Infrastructure for k3s Homelab

Detta GitOps-arkiv är utformat för att hantera min k3s-hemlabbmiljö (med 1 Master-nod och 2 Worker-noder) med hjälp av **Flux Operator** och **Kustomize**. Den här strukturen är renodlad för **infrastruktur** ("infrastructure-only repo"), precis som i stora organisationer. Applikationer körs och underhålls i sina egna separata Git-arkiv.

## Arkitektur & Mappstruktur

Alla baskomponenter är helt fristående ("self-contained"). Varje modulmapp innehåller sitt eget `Namespace`, sin egen `HelmRepository` (källa) samt sin `HelmRelease` (installation) eller manifest:

- **`clusters/base/`**: Innehåller de gemensamma modulerna.
  - **`flux-operator/`**: **Flux Operator**. Driftsätter operatören som hanterar hela livscykeln för Flux på klustret.
  - **`ingress/`**: Nginx Ingress Controller för extern trafik.
  - **`metallb/`**: MetalLB L2 LoadBalancer för lokala IP-adresser.
  - **`cert-manager/`**: Certifikathantering med Let's Encrypt Staging och Production `ClusterIssuers`.
  - **`external-secrets/`**: Synkronisering av externa secrets (från t.ex. Vault, 1Password).
  - **`kyverno/`**: Säkerhetspolicyer. Innehåller [policies.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/kyverno/policies.yaml) som validerar och blockerar privilegierade containrar samt granskar körning som `root` (Pod Security Standards).
  - **`ceph-csi/`**: Drivrutin för att ansluta klustret till ett externt persistent Ceph-lagringskluster (RBD) över nätverket.
  - **`monitoring/`**: Kube-Prometheus-Stack (Prometheus & Grafana) för övervakning.
  - **`logging/`**: **Grafana Loki & Promtail** för centraliserad logghantering. Promtail läser loggar från noderna och skickar dem till Loki, som sparar dem persistent i Ceph-klustret.
  - **`network-policies/`**: **Nätverksisolering**. Innehåller [namespace-isolation.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/network-policies/namespace-isolation.yaml) som implementerar en "Default Deny Ingress" (stoppar all inkommande trafik till poddar som standard) och tillåter trafik enbart från `ingress-nginx`.
  - **`system-upgrade-controller/`**: **Automatiska klusteruppgraderingar**. Innehåller planer ([plans.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/system-upgrade-controller/plans.yaml)) för att utföra rullande uppgraderingar av k3s på dina master- och worker-noder.
- **`clusters/staging/`**, **`clusters/prod/`**, **`clusters/test/`**: Miljöspecifika overlays som ärver från `base` och lägger till miljöspecifika konfigurationer. Mapparna innehåller även `tenant-sync.yaml` som driftsätter alla namespaces och RBAC-roller för dina utvecklingsteam, samt `flux-instance.yaml` som är den deklarativa konfigurationen för att starta din Flux-agent.
- **`tenant/`**:
  - **`tenant-a/`**: En aktiv demo av hur du förbereder multi-tenant isolering för dina externa applikationer. Innehåller [namespace.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/tenant/tenant-a/namespace.yaml) och [rbac.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/tenant/tenant-a/rbac.yaml) som skapar en isolerad miljö där en tenant enbart har admin-rättigheter inom sitt eget namnutrymme.
- **`AGENTS.md`**: Innehåller regler ([AGENTS.md](file:///mnt/c/Users/TeeHit/k8s-infrastructure/AGENTS.md)) för framtida AI-kodningsassistenter som arbetar med detta projekt för att säkerställa att arkitekturen och säkerhetsreglerna bevaras.

---

## Förberedelser innan bootstrap

1. **Skapa Ceph-CSI Secret manuellt i klustret (Säkerhetshantering)**:
   Eftersom känsliga nycklar inte ska sparas i klartext i Git, måste du skapa autentiseringshemligheten för ditt externa Ceph-kluster manuellt på din master-nod innan du bootstrapar:
   ```bash
   kubectl create namespace ceph-csi
   kubectl create secret generic ceph-csi-rbd-secret \
     --namespace ceph-csi \
     --from-literal=userID="k3s" \
     --from-literal=userKey="<DIN_CEPH_NYCKEL>"
   ```

2. **E-post för Let's Encrypt**:
   Uppdatera din e-postadress i [issuers.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/cert-manager/issuers.yaml) så att du kan ta emot certifikatnotiser från Let's Encrypt.

3. **MetalLB IP-pooler**:
   Kontrollera och justera IP-intervallen för ditt hemnätverkssubnät i:
   - Staging: [metallb-config.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/staging/infrastructure/metallb-config.yaml)
   - Prod: [metallb-config.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/prod/infrastructure/metallb-config.yaml)
   - Test: [metallb-config.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/test/infrastructure/metallb-config.yaml)

---

## Bootstrapa klustret med Flux Operator

För att bootstrapa ditt kluster helt deklarativt utan att köra det manuella `flux bootstrap`-kommandot på din lokala maskin:

### 1. Installera Flux Operator
Kör följande kommando på din master-nod (eller maskin med kubectl-access till klustret) för att installera operatören i namnutrymmet `flux-system`:
```bash
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace \
  --wait
```

### 2. Driftsätt din FluxInstance
När operatören är igång på klustret kan du starta synkroniseringen genom att tillämpa `flux-instance.yaml` för din miljö (t.ex. `test`):
```bash
kubectl apply -f clusters/test/flux-instance.yaml
```

Operatören kommer nu automatiskt att installera Flux-kontrollerna i klustret, ansluta till ditt Git-arkiv och börja synkronisera all infrastruktur och dina tenants i rätt ordning!
