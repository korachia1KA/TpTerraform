# TpTerraform

Travaux pratiques et notes de cours sur **Terraform** : de l'installation jusqu'au backend distant S3, en passant par les modules.

## Sommaire

1. [Introduction à Terraform](#1-introduction-à-terraform)
2. [Cas d'utilisation](#2-cas-dutilisation)
3. [Cycle de vie](#3-cycle-de-vie)
4. [Installation et configuration](#4-installation-et-configuration)
5. [Premier projet Terraform](#5-premier-projet-terraform)
6. [Le fichier d'état (state)](#6-le-fichier-détat-state)
7. [Backend distant S3 + DynamoDB](#7-backend-distant-s3--dynamodb)
8. [Bonnes pratiques](#8-bonnes-pratiques)
9. [Les modules](#9-les-modules)
10. [Problèmes courants](#10-problèmes-courants)
11. [Questions d'entretien](#11-questions-dentretien)

---

## 1. Introduction à Terraform

Terraform est un outil open-source d'**Infrastructure as Code (IaC)** développé par HashiCorp. Il permet de définir, provisionner et gérer des infrastructures de manière **déclarative**. Il est compatible avec plusieurs fournisseurs cloud (AWS, Azure, Google Cloud) et peut aussi gérer des infrastructures sur site.

**Pourquoi l'utiliser ?**

- Automatisation complète du provisionnement et de la gestion de l'infrastructure.
- Cohérence et reproductibilité grâce à la gestion sous forme de code.
- Meilleure collaboration entre équipes grâce aux fichiers de configuration partagés.
- Gestion efficace des dépendances entre ressources.

Terraform utilise le langage **HCL** (HashiCorp Configuration Language), lisible et simple à comprendre.

## 2. Cas d'utilisation

- **Gérer toute infrastructure** : cloud (AWS, Azure, GCP), on-premise, hybride.
- **Suivre l'évolution de l'infrastructure** : versionner et tracer les changements.
- **Automatiser les modifications** : les changements sont appliqués sans intervention manuelle.
- **Standardiser les configurations** : les mêmes ressources sont configurées de manière identique.
- **Collaborer efficacement** : partage et réutilisation des configurations entre équipes.

Modèle de fonctionnement :

```
Script Terraform -> Fournisseur (provider) Terraform -> API cible
```

## 3. Cycle de vie

| Étape | Description | Commande |
|-------|-------------|----------|
| **Write** | Définir l'infrastructure dans des fichiers `.tf` | — |
| **Plan** | Prévisualiser les modifications sans les exécuter | `terraform plan` |
| **Apply** | Créer ou modifier l'infrastructure | `terraform apply` |
| **Destroy** | Détruire les ressources quand c'est nécessaire | `terraform destroy` |

```
Écriture -> Planification -> Application -> Suppression
```

## 4. Installation et configuration

### macOS (Homebrew)

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Mise à jour
brew upgrade hashicorp/tap/terraform
```

### Ubuntu / Debian

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update && sudo apt install terraform
```

### Windows

```powershell
choco install terraform
# ou
winget install Hashicorp.Terraform
```

### Vérification

```bash
terraform -version
```

> Page officielle d'installation : <https://developer.hashicorp.com/terraform/install>

## 5. Premier projet Terraform

Un projet Terraform est généralement composé de plusieurs fichiers :

| Fichier | Rôle |
|---------|------|
| `main.tf` | Configuration principale des ressources |
| `variables.tf` | Définition des variables utilisées |
| `outputs.tf` | Définition des sorties |

### Exemple : instance EC2 sur AWS

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "example" {
  ami           = "ami-12345678"   # à remplacer par une AMI valide de votre région
  instance_type = "t2.micro"
}
```

### Commandes essentielles

```bash
terraform init          # initialise le projet (télécharge providers et modules)
terraform fmt           # formate le code (-recursive, -check, -diff)
terraform validate      # vérifie la syntaxe de la configuration
terraform plan          # prévisualise les changements
terraform apply         # applique les changements
terraform destroy       # détruit l'infrastructure
terraform console       # console interactive pour tester des expressions
terraform output        # affiche les sorties
terraform state list    # liste les ressources suivies dans le state
```

## 6. Le fichier d'état (state)

Terraform stocke l'état actuel des ressources dans le fichier `terraform.tfstate`.

**Règles à respecter :**

- Ne jamais stocker le fichier d'état en local ni sur GitHub (il peut contenir des données sensibles).
- Utiliser un **backend distant** (ex. AWS S3 + DynamoDB pour le verrouillage).
- Ne jamais modifier ce fichier manuellement.

Pensez à ajouter ceci à votre `.gitignore` :

```gitignore
.terraform/
*.tfstate
*.tfstate.*
```

## 7. Backend distant S3 + DynamoDB

Le bucket S3 stocke le fichier d'état, et la table DynamoDB gère le **verrouillage** pour éviter que deux personnes appliquent des changements en même temps.

### Étape 1 : créer le bucket S3

```bash
aws s3api create-bucket \
  --bucket my-terraform-backend-youtube \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2
```

### Étape 2 : activer le chiffrement

```bash
aws s3api put-bucket-encryption \
  --bucket my-terraform-backend-youtube \
  --server-side-encryption-configuration '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

### Étape 3 : activer le versioning

Permet de conserver l'historique des fichiers d'état.

```bash
aws s3api put-bucket-versioning \
  --bucket my-terraform-backend-youtube \
  --versioning-configuration Status=Enabled
```

### Étape 4 : créer la table DynamoDB de verrouillage

```bash
aws dynamodb create-table \
  --table-name terraform-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-west-2
```

### Étape 5 : configurer le backend dans Terraform

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-backend-youtube"
    key            = "chemin/vers/statefile.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

Puis relancer l'initialisation pour migrer le state :

```bash
terraform init
```

> **Note :** les versions récentes de Terraform proposent aussi un verrouillage natif sur S3 (option `use_lockfile`), qui rend DynamoDB optionnel. Consultez la documentation officielle du backend S3 pour la version que vous utilisez.

## 8. Bonnes pratiques

- Utiliser des **modules** pour organiser le code.
- Séparer `main.tf`, `variables.tf` et `outputs.tf`.
- Versionner les fichiers Terraform avec Git.
- Toujours lancer `terraform plan` avant `terraform apply`.
- Utiliser un backend distant avec verrouillage.

## 9. Les modules

Les modules rendent le code Terraform **réutilisable** et **modulaire**.

**Types de modules :**

- **Modules publics** : disponibles sur le [Terraform Registry](https://registry.terraform.io/).
- **Modules personnalisés** : créés pour des besoins spécifiques.

Exemple avec un module du Registry pour créer une instance EC2 :

```hcl
module "ec2_instance" {
  source        = "terraform-aws-modules/ec2-instance/aws"
  instance_type = "t2.micro"
  ami           = "ami-12345678"
}
```

## 10. Problèmes courants

1. Le fichier d'état est la **source unique de vérité**.
2. Les modifications manuelles faites directement dans le cloud ne sont pas détectées automatiquement (dérive de configuration).
3. L'intégration avec les outils GitOps (FluxCD, ArgoCD) n'est pas simple.
4. La complexité augmente avec les grosses infrastructures.
5. Terraform est peu adapté comme outil de gestion de configuration.

## 11. Questions d'entretien

1. Qu'est-ce qu'un module dans Terraform ?
2. Comment configurer Terraform ?
3. Quel est le rôle de S3 et DynamoDB dans Terraform ?
4. Citez quelques commandes essentielles de Terraform.
