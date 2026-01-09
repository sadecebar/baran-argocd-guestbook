# CI/CD Slutprojekt – Guestbook med ArgoCD

**Student:** Baran Kizilca  
**Kurs:** CI/CD  
**Typ av arbete:** Individuellt slutprojekt

## Åtkomst och examination
**URL till applikation:** http://baran-guestbook-grupp5.apps.devops24.cloud2.se
**GitHub Repository:** https://github.com/sadecebar/baran-argocd-guestbook

## Inledning
Detta projekt utgör slutuppgiften i CI/CD-kursen och bygger vidare på det tidigare grupparbetet Automate Guestbook. Syftet med uppgiften var att vidareutveckla applikationen genom att införa ArgoCD och arbeta enligt GitOps-principer, där Git används som enda källa till sanning för både applikationskod och driftsättning.

Projektet genomfördes individuellt. För att undvika konflikter med andra studenters resurser sattes en helt separat instans av Guestbook upp i OpenShift. Samtliga resurser prefixades konsekvent med mitt namn (baran-) för att säkerställa isolering och förhindra oavsiktliga krockar i klustret.

## Översikt av lösningen
Lösningen består av följande huvudkomponenter:

* **Backend** – Ett Golang-baserat API som hanterar inlägg och statistik
* **Frontend** – En Nginx-baserad webbapplikation som proxyar API-anrop till backend
* **PostgreSQL** – Databas med persistent lagring för inlägg
* **GitHub Actions** – CI-lösning för att bygga och publicera container images
* **GitHub Container Registry (GHCR)** – Registry där images lagras
* **ArgoCD** – Verktyg för GitOps-baserad driftsättning
* **OpenShift** – Plattform där applikationen körs

## Arkitektur
Applikationen följer en klassisk multi-tier-arkitektur:

Internet  
↓  
OpenShift Route  
↓  
Frontend Service → Frontend Pod (Nginx)  
↓  
Backend Service → Backend Pods  
↓  
PostgreSQL (Persistent Storage)

Arkitekturen är medvetet uppbyggd för att efterlikna en verklig produktionsmiljö med tydlig separation mellan presentation, affärslogik och datalager.

## CI – GitHub Actions
För Continuous Integration används GitHub Actions. Vid varje push till main triggas ett workflow som automatiskt:

1. Checkar ut koden från GitHub
2. Bygger backend-imagen från guestbook-backend/Dockerfile
3. Bygger frontend-imagen från guestbook-frontend/Dockerfile
4. Publicerar båda images till GitHub Container Registry

Exempel på images som byggs:
* `ghcr.io/sadecebar/baran-guestbook-backend:latest`
* `ghcr.io/sadecebar/baran-guestbook-frontend:latest`

GitHub Actions används endast för build och push. Ingen deployment sker här, vilket är ett medvetet arkitekturval. Detta skapar en tydlig separation mellan CI och CD och gör flödet enklare att felsöka och underhålla.

## CD – ArgoCD och GitOps
Continuous Deployment hanteras helt av ArgoCD enligt GitOps-principer. En ArgoCD Application är konfigurerad för att peka på detta GitHub-repo och mappen `openshift/`, där samtliga OpenShift-resurser definieras.

Följande inställningar är aktiverade i ArgoCD:
* **Auto-sync** – Ändringar i Git deployas automatiskt
* **Prune** – Resurser som tas bort ur Git tas även bort ur klustret
* **Self-heal** – Manuella ändringar i OpenShift återställs automatiskt

Detta innebär att GitHub-repot fungerar som single source of truth och att klustret alltid hålls synkroniserat med det önskade tillståndet.

## OpenShift-resurser
Alla resurser i OpenShift definieras som YAML-filer i mappen `openshift/`.

### Backend
Backend körs som en Deployment med definierade CPU- och minnesbegränsningar, vilket är ett krav i OpenShift-miljön. Databasuppgifter tillhandahålls via Secrets, vilket gör att inga känsliga värden behöver hårdkodas i konfigurationen.
Backend exponeras internt via en Service med namnet backend, vilket matchar Nginx-konfigurationen i frontend-containern och möjliggör korrekt service discovery via klustrets DNS.

