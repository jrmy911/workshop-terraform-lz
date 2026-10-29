# Atelier Terraform sur Azure

**Objectif :** découvrir les ressources, références, variables, fichiers `.tfvars`, outputs et le cycle `plan → apply → destroy`.

## Préparation

- Une clé SSH publique : `~/.ssh/terraform_workshop.pub` (Windows : `$HOME/.ssh/terraform_workshop.pub`). Ne remplacez pas une clé existante et ne partagez jamais la clé privée.
- **Bash :** créez le dossier avec `mkdir -p ~/.ssh`, puis la clé avec `ssh-keygen -t rsa -b 4096 -f ~/.ssh/terraform_workshop` si elle n’existe pas.
- **PowerShell :** créez le dossier avec `New-Item -ItemType Directory -Force "$HOME/.ssh"`, puis la clé avec `ssh-keygen -t rsa -b 4096 -f "$HOME/.ssh/terraform_workshop"` si elle n’existe pas. Sous Windows, le client OpenSSH doit être installé pour utiliser `ssh-keygen`.

Azure CLI (`az`) fonctionne dans les deux terminaux. Connectez-vous et sélectionnez la sandbox avec le bloc adapté à votre terminal.

**Azure CLI dans Bash (Linux/macOS/WSL) :**

```bash
az login
az account set --subscription "<id-souscription-sandbox>"
az account show --output table
export ARM_SUBSCRIPTION_ID="$(az account show --query id --output tsv)"
```

**Azure CLI dans PowerShell (Windows) :**

```powershell
az login
az account set --subscription "<id-souscription-sandbox>"
az account show --output table
$env:ARM_SUBSCRIPTION_ID = (az account show --query id --output tsv)
```

Remplacez `jdoe` par votre abréviation dans les noms Azure : **`rg-terraform-workshop-jdoe`** pour John Doe. 

## 1. Première ressource

**Objectif :** configurer Terraform et créer un groupe de ressources pour toutes les ressources de l’atelier.

- Créez `versions.tf` : exigez Terraform `>= 1.6, < 2.0` et le provider `hashicorp/azurerm`, version `~> 4.81.0`.
- Créez `providers.tf` : configurez le provider `azurerm` avec un bloc `features {}` vide.
- Dans `main.tf`, créez un groupe de ressources avec le label Terraform `workshop`, le nom `rg-terraform-workshop-jdoe` et la région `westeurope`.
- Ajoutez les tags `environment = "acc"` et `workshop = "terraform"`. À l’étape suivante, vous remplacerez la région et la valeur d’environnement par des variables.

**Commandes de départ :** `terraform init`, `terraform fmt`, `terraform validate`, `terraform plan`, `terraform apply`.

**Documentation :** [provider AzureRM](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs) · [groupe de ressources](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/resource_group).

## 2. Variables : une configuration, plusieurs valeurs

**Objectif :** modifier la région, l’environnement et la taille de VM sans réécrire la configuration des ressources.

Déclarez dans `variables.tf` :

| Variable | Type | Valeur par défaut | Utilisation |
|---|---|---|---|
| `environment` | `string` | Aucune ; valeur obligatoire | Environnement dans les noms et le tag `environment` du groupe de ressources. |
| `location` | `string` | `"westeurope"` | Région du groupe de ressources. |
| `vm_size` | `string` | `"Standard_B1s"` | Taille de la VM Linux. |

Utilisez maintenant `var.location` pour la région du groupe et `var.environment` pour le tag `environment`. Conservez le tag `workshop = "terraform"`.

Utilisez les fichiers fournis avec ces valeurs :

| Fichier | `environment` | `location` | `vm_size` |
|---|---|---|---|
| `acc.tfvars` | `"acc"` | `"westeurope"` | `"Standard_B1s"` |
| `prd.tfvars` | `"prd"` | `"westeurope"` | `"Standard_B2s"` |

Utilisez désormais `acc.tfvars` pour vos plans et déploiements. Utilisez `prd.tfvars` uniquement pour observer un plan plus tard, sans le déployer : cette configuration n’est pas une architecture de production et utilise le même state Terraform et le même groupe de ressources.

## 3. Construire l’environnement

Ajoutez progressivement les ressources ci-dessous à `main.tf`. Remplacez partout `jdoe` par votre abréviation. `${var.environment}` signifie que vous intégrez la variable au nom, par exemple `vnet-jdoe-acc`.

