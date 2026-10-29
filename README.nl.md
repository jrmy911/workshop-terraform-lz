# Terraform-workshop op Azure

**Doel:** kennismaken met resources, verwijzingen, variabelen, `.tfvars`-bestanden, outputs en de cyclus `plan → apply → destroy`.

## Voorbereiding

- Een publieke SSH-sleutel: `~/.ssh/terraform_workshop.pub` (Windows: `$HOME/.ssh/terraform_workshop.pub`). Overschrijf geen bestaande sleutel en deel nooit de privésleutel.
- **Bash:** maak de map met `mkdir -p ~/.ssh` en daarna de sleutel met `ssh-keygen -t rsa -b 4096 -f ~/.ssh/terraform_workshop` als die nog niet bestaat.
- **PowerShell:** maak de map met `New-Item -ItemType Directory -Force "$HOME/.ssh"` en daarna de sleutel met `ssh-keygen -t rsa -b 4096 -f "$HOME/.ssh/terraform_workshop"` als die nog niet bestaat. Op Windows moet de OpenSSH-client geïnstalleerd zijn om `ssh-keygen` te gebruiken.

Azure CLI (`az`) werkt in beide terminals. Meld je aan en selecteer de sandbox met het blok voor jouw terminal.

**Azure CLI in Bash (Linux/macOS/WSL):**

```bash
az login
az account set --subscription "<sandbox-subscriptie-id>"
az account show --output table
export ARM_SUBSCRIPTION_ID="$(az account show --query id --output tsv)"
```

**Azure CLI in PowerShell (Windows):**

```powershell
az login
az account set --subscription "<sandbox-subscriptie-id>"
az account show --output table
$env:ARM_SUBSCRIPTION_ID = (az account show --query id --output tsv)
```

Vervang `jdoe` door je eigen afkorting in de Azure-namen: **`rg-terraform-workshop-jdoe`** voor John Doe.

## 1. Eerste resource

**Doel:** configureer Terraform en maak een resourcegroep waarin je alle workshopresources verzamelt.

- Maak `versions.tf`: vereis Terraform `>= 1.6, < 2.0` en provider `hashicorp/azurerm` met versie `~> 4.81.0`.
- Maak `providers.tf`: configureer de provider `azurerm` met een leeg `features {}`-blok.
- Maak in `main.tf` een resourcegroep met Terraform-label `workshop`, naam `rg-terraform-workshop-jdoe` en regio `westeurope`.
- Voeg de tags `environment = "acc"` en `workshop = "terraform"` toe. In de volgende stap vervang je de regio en het environment-label door variabelen.

**Eerste commando’s:** `terraform init`, `terraform fmt`, `terraform validate`, `terraform plan`, `terraform apply`.

**Documentatie:** [AzureRM-provider](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs) · [resourcegroep](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/resource_group).

## 2. Variabelen: één configuratie, verschillende waarden

**Doel:** wijzig regio, environment-label en VM-grootte zonder je resourceconfiguratie te herschrijven.

Declareer in `variables.tf`:

| Variabele | Type | Standaardwaarde | Gebruik |
|---|---|---|---|
| `environment` | `string` | Geen; verplicht meegeven | Environment in namen en de tag `environment` van de resourcegroep. |
| `location` | `string` | `"westeurope"` | Regio van de resourcegroep. |
| `vm_size` | `string` | `"Standard_B1s"` | Grootte van de Linux-VM. |

Gebruik nu `var.location` voor de regio van de resourcegroep en `var.environment` voor de tag `environment`. Laat de tag `workshop = "terraform"` ongewijzigd.

Gebruik de meegeleverde bestanden met deze waarden:

| Bestand | `environment` | `location` | `vm_size` |
|---|---|---|---|
| `acc.tfvars` | `"acc"` | `"westeurope"` | `"Standard_B1s"` |
| `prd.tfvars` | `"prd"` | `"westeurope"` | `"Standard_B2s"` |

Gebruik vanaf nu `acc.tfvars` voor je plannen en deployments. Gebruik `prd.tfvars` alleen om later een plan te bekijken, niet om te deployen: de configuratie is geen productieontwerp en gebruikt dezelfde Terraform-state en resourcegroep.

## 3. De omgeving opbouwen

Voeg onderstaande resources stapsgewijs toe aan `main.tf`. Vervang `jdoe` overal door je eigen afkorting. `${var.environment}` betekent dat je de variabele in de naam verwerkt, bijvoorbeeld `vnet-jdoe-acc`.

