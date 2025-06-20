# Helm Chart za BlogSpace Aplikaciju

Jednostavni Helm chart koji omogućuje postavljanje [**BlogSpace**](https://github.com/OliverVujica/BlogSpace) Java aplikacije i pripadajuće **MySQL** baze podataka na Kubernetes klaster.
Chart je napravljen za **edukacijske svrhe**.

## Opis

Chart deploya two-tier arhitekturu koja se sastoji od:

* [BlogSpace Aplikacije](https://github.com/OliverVujica/BlogSpace): Stateless backend aplikacija napisana u Javi, zapakirana kao Docker slika.
* **MySQL Baze Podataka**: Stateful instanca MySQL baze koja služi za pohranu podataka aplikacije.

## Arhitektura

Kada se chart deploya, kreiraju se sljedeći Kubernetes resursi:

* **Deployment**: Za pokretanje BlogSpace aplikacije. Omogućuje laku skalabilnost (promjenom broja replika).
* **StatefulSet**: Za pokretanje MySQL baze podataka. Osigurava stabilan mrežni identitet i perzistentnu pohranu za bazu.
* **Service (ClusterIP)**: Kreira interni servis za BlogSpace aplikaciju, čineći je dostupnom unutar klastera.
* **Headless Service**: Kreira se za MySQL StatefulSet kako bi se omogućila izravna komunikacija s podovima baze.
* **PersistentVolumeClaim (PVC)**:
    * Jedan PVC za BlogSpace aplikaciju koji služi za pohranu uploadanih datoteka (npr. slika).
    * Jedan PVC (kroz `volumeClaimTemplates`) za MySQL bazu podataka kako bi podaci bili sačuvani i nakon restarta poda.
* **Secret**: Za sigurno pohranjivanje i upravljanje lozinkama za MySQL bazu podataka.

## Preduvjeti

* Kubernetes klaster
* [Helm](https://helm.sh/docs/intro/quickstart/) 
* Instaliran `kubectl`
* Konfiguriran `StorageClass` na klasteru koji podržava dinamičko provisioniranje volumena (npr. Longhorn, Ceph, itd.). Zadana vrijednost u chartu je `longhorn`.
* Instaliran Ingress kontroler na klasteru (npr. NGINX Ingress Controller) ako želite koristiti Ingress.

## Instalacija Charta

1.  **Klonirajte repozitorij (ili preuzmite chart)**
    ```sh
    git clone <URL_VAŠEG_REPOZITORIJA>
    cd blogspace-chart
    ```
2.  **Prilagodite `values.yaml` (Opcionalno)**
    Pregledajte i po potrebi izmijenite datoteku `values.yaml`.
3.  **Instalirajte chart**
    Koristite `helm install` naredbu za deployanje aplikacije. Preporučuje se instalacija u poseban namespace.
    ```sh
    helm install blogspace . --namespace blogspace --create-namespace
    ```

## Pristup Aplikaciji putem Ingressa

Po zadanim postavkama, servis za aplikaciju je tipa `ClusterIP`, što znači da je dostupan samo unutar klastera. Najbolji način za izlaganje aplikacije izvan klastera je putem Ingress resursa. <br>
Preduvjet za ispravan rad Ingressa je postojanje DNS zapisa za vašu domenu. U postavkama svoje domene (npr. na Cloudflareu) treba dodati A zapis (koji pokazuje na IP adresu Load Balancera vašeg Ingress kontrolera ili jednostavno na ip adresu node-a) ili CNAME zapis (koji pokazuje na postojeći DNS naziv kontrolera).

**1. Kreirajte Ingress YAML datoteku**

Spremite sljedeći sadržaj u datoteku, npr. `ingress.yaml`. Ovaj primjer koristi NGINX Ingress Controller i `cert-manager` za automatsko generiranje TLS certifikata.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: blogspace-ingress
  # VAŽNO: Ovaj namespace mora biti isti kao onaj gdje je instaliran vaš chart
  namespace: blogspace 
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    # Ovdje unesite vašu domenu
    - blogspace.mojadomena.com
    secretName: blogspace-tls
  rules:
  - host: blogspace.mojadomena.com # I ovdje unesite vašu domenu
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            # Naziv servisa definiran je u chartu kao {{ .Values.app.name }}-service
            name: blogspace-service
            port:
              # Port servisa definiran je u values.yaml i iznosi 8080
              number: 8080 
```

**2. Primijenite Ingress na klaster**

Koristite `kubectl apply` kako biste kreirali Ingress resurs.

```sh
kubectl apply -f ingress.yaml
```

Nakon nekoliko trenutaka, vaš Ingress kontroler će dodijeliti javnu IP adresu i promet na `blogspace.mojadomena.com` će se usmjeravati na vašu aplikaciju.

## Deinstalacija Charta

Za uklanjanje deployane aplikacije i svih njenih resursa, koristite `helm uninstall` naredbu:

```sh
helm uninstall blogspace --namespace blogspace
```

## Konfiguracija

Sve konfiguracijske opcije nalaze se u `values.yaml` datoteci.

### Konfiguracija Aplikacije (`app`)

| Parametar | Opis | Zadana vrijednost |
| :--- | :--- | :--- |
| `name` | Naziv resursa za aplikaciju. | `blogspace` |
| `replicaCount` | Broj replika (podova) za aplikaciju. | `1` |
| `image.repository`| Repozitorij Docker slike za aplikaciju. | `olivervja/blogspace` |
| `image.pullPolicy`| Politika povlačenja slike. | `IfNotPresent` |
| `image.tag` | Tag Docker slike. | `"latest"` |
| `service.type` | Tip Kubernetes servisa za aplikaciju. | `ClusterIP` |
| `service.port` | Port na kojem servis izlaže aplikaciju. | `8080` |
| `persistence.enabled`| Omogućuje li se perzistentna pohrana. | `true` |
| `persistence.storageClass`| Naziv StorageClass-a za PVC. | `"longhorn"` |
| `persistence.accessMode`| Access mode za PVC. | `ReadWriteOnce` |
| `persistence.size`| Veličina volumena za pohranu. | `1Gi` |
| `persistence.mountPath`| Putanja unutar containera gdje se volumen montira.| `/app/uploads` |
| `resources` | CPU/Memory resursi (requests i limits). | (pogledati `values.yaml`) |

### Konfiguracija MySQL Baze (`mysql`)

| Parametar | Opis | Zadana vrijednost |
| :--- | :--- | :--- |
| `name` | Naziv resursa za MySQL. | `mysql` |
| `image.repository`| Repozitorij Docker slike za MySQL. | `mysql` |
| `image.tag` | Tag Docker slike. | `"latest"` |
| `db.name` | Naziv baze podataka koja će se kreirati. | `blog` |
| `db.user` | Korisničko ime za pristup bazi. | `restouser` |
| `persistence.enabled`| Omogućuje li se perzistentna pohrana za bazu. | `true` |
| `persistence.storageClass`| Naziv StorageClass-a za PVC baze. | `"longhorn"` |
| `persistence.accessMode`| Access mode za PVC baze. | `ReadWriteOnce` |
| `persistence.size`| Veličina volumena za pohranu podataka baze. | `2Gi` |
| `persistence.mountPath`| Putanja unutar containera gdje se podaci baze pohranjuju.| `/var/lib/mysql` |
| `resources` | CPU/Memory resursi (requests i limits) za bazu.| (pogledati `values.yaml`) |
