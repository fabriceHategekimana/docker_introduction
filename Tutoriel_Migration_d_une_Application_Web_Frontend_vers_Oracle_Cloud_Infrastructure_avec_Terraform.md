# Tutoriel : Migration d'une Application Web Frontend vers Oracle Cloud Infrastructure (OCI) avec Terraform

Bienvenue dans ce tutoriel guidé qui vous accompagnera pas à pas dans la migration de votre application web frontend locale vers Oracle Cloud Infrastructure (OCI) en utilisant Terraform. Ce guide est conçu pour les débutants en OCI et en Infrastructure as Code (IaC) avec Terraform.

## Table des Matières

1.  Introduction
2.  Prérequis
3.  Configuration Initiale d'Oracle Cloud Infrastructure (OCI)
    *   Création d'un Compte OCI et d'un Compartiment
    *   Génération des Clés API et Configuration de l'Utilisateur
4.  Installation et Configuration de Terraform
5.  Préparation de l'Application Frontend
6.  Écriture du Code Terraform
    *   Configuration du Fournisseur OCI
    *   Création du Bucket Object Storage pour l'Hébergement Statique
    *   Déploiement de l'API Gateway OCI
    *   Définition des Routes de l'API Gateway
7.  Déploiement de l'Application avec Terraform
8.  Test et Validation
9.  Nettoyage des Ressources
10. Conclusion

## 1. Introduction

Ce tutoriel a pour objectif de vous montrer comment automatiser le déploiement d'une application web frontend statique sur OCI. Nous utiliserons Terraform pour définir et provisionner l'infrastructure nécessaire, notamment un bucket Object Storage pour héberger les fichiers de votre application et une API Gateway pour exposer votre site de manière sécurisée et performante. L'approche Infrastructure as Code (IaC) garantit la reproductibilité et la gestion simplifiée de votre infrastructure.

## 2. Prérequis

Avant de commencer, assurez-vous de disposer des éléments suivants :

*   **Une application web frontend statique** : Il peut s'agir d'une application développée avec des frameworks comme React, Vue.js, Angular, ou simplement un site HTML/CSS/JavaScript pur. Assurez-vous d'avoir les fichiers de build prêts (généralement dans un dossier `dist` ou `build`).
*   **Accès à un compte Oracle Cloud Infrastructure (OCI)** : Un compte Free Tier est suffisant pour ce tutoriel. Si vous n'en avez pas, vous pouvez en créer un sur le site d'Oracle.
*   **Connaissances de base en ligne de commande** : Vous devrez exécuter des commandes dans un terminal.
*   **Un éditeur de texte** : Pour écrire le code Terraform (VS Code, Sublime Text, etc.).

## 3. Configuration Initiale d'Oracle Cloud Infrastructure (OCI)

Cette section vous guidera à travers les étapes essentielles pour préparer votre environnement OCI.

### Création d'un Compte OCI et d'un Compartiment

Si vous n'avez pas encore de compte OCI, inscrivez-vous pour un compte Free Tier. Une fois connecté à la console OCI :

1.  **Créez un Compartiment** : Les compartiments sont des conteneurs logiques pour organiser et isoler vos ressources OCI. C'est une bonne pratique de créer un compartiment dédié pour ce projet.
    *   Dans la console OCI, naviguez vers **Identité et Sécurité** > **Compartiments**.
    *   Cliquez sur **Créer un compartiment**.
    *   Donnez-lui un **Nom** (par exemple, `frontend-app-compartment`) et une **Description**.
    *   Cliquez sur **Créer un compartiment**.

### Génération des Clés API et Configuration de l'Utilisateur

Terraform a besoin de s'authentifier auprès d'OCI pour gérer vos ressources. Nous utiliserons des clés API pour cela.

1.  **Générez une paire de clés API (publique/privée)** sur votre machine locale. Ouvrez un terminal et exécutez :
    ```bash
    mkdir ~/.oci
    openssl genrsa -out ~/.oci/oci_api_key.pem 2048
    chmod go-rwx ~/.oci/oci_api_key.pem
    openssl rsa -pubout -in ~/.oci/oci_api_key.pem -out ~/.oci/oci_api_key_public.pem
    ```
    Cela créera deux fichiers : `oci_api_key.pem` (clé privée) et `oci_api_key_public.pem` (clé publique) dans le dossier `~/.oci`.

