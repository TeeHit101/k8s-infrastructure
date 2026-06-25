# Coding Assistant Rules for k8s-infrastructure

Den här filen innehåller regler och riktlinjer för framtida AI-assistenter som arbetar med detta GitOps-arkiv. Följ dessa regler strikt för att bibehålla projektets integritet, säkerhet och struktur.

---

## 1. Arkitektur och Mappstruktur
* **Renodlad Infrastruktur (Infrastructure-Only)**: Det här arkivet får *inte* innehålla applikationsmanifest (Deployments, Services etc. för slutanvändarappar). Applikationer ska hanteras i separata Git-arkiv. Det här arkivet hanterar endast klusterinfrastruktur och övergripande tenants (namnutrymmen och RBAC).
* **Fristående Moduler (Self-Contained Modules)**: Varje komponent under [clusters/base/](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/) måste vara helt självförsörjande. En modulmapp ska innehålla:
  - `namespace.yaml`
  - `repository.yaml` (lokal `HelmRepository`)
  - `release.yaml` (lokal `HelmRelease`)
  - `kustomization.yaml` (som bundlar ovanstående resurser)
* **Gemensamma källor**: Skapa inte en central `helm/`-mapp för källor; behåll dem lokalt i modulens egen mapp för att underlätta underhåll.

---

## 2. Versionshantering och Dependabot
* **Spikade versioner (Version Pinning)**: Alla `HelmRelease`-definitioner måste använda **exakta versioner** (t.ex. `version: 1.15.0`) istället för intervall (som `>=1.15.0 <1.16.0`). Detta krävs för att **Dependabot** ska kunna spåra och uppdatera versionerna på ett deterministiskt sätt.
* **GitHub Actions**: Alla Actions i `.github/workflows/` måste refereras med sina unika, **immutabla commit-SHAs** istället för rörliga versionstaggar (t.ex. `uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683` istället för `uses: actions/checkout@v4`). Lägg alltid till en kommentar som anger versionen (t.ex. `# v4.2.2`).

---

## 3. Säkerhet och Policyer
* **Kyverno Security**: Alla nya system eller pods som läggs till måste följa säkerhetsreglerna i [policies.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/kyverno/policies.yaml). Undvik att skapa privilegierade containrar om det inte är absolut nödvändigt för plattformsverktyg.
* **Nätverksisolering**: Alla nya tjänster som ska exponeras externt i `default`-namnutrymmet måste följa nätverksreglerna i [namespace-isolation.yaml](file:///mnt/c/Users/TeeHit/k8s-infrastructure/clusters/base/network-policies/namespace-isolation.yaml) och tillåta nätverkstrafik via `ingress-nginx`.

---

## 4. Validering före Commit
* **Kustomize Validation**: Innan du föreslår ändringar eller slutför en turn, verifiera alltid att alla overlays bygger korrekt genom att köra:
  ```bash
  kustomize build clusters/staging > /dev/null && \
  kustomize build clusters/prod > /dev/null && \
  kustomize build clusters/test > /dev/null && \
  kustomize build tenant > /dev/null
  ```
  Detta motsvarar CI-pipelinen och fångar upp eventuella referensfel.
