# n8n na GKE

Nasazení n8n (workflow automation) s PostgreSQL na Google Kubernetes Engine s HTTPS certifikátem.

## Architektura

```
HTTPS → GCE LB → Ingress → n8n:5678
                               └─ postgres-service:5432
```

TLS řešeno přes **Google ManagedCertificate** (nativní GKE řešení), nikoliv cert-manager – HTTP-01 challenge vytváří separátní LB, což s GCE Ingressem nefunguje.

## Předpoklady

- GKE cluster
- Doménové jméno (např. `multishoping.eu`) s možností přidat DNS A záznam
- `kubectl` a `gcloud` nástroje

## Postup nasazení

### 1. Rezervovat statickou IP

```bash
gcloud compute addresses create n8n-ip --global
gcloud compute addresses describe n8n-ip --format='value(address)' --global
# → 8.232.167.70 (příklad)
```

### 2. Nastavit DNS A záznam

V administraci DNS providera vytvořit:

```
n8n.multishoping.eu  →  8.232.167.70
```

Ověřit propagaci:

```bash
dig n8n.multishoping.eu
```

### 3. Vytvořit namespace

```bash
kubectl apply -f namespace.yaml
```

### 4. Nasazení PostgreSQL

```bash
# PVC pro data (300Gi)
kubectl apply -f postgres-claim0-persistentvolumeclaim.yaml

# Secret s přihlašovacími údaji
kubectl apply -f postgres-secret.yaml

# ConfigMap s init skriptem (vytvoří non-root uživatele)
kubectl apply -f postgres-configmap.yaml

# Deployment Postgresu 16
kubectl apply -f postgres-deployment.yaml

# Service (headless, ClusterIP: None)
kubectl apply -f postgres-service.yaml
```

### 5. Nasazení n8n

```bash
# PVC pro n8n data (2Gi)
kubectl apply -f n8n-claim0-persistentvolumeclaim.yaml

# Encryption key secret
kubectl apply -f n8n-secret.yaml

# Deployment n8n
kubectl apply -f n8n-deployment.yaml

# Service (NodePort)
kubectl apply -f n8n-service.yaml
```

Důležité konfigurace v deploymentu:

| Proměnná | Hodnota | Význam |
|----------|---------|--------|
| `DB_TYPE` | `postgresdb` | Backend databáze |
| `DB_POSTGRESDB_HOST` | `postgres-service.n8n.svc.cluster.local` | Interní DNS PostgreSQL |
| `N8N_PROTOCOL` | `https` | Protokol pro webhooky |
| `N8N_HOST` | `n8n.multishoping.eu` | Veřejná doména |
| `N8N_PORT` | `5678` | Port aplikace |
| Resources requests | `2048Mi` | Min. paměť |
| Resources limits | `4096Mi` | Max. paměť |

### 6. TLS certifikát (Google ManagedCertificate)

```bash
# ManagedCertificate
kubectl apply -f managed-cert.yaml

# FrontendConfig (HTTP→HTTPS redirect)
kubectl apply -f frontend-config-n8n.yaml

# Ingress
kubectl apply -f ingress-n8n.yaml --server-side
```

Google ManagedCertificate se postará o HTTPS automaticky. Certifikát je vydaný Googlem přímo, bez potřeby Let's Encrypt.

Sledovat stav:

```bash
kubectl get managedcertificate -n n8n -o wide
kubectl get ingress -n n8n -o wide
```

Po dokončení by měl být ingress dostupný na IP a HTTPS by měl fungovat:

```bash
curl -sI https://n8n.multishoping.eu/
```

### 7. Ověření

```bash
# HTTP -> HTTPS redirect (pokud FrontendConfig funguje)
curl -sI http://n8n.multishoping.eu/

# HTTPS
curl -sI https://n8n.multishoping.eu/

# Otevřít v prohlížeči
echo "https://n8n.multishoping.eu"
```

## Použité soubory

| Soubor | Popis |
|--------|-------|
| `namespace.yaml` | Namespace `n8n` |
| `postgres-secret.yaml` | PostgreSQL přihlašovací údaje |
| `postgres-configmap.yaml` | Init skript pro vytvoření non-root uživatele |
| `postgres-claim0-persistentvolumeclaim.yaml` | PVC 300Gi pro Postgres |
| `postgres-deployment.yaml` | Deployment PostgreSQL 16 |
| `postgres-service.yaml` | Headless service Postgres |
| `n8n-secret.yaml` | Encryption key pro n8n |
| `n8n-claim0-persistentvolumeclaim.yaml` | PVC 2Gi pro n8n |
| `n8n-deployment.yaml` | Deployment n8n (stable) |
| `n8n-service.yaml` | Service NodePort 5678 |
| `managed-cert.yaml` | Google ManagedCertificate pro `n8n.multishoping.eu` |
| `frontend-config-n8n.yaml` | HTTP→HTTPS redirect |
| `ingress-n8n.yaml` | Ingress se statickou IP a managed cert |
| `cluster-issuer.yaml` | (nepoužito) ClusterIssuer pro cert-manager – zachováno pro referenci |
| `certificate.yaml` | (nepoužito) Certificate resource – zachováno pro referenci |
| `ingress-n8n-tls-backup.yaml` | (nepoužito) Backup ingress s cert-manager configem |

## Důležité poznámky

- **NodeSelector**: Všechny pody běží na nodu s labelem `l4-gpu: "true"` (main-pool).
- **ManagedCertificate**: Oproti registru používá n8n Google ManagedCertificate, nikoliv cert-manager. Je to jednodušší a plně integrované s GKE. cert-manager HTTP-01 na GKE nefunguje, protože vytváří separátní Ingress s vlastním LB.
- **FrontendConfig**: HTTP→HTTPS redirect není vždy funkční s ManagedCertificate. HTTPS přístup funguje vždy.
- **Postgres PVC**: 300Gi, použít `pd-standard` storage class.
- **InitContainer n8n**: Opravuje permissions pro `/data` na uživatele `1000:1000`.
- **První přihlášení**: Po nasazení otevřít `https://n8n.multishoping.eu/` a vytvořit admin účet (během prvního přístupu).