Plaats alle resources in de bestaande resourcegroep. Gebruik voor regionale resources een verwijzing naar de regio van de resourcegroep. Gebruik ook voor namen en IDs van gekoppelde resources Terraform-verwijzingen in plaats van waarden uit de portal over te nemen.

### 3.1 Netwerk

**Doel:** maak één virtueel netwerk met aparte subnetten voor de VM en voor private endpoints.

- Maak een VNet met naam `vnet-jdoe-${var.environment}` en adresruimte `10.10.0.0/16`.
- Maak daarin subnet `snet-workload` met adresbereik `10.10.1.0/24`. Hier komt de netwerkinterface van de VM.
- Maak daarin subnet `snet-private-endpoints` met adresbereik `10.10.2.0/24`. Hier komt het private endpoint van Key Vault.
- Stel op het private-endpoint-subnet `private_endpoint_network_policies = "Disabled"` in.

**Documentatie:** [VNet](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/virtual_network) · [Subnet](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/subnet).

### 3.2 Netwerkbeveiliging

**Doel:** koppel een netwerkbeveiligingsgroep (NSG) aan het workload-subnet.

- Maak een NSG met naam `nsg-jdoe-${var.environment}`.
- Koppel deze via een subnet-NSG-associatie aan `snet-workload`.
- Voeg geen eigen inkomende regels toe. De standaardregels blokkeren inkomend internetverkeer; open dus ook geen SSH-poort.

**Documentatie:** [NSG](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/network_security_group) · [Subnet-NSG-associatie](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/subnet_network_security_group_association).

### 3.3 Linux-VM

**Doel:** maak een Ubuntu-VM in het workload-subnet, met een eigen Azure-identiteit voor toegang tot Key Vault.

**1. Publiek IP-adres**

- Naam: `pip-jdoe-${var.environment}`.
- SKU: `Standard`.
- Toewijzingsmethode: `Static`.

Dit publieke IP-adres ondersteunt uitgaande verbindingen voor de VM-agent en Azure Run Command. **Voeg geen inkomende SSH-regel toe.** Dit is een vereenvoudiging voor de workshop, geen volledig private VM-opstelling.

**2. Netwerkinterface**

- Naam: `nic-jdoe-${var.environment}`.
- Maak één IP-configuratie met naam `internal`.
- Verbind deze met subnet `snet-workload`.
- Laat Azure het private IP-adres dynamisch toewijzen (`Dynamic`).
- Koppel het publieke IP-adres uit de vorige stap.

**3. Linux-VM**

| Instelling | Vereiste waarde |
|---|---|
| Naam | `vm-jdoe-${var.environment}` |
| VM-grootte | `var.vm_size` |
| Beheerder | `azureuser` |
| Netwerkinterface | De interface die je hierboven hebt gemaakt |
| Publieke SSH-sleutel | Lees `~/.ssh/terraform_workshop.pub` met `file(pathexpand("~/.ssh/terraform_workshop.pub"))` |
| Gebruiker in het SSH-sleutelblok | `azureuser` |
| Image publisher | `Canonical` |
| Image offer | `0001-com-ubuntu-server-jammy` |
| Image SKU | `22_04-lts-gen2` |
| Image version | `latest` |
| OS-disk: caching | `ReadWrite` |
| OS-disk: opslagtype | `Standard_LRS` |
| Managed identity | `SystemAssigned` |

Een **system-assigned managed identity** is een identiteit die Azure aan de VM koppelt. Later geef je die identiteit leesrechten op Key Vault-secrets, zonder een wachtwoord in je code op te nemen. Als `Standard_B1s` niet beschikbaar is in je sandbox, vraag dan aan de begeleider welke grootte je mag gebruiken.

**Documentatie:** [Publiek IP-adres](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/public_ip) · [Netwerkinterface](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/network_interface) · [Linux-VM](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/linux_virtual_machine).

### 3.4 Key Vault

**Doel:** maak een Key Vault zonder publieke netwerktoegang en gebruik Azure-rollen voor toegangsbeheer.

- Voeg een data source `azurerm_client_config` toe met label `current`. Deze leest informatie over je huidige Azure-configuratie; ze maakt geen resource aan.
- Maak de Key Vault met onderstaande instellingen.

| Instelling | Vereiste waarde |
|---|---|
| Naam | `kv-tfw-jdoe-${var.environment}-a1b2`; vervang `jdoe` en `a1b2` door je eigen afkorting en unieke suffix |
| Naamvereisten | Wereldwijd uniek, maximaal 24 tekens |
| Tenant | `tenant_id` uit de data source `azurerm_client_config.current` |
| SKU | `standard` |
| Autorisatie | `rbac_authorization_enabled = true` |
| Publieke netwerktoegang | `public_network_access_enabled = false` |
| Soft-delete-retentie | `soft_delete_retention_days = 7` |
| Purge protection | `purge_protection_enabled = false` |