Placez toutes les ressources dans le groupe existant. Pour les ressources régionales, référencez la région du groupe. Utilisez également des références Terraform pour les noms et IDs des ressources liées, plutôt que de recopier des valeurs depuis le portail.

### 3.1 Réseau

**Objectif :** créer un réseau virtuel avec des subnets distincts pour la VM et les private endpoints.

- Créez un VNet nommé `vnet-jdoe-${var.environment}` avec l’espace d’adressage `10.10.0.0/16`.
- Dans ce VNet, créez le subnet `snet-workload` avec la plage `10.10.1.0/24`. Il accueillera l’interface réseau de la VM.
- Créez également le subnet `snet-private-endpoints` avec la plage `10.10.2.0/24`. Il accueillera le private endpoint du Key Vault.
- Sur le subnet des private endpoints, configurez `private_endpoint_network_policies = "Disabled"`.

**Documentation :** [VNet](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/virtual_network) · [Subnet](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/subnet).

### 3.2 Sécurité réseau

**Objectif :** associer un groupe de sécurité réseau (NSG) au subnet workload.

- Créez un NSG nommé `nsg-jdoe-${var.environment}`.
- Associez-le à `snet-workload` avec une association subnet-NSG.
- N’ajoutez aucune règle entrante personnalisée. Les règles par défaut bloquent le trafic entrant depuis Internet ; n’ouvrez donc pas non plus de port SSH.

**Documentation :** [NSG](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/network_security_group) · [Association subnet-NSG](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/subnet_network_security_group_association).

### 3.3 VM Linux

**Objectif :** créer une VM Ubuntu dans le subnet workload, avec sa propre identité Azure pour accéder au Key Vault.

**1. Adresse IP publique**

- Nom : `pip-jdoe-${var.environment}`.
- SKU : `Standard`.
- Méthode d’allocation : `Static`.

Cette adresse IP publique permet les connexions sortantes pour l’agent de la VM et Azure Run Command. **N’ajoutez pas de règle SSH entrante.** C’est une simplification pour l’atelier, pas une architecture de VM entièrement privée.

**2. Interface réseau**

- Nom : `nic-jdoe-${var.environment}`.
- Créez une configuration IP nommée `internal`.
- Reliez-la au subnet `snet-workload`.
- Laissez Azure attribuer l’adresse IP privée dynamiquement (`Dynamic`).
- Associez l’adresse IP publique créée précédemment.

**3. VM Linux**

| Paramètre | Valeur requise |
|---|---|
| Nom | `vm-jdoe-${var.environment}` |
| Taille de VM | `var.vm_size` |
| Administrateur | `azureuser` |
| Interface réseau | L’interface créée ci-dessus |
| Clé SSH publique | Lisez `~/.ssh/terraform_workshop.pub` avec `file(pathexpand("~/.ssh/terraform_workshop.pub"))` |
| Utilisateur dans le bloc de clé SSH | `azureuser` |
| Image publisher | `Canonical` |
| Image offer | `0001-com-ubuntu-server-jammy` |
| Image SKU | `22_04-lts-gen2` |
| Image version | `latest` |
| Disque OS : caching | `ReadWrite` |
| Disque OS : type de stockage | `Standard_LRS` |
| Identité managée | `SystemAssigned` |

Une **identité managée affectée par le système** est une identité qu’Azure associe à la VM. Vous lui donnerez ensuite le droit de lire les secrets du Key Vault, sans placer de mot de passe dans le code. Si `Standard_B1s` n’est pas disponible dans votre sandbox, demandez au formateur quelle taille utiliser.

**Documentation :** [IP publique](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/public_ip) · [Interface réseau](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/network_interface) · [VM Linux](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/linux_virtual_machine).

### 3.4 Key Vault

**Objectif :** créer un Key Vault sans accès réseau public et utiliser les rôles Azure pour gérer les autorisations.

- Ajoutez une data source `azurerm_client_config` avec le label `current`. Elle lit les informations de votre configuration Azure actuelle ; elle ne crée pas de ressource.
- Créez le Key Vault avec les paramètres suivants.