2.  **Ajoutez la clé publique à votre utilisateur OCI** :
    *   Dans la console OCI, cliquez sur l'icône **Profil** (en haut à droite) > **Paramètres utilisateur**.
    *   Sous **Ressources**, cliquez sur **Clés API**.
    *   Cliquez sur **Ajouter une clé API**.
    *   Sélectionnez **Coller la clé publique** et collez le contenu du fichier `~/.oci/oci_api_key_public.pem`.
    *   Cliquez sur **Ajouter**.

3.  **Récupérez les informations d'identification OCI** : Vous aurez besoin des OCID suivants pour configurer Terraform :
    *   **Tenancy OCID** : Disponible dans les **Paramètres utilisateur** (sous **Détails de la location**).
    *   **User OCID** : Disponible dans les **Paramètres utilisateur** (sous **Informations utilisateur**).
    *   **Compartment OCID** : Disponible dans les détails du compartiment que vous avez créé.
    *   **Région** : La région OCI où vous souhaitez déployer vos ressources (par exemple, `eu-marseille-1`).

## 4. Installation et Configuration de Terraform

### Installation de Terraform

Suivez les instructions officielles de HashiCorp pour installer Terraform sur votre système d'exploitation :
[https://developer.hashicorp.com/terraform/downloads](https://developer.hashicorp.com/terraform/downloads)

Après l'installation, vérifiez que Terraform est correctement installé en exécutant :
```bash
terraform -v
```

### Configuration du Fichier `~/.oci/config`

Créez ou modifiez le fichier de configuration OCI à l'emplacement `~/.oci/config` avec les informations que vous avez récupérées. Remplacez les valeurs par les vôtres :

```ini
[DEFAULT]
user = <votre_user_ocid>
fingerprint = <empreinte_de_votre_cle_api_publique>
tenancy = <votre_tenancy_ocid>
region = <votre_region_oci>
key_file = ~/.oci/oci_api_key.pem
```

Pour obtenir l'empreinte (`fingerprint`) de votre clé API publique, vous pouvez la trouver dans la console OCI, sous **Paramètres utilisateur** > **Clés API**, ou l'obtenir via la commande :
```bash
openssl rsa -pubin -in ~/.oci/oci_api_key_public.pem -modulus -noout | openssl md5
```

## 5. Préparation de l'Application Frontend

Assurez-vous que votre application frontend est prête pour le déploiement. Cela signifie généralement que vous avez exécuté la commande de build de votre framework (par exemple, `npm run build` pour React/Vue/Angular) et que tous les fichiers statiques sont compilés dans un répertoire de sortie (souvent `dist` ou `build`).

Pour ce tutoriel, nous supposerons que vos fichiers de build se trouvent dans un dossier nommé `dist` à la racine de votre projet Terraform.

## 6. Écriture du Code Terraform

Créez un nouveau répertoire pour votre projet Terraform (par exemple, `oci-frontend-app`) et naviguez-y. Nous allons créer plusieurs fichiers `.tf`.

### `provider.tf` : Configuration du Fournisseur OCI

Ce fichier indique à Terraform d'utiliser le fournisseur OCI et de se baser sur votre fichier de configuration OCI.

```terraform
# provider.tf
provider "oci" {
  tenancy_ocid     = var.tenancy_ocid
  user_ocid        = var.user_ocid
  fingerprint      = var.fingerprint
  private_key_path = var.private_key_path
  region           = var.region
}
```

### `variables.tf` : Définition des Variables

Ce fichier contiendra les variables nécessaires, y compris les OCID et le chemin de la clé privée.

```terraform
# variables.tf
variable "tenancy_ocid" {
  description = "The OCID of the tenancy."
  type        = string
}

variable "user_ocid" {
  description = "The OCID of the user."
  type        = string
}

variable "fingerprint" {
  description = "The fingerprint of the API key."
  type        = string
}

variable "private_key_path" {
  description = "The path to the private key file."
  type        = string
}

variable "region" {
  description = "The OCI region."
  type        = string
}

variable "compartment_ocid" {
  description = "The OCID of the compartment where resources will be created."
  type        = string
}

variable "app_name" {
  description = "Name of the frontend application."
  type        = string
  default     = "my-frontend-app"
}

variable "source_path" {
  description = "Path to the compiled frontend application files (e.g., ./dist)."
  type        = string
  default     = "./dist"
}
```

### `main.tf` : Ressources Principales (Bucket Object Storage et API Gateway)

Ce fichier définira le bucket Object Storage et l'API Gateway.

```terraform
# main.tf

# --- Object Storage Bucket ---
resource "oci_objectstorage_bucket" "frontend_bucket" {
  compartment_id = var.compartment_ocid
  name           = "${var.app_name}-bucket"
  namespace      = data.oci_objectstorage_namespace.this.namespace
  access_type    = "ObjectRead"
  # Pour l'hébergement de site statique, le bucket doit être public ou accessible via PAR
  # Nous allons le rendre public pour simplifier l'accès via l'API Gateway
  # Note: Pour une sécurité accrue, utilisez des PARs ou des politiques IAM plus restrictives
}

data "oci_objectstorage_namespace" "this" {
  compartment_id = var.tenancy_ocid
}

# Upload des fichiers de l'application
resource "oci_objectstorage_object" "app_files" {
  for_each = fileset(var.source_path, "**/*")

  bucket_name = oci_objectstorage_bucket.frontend_bucket.name
  namespace   = data.oci_objectstorage_namespace.this.namespace
  name        = each.value
  content     = file("${var.source_path}/${each.value}")
  # Définir le content_type pour que le navigateur interprète correctement les fichiers
  content_type = lookup(local.mime_types, regex(".*\\.(.*)$", each.value)[0], "application/octet-stream")
}

locals {
  mime_types = {
    "html" = "text/html"
    "css"  = "text/css"
    "js"   = "application/javascript"
    "json" = "application/json"
    "png"  = "image/png"
    "jpg"  = "image/jpeg"
    "jpeg" = "image/jpeg"
    "gif"  = "image/gif"
    "svg"  = "image/svg+xml"
    "ico"  = "image/x-icon"
  }
}

# --- API Gateway ---
# Nécessite un VCN et un Subnet Public. Pour cet exercice, nous allons supposer qu'ils existent.
# Dans un cas réel, vous les définiriez aussi avec Terraform.

# Récupérer un VCN existant (à adapter à votre environnement)
data "oci_core_vcns" "this" {
  compartment_id = var.compartment_ocid
  display_name   = "<Nom_de_votre_VCN>" # Remplacez par le nom de votre VCN
}

# Récupérer un Subnet Public existant (à adapter à votre environnement)
data "oci_core_subnets" "this" {
  compartment_id = var.compartment_ocid
  vcn_id         = data.oci_core_vcns.this.vcns[0].id
  display_name   = "<Nom_de_votre_Subnet_Public>" # Remplacez par le nom de votre Subnet Public
  state          = "AVAILABLE"
}

resource "oci_apigateway_gateway" "frontend_api_gateway" {
  compartment_id = var.compartment_ocid
  endpoint_type  = "PUBLIC"
  subnet_id      = data.oci_core_subnets.this.subnets[0].id
  display_name   = "${var.app_name}-gateway"
}

resource "oci_apigateway_deployment" "frontend_deployment" {
  compartment_id = var.compartment_ocid
  gateway_id     = oci_apigateway_gateway.frontend_api_gateway.id
  path_prefix    = "/"
  display_name   = "${var.app_name}-deployment"

  specification {
    request_policies {
      cors {
        allowed_origins = ["*"]
        allowed_methods = ["GET"]
        allowed_headers = ["*"]
        expose_headers  = ["*"]
        max_age_in_seconds = 300
      }
    }

    routes {
      path = "/{path*}"
      methods = ["GET"]
      backend {
        type = "OBJECT_STORAGE"
        # L'URL du bucket Object Storage pour l'hébergement statique
        # Note: L'API Gateway gérera la résolution des fichiers dans le bucket
        url = "https://${data.oci_objectstorage_namespace.this.namespace}.objectstorage.${var.region}.oci.customer-oci.com/n/${data.oci_objectstorage_namespace.this.namespace}/b/${oci_objectstorage_bucket.frontend_bucket.name}/o/"
      }
    }
  }
}
```

### `outputs.tf` : Sorties Utiles

Ce fichier affichera l'URL de votre application après le déploiement.

```terraform
# outputs.tf
output "app_url" {
  description = "The URL of the deployed frontend application."
  value       = oci_apigateway_deployment.frontend_deployment.endpoint
}
```

## 7. Déploiement de l'Application avec Terraform

Maintenant que votre code Terraform est prêt, vous pouvez déployer votre application.

1.  **Initialisez Terraform** : Naviguez vers le répertoire de votre projet Terraform et exécutez :
    ```bash
    terraform init
    ```
    Cette commande télécharge le fournisseur OCI et initialise le répertoire de travail.

2.  **Planifiez le déploiement** : Vérifiez les changements que Terraform va effectuer sans les appliquer réellement :
    ```bash
    terraform plan -var 'tenancy_ocid=<votre_tenancy_ocid>' \
                   -var 'user_ocid=<votre_user_ocid>' \
                   -var 'fingerprint=<votre_fingerprint>' \
                   -var 'private_key_path=~/.oci/oci_api_key.pem' \
                   -var 'region=<votre_region_oci>' \
                   -var 'compartment_ocid=<votre_compartment_ocid>'
    ```
    Remplacez les placeholders par vos propres valeurs. Vous pouvez également créer un fichier `terraform.tfvars` pour stocker ces variables et éviter de les taper à chaque fois.

    Exemple de `terraform.tfvars` :
    ```
    tenancy_ocid     = "ocid1.tenancy.oc1..xxxx"
    user_ocid        = "ocid1.user.oc1..xxxx"
    fingerprint      = "xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx:xx"
    private_key_path = "/home/ubuntu/.oci/oci_api_key.pem"
    region           = "eu-marseille-1"
    compartment_ocid = "ocid1.compartment.oc1..xxxx"
    ```
    Si vous utilisez `terraform.tfvars`, la commande `plan` devient plus simple :
    ```bash
    terraform plan
    ```

3.  **Appliquez le déploiement** : Si le plan vous convient, appliquez les changements :
    ```bash
    terraform apply
    ```
    Terraform vous demandera de confirmer en tapant `yes`. Une fois l'opération terminée, l'URL de votre application sera affichée dans la section `Outputs`.

## 8. Test et Validation

Copiez l'URL fournie par Terraform dans votre navigateur web. Vous devriez voir votre application frontend s'afficher. Si ce n'est pas le cas, vérifiez les points suivants :

*   **Fichiers de build** : Assurez-vous que le dossier `dist` contient bien tous les fichiers nécessaires et que `index.html` est à la racine.
*   **Politiques de sécurité OCI** : Vérifiez les règles de sécurité de votre VCN et de votre Subnet pour l'API Gateway (Groupes de Sécurité Réseau ou Listes de Sécurité).
*   **Configuration de l'API Gateway** : Assurez-vous que les routes sont correctement définies et que le backend pointe vers le bon bucket Object Storage.

## 9. Nettoyage des Ressources

Pour éviter des coûts inutiles, il est important de nettoyer les ressources OCI après avoir terminé l'exercice.

1.  **Détruisez les ressources Terraform** : Naviguez vers le répertoire de votre projet Terraform et exécutez :
    ```bash
    terraform destroy
    ```
    Terraform vous demandera de confirmer en tapant `yes`. Cette commande supprimera toutes les ressources OCI que Terraform a créées.

## 10. Conclusion

Félicitations ! Vous avez réussi à migrer votre application web frontend locale vers Oracle Cloud Infrastructure en utilisant Terraform. Vous avez appris à :

*   Configurer votre environnement OCI pour l'IaC.
*   Installer et utiliser Terraform.
*   Définir des ressources OCI (Object Storage, API Gateway) avec Terraform.
*   Déployer et nettoyer votre infrastructure.

Cette approche vous offre une base solide pour gérer vos déploiements cloud de manière automatisée et reproductible. N'hésitez pas à explorer davantage les capacités de Terraform et d'OCI pour des architectures plus complexes.

---

## Références

*   [1] [Set Up OCI Object Storage and Oracle API Gateway for Static Website Hosting](https://docs.oracle.com/en/learn/oci-api-gateway-web-hosting/index.html) - Oracle Documentation
*   [2] [Install Terraform](https://developer.hashicorp.com/terraform/downloads) - HashiCorp Documentation
*   [3] [oci_objectstorage_bucket](https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/objectstorage_bucket) - Terraform Registry
*   [4] [oci_apigateway_gateway](https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/apigateway_gateway) - Terraform Registry
*   [5] [oci_apigateway_deployment](https://registry.terraform.io/providers/oracle/oci/latest/docs/resources/apigateway_deployment) - Terraform Registry
