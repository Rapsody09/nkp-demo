# WordPress — Démo StatefulSet via Flux HelmRelease

Démo NKP : WordPress + MariaDB déployés via **Flux HelmRelease**, exposés en HTTPS sur `https://wordpress.poc.devops-hc.dev` grâce à **cert-manager** + **external-dns**, images tirées du miroir Dockerhub `nkpdemo.devops-hc.dev:5000/dockerhub-cache`.

## Architecture

```
                  Internet
                     │
                     ▼
            ┌─────────────────┐
            │ Ingress (nginx) │  ← cert-manager (TLS) + external-dns (DNS)
            └────────┬────────┘
                     │
              ┌──────▼──────┐
              │  WordPress  │  StatefulSet (1 replica, PVC 10Gi)
              │ (php-fpm +  │  → wp-content (thèmes, plugins, uploads)
              │   apache)   │
              └──────┬──────┘
                     │
              ┌──────▼──────┐
              │   MariaDB   │  StatefulSet (1 replica, PVC 8Gi)
              │ (sous-chart)│  → données du site
              └─────────────┘
```

## Structure

| Fichier | Rôle |
|---|---|
| [namespace.yaml](namespace.yaml) | Namespace `wordpress` |
| [secret.yaml](secret.yaml) | Mots de passe WP + MariaDB (à remplacer par sealed-secrets/sops en prod) |
| [helmrepository.yaml](helmrepository.yaml) | Source Flux : repo Bitnami |
| [helmrelease.yaml](helmrelease.yaml) | Release Flux + values (mirror registry, StatefulSet, ingress) |
| [kustomization.yaml](kustomization.yaml) | Aggregator Kustomize |

## Pré-requis cluster

- Flux installé (`source-controller` + `helm-controller`).
- `cert-manager` avec un `ClusterIssuer` nommé `letsencrypt-prod`.
- `external-dns` configuré sur la zone `poc.devops-hc.dev`.
- Ingress controller avec `ingressClassName: nginx`.
- StorageClass par défaut (PVC RWO, 10Gi + 8Gi).
- Miroir `nkpdemo.devops-hc.dev:5000/dockerhub-cache` capable de pull `bitnami/wordpress` et `bitnami/mariadb`.

## Déploiement

Ces manifests sont normalement embarqués dans une `Kustomization` Flux (côté repo de bootstrap). En manuel :

```powershell
kubectl apply -k .\helm\wordpress
flux -n wordpress get helmreleases
kubectl -n wordpress get pods,pvc,ingress,certificate
```

## Démo "persistance StatefulSet"

```powershell
# 1. Se connecter à https://wordpress.poc.devops-hc.dev/wp-admin
#    (user: admin, pwd: voir secret.yaml)
# 2. Publier un article, uploader une image
# 3. Tuer le pod WordPress
kubectl -n wordpress delete pod wordpress-0

# 4. Le STS recrée le pod, réattache le PVC wp-content
kubectl -n wordpress get pods -w

# 5. Vérifier que l'article + l'image sont toujours là

# Bonus : tuer MariaDB
kubectl -n wordpress delete pod wordpress-mariadb-0
# → données DB préservées via son PVC
```

## Identifiants par défaut

| Compte | Valeur |
|---|---|
| WordPress admin user | `admin` |
| WordPress admin pwd | `ChangeMe123!` (cf. [secret.yaml](secret.yaml)) |
| WordPress admin email | `admin@poc.devops-hc.dev` |
| MariaDB database | `bitnami_wordpress` |
| MariaDB user | `bn_wordpress` |

⚠️ **Changer impérativement les mots de passe** dans [secret.yaml](secret.yaml) avant tout usage non-démo, idéalement via `SealedSecret` ou `ExternalSecret`.