| Paramètre | Valeur requise |
|---|---|
| Nom | `kv-tfw-jdoe-${var.environment}-a1b2` ; remplacez `jdoe` et `a1b2` par votre abréviation et un suffixe unique |
| Contraintes du nom | Unique mondialement, maximum 24 caractères |
| Tenant | `tenant_id` de la data source `azurerm_client_config.current` |
| SKU | `standard` |
| Autorisation | `rbac_authorization_enabled = true` |
| Accès réseau public | `public_network_access_enabled = false` |
| Rétention après suppression réversible | `soft_delete_retention_days = 7` |
| Protection contre la purge | `purge_protection_enabled = false` |

**Désactiver la protection contre la purge est réservé à cette sandbox temporaire ; ce n’est pas une recommandation pour la production.** Vous ne devez pas créer de secrets pour cet exercice.

**Documentation :** [Configuration client](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/data-sources/client_config) · [Key Vault](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/key_vault).

### 3.5 Accès privé au Key Vault

**Objectif :** donner au Key Vault une adresse IP privée dans votre réseau et permettre à la VM de résoudre son nom vers cette adresse.

**1. Zone DNS privée**

- Créez la zone `privatelink.vaultcore.azure.net`.
- Créez un lien nommé `link-workshop` entre cette zone et votre VNet. Laissez l’enregistrement DNS automatique désactivé.

**2. Private endpoint**

- Nom : `pe-kv-jdoe-${var.environment}`.
- Placez-le dans `snet-private-endpoints`.
- Ajoutez un bloc `private_service_connection` nommé `psc-key-vault`.
- Pour `private_connection_resource_id`, référencez l’ID de votre Key Vault.
- Utilisez `subresource_names = ["vault"]` et `is_manual_connection = false`.
- Ajoutez un bloc `private_dns_zone_group` nommé `key-vault` qui référence l’ID de votre zone DNS privée.

**Documentation :** [Zone DNS privée](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/private_dns_zone) · [Lien au VNet](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/private_dns_zone_virtual_network_link) · [Private endpoint](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/private_endpoint).

### 3.6 Autoriser la VM à lire les secrets

**Objectif :** autoriser l’identité de la VM à lire les secrets de ce Key Vault.

Créez une attribution de rôle avec :

- `scope` : l’ID du Key Vault, pas celui du groupe de ressources ni de la souscription.
- `role_definition_name` : `Key Vault Secrets User`.
- `principal_id` : le `principal_id` de l’identité managée de la VM (`identity[0].principal_id`).
- `skip_service_principal_aad_check = true`, utilisé dans cet exercice pour l’identité managée nouvellement créée.

L’accès réseau et les autorisations sont distincts : le private endpoint fournit la connexion ; l’attribution de rôle détermine ce que la VM peut faire.

**Documentation :** [Attribution de rôle](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/role_assignment).

### Utiliser Terraform pendant l’exercice

Dans la documentation, **Argument Reference** décrit les valeurs à configurer ; **Attributes Reference** décrit les valeurs à référencer, par exemple `id`.

Après chaque étape, dans **Bash ou PowerShell** :

```shell
terraform fmt
terraform validate
terraform plan "-var-file=acc.tfvars"
terraform apply "-var-file=acc.tfvars"
```

Lisez le plan avant de confirmer. Une référence comme `azurerm_resource_group.workshop.name` crée aussi une dépendance : Terraform détermine l’ordre de création.

## 4. Observer et modifier

Dans `outputs.tf`, créez les outputs suivants. Utilisez chaque fois une référence à la ressource concernée.

| Output | Valeur |
|---|---|
| `resource_group_name` | Nom du groupe de ressources |
| `vm_name` | Nom de la VM Linux |
| `vm_private_ip` | Adresse IP privée de l’interface réseau |
| `key_vault_uri` | `vault_uri` du Key Vault |

Consultez-les avec `terraform output`. Regardez les ressources suivies avec `terraform state list`. Ne modifiez et ne supprimez pas le fichier state.

Exécutez ensuite `terraform plan "-var-file=prd.tfvars"` pour observer l’effet d’autres valeurs de variables. **N’appliquez pas ce plan.**

## 5. Nettoyer avant de partir

Dans **Bash ou PowerShell** :

```shell
terraform destroy "-var-file=acc.tfvars"
terraform state list
```

Vérifiez que le groupe de ressources a disparu et que la liste des ressources suivies est vide. En cas d’erreur, conservez le state et demandez de l’aide. Ne partagez pas les fichiers state, les plans enregistrés ou les clés privées.

**À retenir :** une variable déclare une entrée ; un `.tfvars` lui donne une valeur ; une référence relie des ressources ; le state associe votre code aux ressources Azure.
