# cert-manager – instalace přes Helm

cert-manager je Kubernetes add-on pro vydávání TLS certifikátů. Podporuje Let's Encrypt, HashiCorp Vault, Venafi a self-signed certifikáty.

## Předpoklady

- Kubernetes cluster (GKE, EKS, AKS, ...)
- Helm CLI nainstalovaný (viz `../install-helm.sh`)
- Přístup do OCI registru `quay.io`

## Instalace cert-manageru

### Pomocí připraveného skriptu

```bash
chmod +x install.sh
./install.sh
```

### Manuálně krok za krokem

#### 1. Stažení GPG klíče pro ověření podpisu chartu

```bash
curl -LO https://cert-manager.io/public-keys/cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg
```

Tento klíč slouží k ověření integrity a autenticity Helm chartu před instalací. Pomocí `--verify` a `--keyring` Helm zkontroluje, že chart není padělaný.

#### 2. Instalace chartu

```bash
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.0 \
  --namespace cert-manager \
  --create-namespace \
  --verify \
  --keyring ./cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg \
  --set crds.enabled=true
```

### Parametry instalace

| Parametr | Hodnota | Význam |
|----------|---------|--------|
| `cert-manager` | (název release) | Jméno Helm releasu |
| `oci://quay.io/jetstack/charts/cert-manager` | URL | OCI registry s chartem |
| `--version` | `v1.21.0` | Konkrétní verze cert-manageru |
| `--namespace` | `cert-manager` | Namespace pro instalaci |
| `--create-namespace` | (přepínač) | Vytvoří namespace, pokud neexistuje |
| `--verify` | (přepínač) | Ověří podpis chartu pomocí GPG klíče |
| `--keyring` | cesta ke klíči | GPG veřejný klíč pro ověření |
| `--set crds.enabled=true` | `true` | Automaticky nainstaluje CRD |

## Ověření instalace

### Pods

```bash
kubectl get pods -n cert-manager
```

Očekávaný výstup:

```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-8fcb9d456-xxxxx               1/1     Running   0          2m
cert-manager-cainjector-85c8bf6d8b-xxxxx   1/1     Running   0          2m
cert-manager-webhook-85d7d5497-xxxxx       1/1     Running   0          2m
```

### Custom Resource Definitions

```bash
kubectl get crd | grep cert-manager
```

Očekávaný výstup:

```
certificaterequests.cert-manager.io        2026-07-29
certificates.cert-manager.io               2026-07-29
challenges.acme.cert-manager.io            2026-07-29
clusterissuers.cert-manager.io             2026-07-29
issuers.cert-manager.io                    2026-07-29
orders.acme.cert-manager.io                2026-07-29
```

## První kroky po instalaci

### Vytvoření ClusterIssuer (Let's Encrypt Production)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
    - http01:
        ingress:
          name: my-ingress
```

### Vytvoření namespace-scoped Issuer (Let's Encrypt Staging)

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: letsencrypt-staging
  namespace: my-app
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          name: my-ingress
```

### Důležité upozornění pro GKE

Při použití HTTP-01 challenge na GKE musí mít solver nastavení `http01.ingress.name: <ingress-name>` (název existujícího ingressu), nikoliv `class: gce`. Pokud je použitý `class`, cert-manager vytvoří separátní Ingress s vlastním load balancerem a Let's Encrypt nedokáže validovat doménu (dostane 404).

Více v oficiálním tutoriálu:
https://cert-manager.io/docs/tutorials/getting-started-with-cert-manager-on-google-kubernetes-engine-using-lets-encrypt-for-ingress-ssl/

## Upgrade cert-manageru

```bash
helm upgrade \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.22.0 \
  --namespace cert-manager \
  --verify \
  --keyring ./cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg \
  --set crds.enabled=true
```

## Odinstalace

```bash
helm uninstall cert-manager -n cert-manager
```

## Použité soubory

| Soubor | Popis |
|--------|-------|
| `install.sh` | Instalační skript cert-manageru |
| `README.md` | Tento soubor |

## Užitečné zdroje

- [cert-manager Documentation](https://cert-manager.io/docs/)
- [cert-manager Helm Chart](https://artifacthub.io/packages/helm/cert-manager/cert-manager)
- [Getting Started on GKE](https://cert-manager.io/docs/tutorials/getting-started-with-cert-manager-on-google-kubernetes-engine-using-lets-encrypt-for-ingress-ssl/)
