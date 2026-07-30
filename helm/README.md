# Helm – instalace a použití

Helm je package manager pro Kubernetes. Umožňuje instalovat, spravovat a aktualizovat aplikace v K8s clusteru pomocí tzv. chartů.

## Instalace Helm CLI

### Debian/Ubuntu (APT)

Pomocí připraveného skriptu:

```bash
chmod +x install-helm.sh
./install-helm.sh
```

Skript provede:

1. Instalace závislostí (`curl`, `gpg`, `apt-transport-https`)
2. Stažení GPG klíče Helm repozitáře a ověření otisku
3. Přidání APT repozitáře Helm Buildkite
4. Instalace Helm CLI

Manuálně krok za krokem:

```bash
# Závislosti
sudo apt-get install curl gpg apt-transport-https --yes

# Stažení GPG klíče
curl -fsSL https://packages.buildkite.com/helm-linux/helm-debian/gpgkey > /tmp/helm.gpg

# Ověření klíče (ochrana proti kompromitaci repozitáře)
HELM_BUILDKITE_APT_KEY_ID="DDF78C3E6EBB2D2CC223C95C62BA89D07698DBC6"
if [ "$(gpg --show-keys --with-colons /tmp/helm.gpg | awk -F: '$1 == "fpr" {print $10}' | head -n 1)" != "${HELM_BUILDKITE_APT_KEY_ID}" ]; then
  echo "ERROR: Unexpected Helm APT key ID: potential key compromise"
  exit 1
fi

# Přidání repozitáře
cat /tmp/helm.gpg | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/helm.gpg] https://packages.buildkite.com/helm-linux/helm-debian/any/ any main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

# Instalace
sudo apt-get update
sudo apt-get install helm
```

### macOS (Homebrew)

```bash
brew install helm
```

### Windows (chocolatey)

```powershell
choco install kubernetes-helm
```

## Ověření instalace

```bash
helm version
# → version.BuildInfo{Version:"v3.17.2", ...}

helm help
```

## Základní příkazy

| Příkaz | Popis |
|--------|-------|
| `helm repo add <name> <url>` | Přidání chart repozitáře |
| `helm repo update` | Aktualizace repozitářů |
| `helm search hub <keyword>` | Vyhledávání v Artifact HUB |
| `helm install <name> <chart>` | Instalace chartu |
| `helm upgrade <name> <chart>` | Upgrade release |
| `helm uninstall <name>` | Odstranění release |
| `helm list` | Seznam nainstalovaných release |
| `helm show values <chart>` | Zobrazení výchozích hodnot chartu |
| `helm get values <name>` | Zobrazení aktuálních hodnot release |
| `helm history <name>` | Historie release |

## Instalace cert-manager přes Helm

Repozitář obsahuje skript pro instalaci cert-manageru:

```bash
chmod +x cert-manager/install.sh
./cert-manager/install.sh
```

Skript provede:

1. Stažení GPG klíče pro ověření podpisu chartu
2. Instalace cert-manageru do namespace `cert-manager` s ověřením podpisu

Manuálně:

```bash
# Stažení veřejného klíče pro ověření podpisu
curl -LO https://cert-manager.io/public-keys/cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg

# Instalace cert-manager chartu s ověřením
helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.21.0 \
  --namespace cert-manager \
  --create-namespace \
  --verify \
  --keyring ./cert-manager-keyring-2021-09-20-1020CF3C033D4F35BAE1C19E1226061C665DF13E.gpg \
  --set crds.enabled=true
```

Parametry:

| Parametr | Význam |
|----------|--------|
| `oci://quay.io/jetstack/charts/cert-manager` | OCI registry s chartem |
| `--version v1.21.0` | Verze cert-manageru |
| `--namespace cert-manager` | Cílový namespace |
| `--create-namespace` | Vytvořit namespace, pokud neexistuje |
| `--verify` | Ověřit podpis chartu |
| `--keyring` | GPG klíč pro ověření |
| `--set crds.enabled=true` | Automaticky nainstalovat CRD |

Ověření instalace cert-manageru:

```bash
kubectl get pods -n cert-manager
# NAME                                       READY   STATUS    RESTARTS   AGE
# cert-manager-8fcb9d456-xxxxx               1/1     Running   0          5m
# cert-manager-cainjector-85c8bf6d8b-xxxxx   1/1     Running   0          5m
# cert-manager-webhook-85d7d5497-xxxxx       1/1     Running   0          5m

kubectl get crd | grep cert-manager
# certificaterequests.cert-manager.io
# certificates.cert-manager.io
# challenges.acme.cert-manager.io
# clusterissuers.cert-manager.io
# issuers.cert-manager.io
# orders.acme.cert-manager.io
```

## Použité soubory

| Soubor | Popis |
|--------|-------|
| `install-helm.sh` | Instalační skript Helm CLI pro Debian/Ubuntu |
| `cert-manager/install.sh` | Instalační skript cert-manageru přes Helm |

## Užitečné zdroje

- [Helm Docs](https://helm.sh/docs/)
- [Artifact HUB](https://artifacthub.io/) – veřejný katalog Helm chartů
- [cert-manager Docs](https://cert-manager.io/docs/)