### Frontend
Frontend består av en Nginx-container som serverar webbgränssnittet och proxyar API-anrop till backend-tjänsten. Även frontend har tydligt definierade resursgränser för att uppfylla OpenShift-krav och säkerställa stabil schemaläggning.
Frontend exponeras externt via en OpenShift Route.

### PostgreSQL
PostgreSQL är konfigurerad med:
* Secret för databasnamn, användarnamn och lösenord
* PersistentVolumeClaim för att säkerställa att data finns kvar vid omstarter
* Deployment med persistent lagring
* Service för intern kommunikation

Denna uppsättning säkerställer att data överlever både pod-omstarter och uppdateringar av applikationen.

## Extern åtkomst och verifiering
Applikationen exponeras via en OpenShift Route och kan nås från en vanlig webbläsare utanför klustret. Funktionaliteten verifierades genom att skriva inlägg i guestbooken och kontrollera att dessa finns kvar efter omladdning samt vid åtkomst via inkognito-läge.
Detta bekräftar att persistent lagring fungerar korrekt och att applikationen uppfyller kraven för extern åtkomst.

## Skalning med ArgoCD
För att verifiera att GitOps-flödet fungerade korrekt testades horisontell skalning av backend-tjänsten. Detta gjordes genom att ändra värdet för `replicas` i deployment-YAML-filen i GitHub-repot. Ändringen committades till main utan att några manuella kommandon kördes i OpenShift.

När ändringen pushades upptäckte ArgoCD automatiskt skillnaden mellan önskat tillstånd i Git och det faktiska tillståndet i klustret. ArgoCD genomförde därefter en automatisk uppskalning av backend-tjänsten från en till tre pods. Samtliga pods startade korrekt och applikationen fortsatte fungera utan avbrott.
Detta test bekräftade att ArgoCD korrekt hanterar både driftsättning och skalning baserat på Git som single source of truth.

## Felsökning och lärdomar
Under arbetets gång uppstod flera problem som bidrog till en djupare förståelse för både OpenShift och GitOps:

* **ImagePullBackOff** uppstod när backend-imagen ännu inte fanns publicerad i GHCR
* **PVC Pending** löstes genom att låta OpenShift använda default StorageClass
* **Nginx upstream-fel** orsakades av mismatch mellan Service-namn och Nginx-konfiguration
* **OutOfSync** i ArgoCD löstes genom att aktivera prune och låta ArgoCD ta bort överflödiga resurser

Gemensamt för samtliga problem var att de löstes genom att analysera ArgoCD:s status och konsekvent göra ändringar via Git istället för manuella ingrepp i klustret.

## Reflektion
Projektet gav en tydlig förståelse för skillnaden mellan traditionell CI/CD och GitOps. Genom att låta ArgoCD hantera all driftsättning minimeras risken för konfigurationsdrift och manuella misstag. Separationen mellan CI och CD skapade ett tydligare ansvarsförhållande och gjorde lösningen mer robust och lättare att felsöka.
Arbetet visade även vikten av konsekvent namngivning, tydliga resursbegränsningar och korrekt hantering av beroenden mellan tjänster.

## Förbättringsförslag
Med mer tid hade följande förbättringar kunnat implementeras:
* Privata container images med imagePullSecrets
* TLS-konfiguration på OpenShift Routes
* Horizontal Pod Autoscaler (HPA)
* Backup- och restore-strategi för PostgreSQL
* Separata namespaces för olika miljöer (dev/test/prod)

## Sammanfattning
Projektet uppfyller samtliga krav för slutuppgiften:
* Guestbook är externt tillgänglig via URL
* Data lagras persistent
* GitHub Actions bygger och publicerar images automatiskt
* ArgoCD hanterar all driftsättning
* Ändringar i kod och YAML deployas automatiskt
* Skalning fungerar via GitOps

Lösningen demonstrerar ett komplett och modernt CI/CD-flöde baserat på GitOps och ArgoCD, motsvarande arbetssätt som används i professionella Kubernetes-miljöer.
