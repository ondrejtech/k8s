# Docker Registry + Registry UI na GKE

Nasazení privátního Docker registru s webovým rozhraním na Google Kubernetes Engine s certifikátem od Let's Encrypt.

## Architektura

```
HTTPS → GCE LB → Ingress → registry-ui:80 (nginx)
                               ├─ / → UI (static files)
                               └─ /v2 → proxy_pass → registry:5000 (auth)
```

## Předpoklady

- GKE cluster s nainstalovaným cert-managerem
- Doménové jméno (např. `multishoping.eu`) s možností přidat DNS A záznam
- `kubectl` a `gcloud` nástroje

## Postup nasazení

### 1. Rezervovat statickou IP

```bash
gcloud compute addresses create registry-ip --global
gcloud compute addresses describe registry-ip --format='value(address)' --global
# → 136.68.9.54 (příklad)
```

### 2. Nastavit DNS A záznam

V administraci DNS providera (např. Czechia) vytvořit:

```
registry.multishoping.eu  →  136.68.9.54
```

Ověřit propagaci:

```bash
dig registry.multishoping.eu
```

### 3. Vytvořit namespace

```bash
kubectl apply -f namespace-registry.yaml
```

### 4. Nasazení Docker Registry

```bash
# PersistentVolumeClaim pro ukládání image
kubectl apply -f registry-pvc.yaml

# Deployment registru
kubectl apply -f registry-deployment.yaml

# Service registru
kubectl apply -f registry-service.yaml
```

### 5. TLS certifikát přes cert-manager (Let's Encrypt)

Nejprve vytvořit **prázdný TLS secret** – workaround pro chicken-egg problém GCE controlleru:

```bash
kubectl apply -f registry-tls-secret.yaml
```

Poté Issuer a Ingress:

```bash
kubectl apply -f issuer-letsencrypt-prod.yaml
kubectl apply -f ingress-registry.yaml --server-side --force-conflicts
```

cert-manager automaticky:
- Vytvoří Certificate
- Přidá `.well-known/acme-challenge/` path do ingressu
- Let's Encrypt validuje doménu
- Uloží certifikát do secretu `registry-tls`
- GCE controller nahraje certifikát do load balanceru

Sledovat stav:

```bash
kubectl get certificate -n registry -o wide
kubectl describe ingress -n registry registry-ingress
```

Po vydání certifikátu by měl ingress ukazovat porty 80 i 443 a HTTPS by měl fungovat:

```bash
curl -sI https://registry.multishoping.eu/
```

### 6. Autentizace (basic auth)

Secret s htpasswd souborem:

```bash
kubectl apply -f htpasswd-secret.yaml
```

Pro vygenerování nového hesla:

```bash
docker run --rm httpd:alpine htpasswd -Bbn "uzivatel@email.cz" "heslo"
```

### 7. Registry UI (webové rozhraní)

```bash
kubectl apply -f registry-ui-deployment.yaml
kubectl apply -f registry-ui-service.yaml
```

Posledním krokem je přepnutí ingressu z registry backendu na UI (které proxysuje API volání).

Upravit `ingress-registry.yaml` – změnit backend:

```yaml
backend:
  service:
    name: registry-ui   # místo registry
    port:
      number: 80        # místo 5000
```

Aplikovat:

```bash
kubectl apply -f ingress-registry.yaml --server-side --force-conflicts
```

### 8. Ověření

```bash
# UI – v prohlížeči https://registry.multishoping.eu
curl -sI https://registry.multishoping.eu/

# API – vyžaduje autentizaci
curl -s https://registry.multishoping.eu/v2/

# Docker login
echo "heslo" | docker login registry.multishoping.eu -u "uzivatel@email.cz" --password-stdin

# Docker push
docker pull hello-world
docker tag hello-world registry.multishoping.eu/hello-world
docker push registry.multishoping.eu/hello-world

# Docker pull
docker pull registry.multishoping.eu/hello-world
```

## Použité soubory

| Soubor | Popis |
|--------|-------|
| `namespace-registry.yaml` | Namespace `registry` |
| `registry-pvc.yaml` | PersistentVolumeClaim 10Gi |
| `registry-deployment.yaml` | Deployment `registry:2` + auth config |
| `registry-service.yaml` | Service NodePort 5000 |
| `registry-tls-secret.yaml` | Prázdný TLS secret (workaround) |
| `issuer-letsencrypt-prod.yaml` | Issuer pro Let's Encrypt (HTTP-01, `name: registry-ingress`) |
| `ingress-registry.yaml` | Ingress s cert-manager anotací a statickou IP |
| `htpasswd-secret.yaml` | Basic auth credentials (htpasswd) |
| `registry-ui-deployment.yaml` | Deployment `joxit/docker-registry-ui:latest` |
| `registry-ui-service.yaml` | Service NodePort 80 |

## Environment variables registry-ui

| Proměnná | Hodnota | Význam |
|----------|---------|--------|
| `NGINX_PROXY_PASS_URL` | `http://registry:5000` | Backend registry |
| `SINGLE_REGISTRY` | `true` | Režim jednoho registru |
| `REGISTRY_TITLE` | `registry` | Titulek UI |
| `THEME` | `dark` | Tmavý režim |
| `DELETE_IMAGES` | `true` | Povolit mazání image |
| `REGISTRY_SECURED` | `true` | Registry vyžaduje auth |
| `SHOW_CONTENT_DIGEST` | `true` | Zobrazit SHA digesty |
| `SHOW_CATALOG_NB_TAGS` | `true` | Počet tagů v katalogu |
| `TAGLIST_PAGE_SIZE` | `100` | Tagů na stránku |

## Důležité poznámky

- **cert-manager HTTP-01**: Issuer musí mít `http01.ingress.name: registry-ingress` (NE `class: gce`), aby cert-manager přidal challenge path do existujícího ingressu, nevytvářel nový.
- **Empty TLS secret**: Nutný workaround, jinak GCE controller hlásí `404 SSL certificate not found`.
- **Force conflicts**: `--force-conflicts` je potřeba při apply ingressu, protože cert-manager spravuje `.spec.rules` (ACME challenge path).
- **Mazání image**: Po smazání image přes UI je potřeba spustit garbage collector: `kubectl exec -n registry deploy/registry -- registry garbage-collect /etc/docker/registry/config.yml`