**Purge protection uitschakelen is alleen bedoeld voor deze tijdelijke sandbox, niet als productieaanbeveling.** Je hoeft voor deze oefening geen secrets aan te maken.

**Documentatie:** [Client-configuratie](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/data-sources/client_config) · [Key Vault](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/key_vault).

### 3.5 Private toegang tot Key Vault

**Doel:** geef Key Vault een privaat IP-adres in je netwerk en zorg dat de VM de Key Vault-naam naar dat adres kan vertalen.

**1. Private DNS-zone**

- Maak de zone `privatelink.vaultcore.azure.net`.
- Maak een VNet-koppeling met naam `link-workshop` tussen deze zone en je VNet. Laat automatische DNS-registratie uitgeschakeld.

**2. Private endpoint**

- Naam: `pe-kv-jdoe-${var.environment}`.
- Plaats het endpoint in `snet-private-endpoints`.
- Voeg een `private_service_connection` toe met naam `psc-key-vault`.
- Verwijs voor `private_connection_resource_id` naar de ID van je Key Vault.
- Gebruik `subresource_names = ["vault"]` en `is_manual_connection = false`.
- Voeg een `private_dns_zone_group` toe met naam `key-vault` en verwijs daarin naar de ID van je private DNS-zone.

**Documentatie:** [Private DNS-zone](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/private_dns_zone) · [VNet-koppeling](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/private_dns_zone_virtual_network_link) · [Private endpoint](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/private_endpoint).

### 3.6 De VM toegang geven tot secrets

**Doel:** geef de identiteit van de VM toestemming om secrets uit deze Key Vault te lezen.

Maak een roltoewijzing met:

- `scope`: de ID van je Key Vault, niet de hele resourcegroep of subscription.
- `role_definition_name`: `Key Vault Secrets User`.
- `principal_id`: de `principal_id` van de system-assigned identity van je VM (`identity[0].principal_id`).
- `skip_service_principal_aad_check = true`, zoals in deze oefening gebruikt voor de nieuw aangemaakte managed identity.

Netwerktoegang en rechten zijn verschillende zaken: het private endpoint zorgt voor de verbinding; de roltoewijzing bepaalt wat de VM mag doen.

**Documentatie:** [Roltoewijzing](https://registry.terraform.io/providers/hashicorp/azurerm/4.81.0/docs/resources/role_assignment).

### Terraform gebruiken tijdens de oefening

In de documentatie beschrijft **Argument Reference** de instellingen die je opgeeft. **Attributes Reference** beschrijft de waarden waarnaar je kunt verwijzen, zoals `id`.

Na elke stap, in **Bash of PowerShell**:

```shell
terraform fmt
terraform validate
terraform plan "-var-file=acc.tfvars"
terraform apply "-var-file=acc.tfvars"
```

Lees het plan voordat je bevestigt. Een verwijzing zoals `azurerm_resource_group.workshop.name` legt ook een afhankelijkheid vast: Terraform bepaalt de aanmaakvolgorde.

## 4. Bekijken en wijzigen

Maak in `outputs.tf` de volgende outputs. Gebruik telkens een verwijzing naar de betreffende resource.

| Output | Waarde |
|---|---|
| `resource_group_name` | Naam van de resourcegroep |
| `vm_name` | Naam van de Linux-VM |
| `vm_private_ip` | Privaat IP-adres van de netwerkinterface |
| `key_vault_uri` | `vault_uri` van de Key Vault |

Lees ze uit met `terraform output`. Bekijk de beheerde resources met `terraform state list`. Bewerk of verwijder het statebestand niet.

Bekijk daarna met `terraform plan "-var-file=prd.tfvars"` hoe andere variabelewaarden het plan veranderen. **Voer dit plan niet uit.**

## 5. Opruimen voor je vertrekt

In **Bash of PowerShell**:

```shell
terraform destroy "-var-file=acc.tfvars"
terraform state list
```

Controleer dat de resourcegroep verdwenen is en de lijst met beheerde resources leeg is. Bewaar bij fouten de state en vraag hulp. Deel geen statebestanden, opgeslagen plannen of privésleutels.

**Onthoud:** een variabele definieert een invoer; een `.tfvars`-bestand geeft die een waarde; een verwijzing verbindt resources; de state koppelt je code aan de Azure-resources.
