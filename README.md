# VM deployen naar ESXi met Terraform

(Opdracht 1) In deze README staat hoe je met Terraform een Ubuntu VM deployt naar ESXi. Daarna wordt kort uitgelegd hoe je Ansible gebruikt om de VM te configureren.

## Voorwaarden

Voordat je begint, moeten de volgende onderdelen aanwezig zijn:

- Terraform is geïnstalleerd
- Ansible is geïnstalleerd
- Je hebt toegang tot de ESXi-host
- Je hebt een SSH-key voor de Ubuntu VM

## Bestanden

De Terraform-configuratie bestaat uit deze bestanden:

- `main.tf`
- `provider.tf`
- `variables.tf`
- `versions.tf`
- `metadata.yaml`
- `userdata.yaml`

## main.tf

```hcl
resource "esxi_guest" "ubuntu" {
  guest_name = var.vm_name
  disk_store = var.datastore

  numvcpus = 1
  memsize  = 1024

  boot_firmware = "efi"
  guestos       = "ubuntu-64"

  network_interfaces {
    virtual_network = var.network
  }

  boot_disk_type = "thin"
  boot_disk_size = 20

  power = "on"

  extra_config = {
    "guestinfo.metadata"          = base64encode(file("${path.module}/metadata.yaml"))
    "guestinfo.metadata.encoding" = "base64"

    "guestinfo.userdata"          = base64encode(templatefile("${path.module}/userdata.yaml", {
      admin_user = var.esxi_user
      ssh_pubkey = file(var.ssh_pubkey)
    }))
    "guestinfo.userdata.encoding" = "base64"
  }
}

resource "local_file" "ansible_inventory" {
  filename = "${path.module}/inventory.ini"

  content = <<EOF
[ubuntu]
${esxi_guest.ubuntu.ip_address} app_name=demoapp ansible_user=${var.esxi_user}
EOF
}
```

## provider.tf

```hcl
provider "esxi" {
  esxi_hostname = var.esxi_host
  esxi_hostport = 22
  esxi_hostssl  = 443
  esxi_username = var.esxi_user
  esxi_password = var.esxi_password
}
```

## variables.tf

```hcl
variable "esxi_host" {
  type = string
}

variable "esxi_user" {
  type = string
}

variable "esxi_password" {
  type      = string
  sensitive = true
}

variable "datastore" {
  type = string
}

variable "network" {
  type = string
}

variable "vm_name" {
  type    = string
  default = "ubuntu-01"
}

variable "ssh_pubkey" {
  type = string
}
```

## versions.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    esxi = {
      source  = "josenk/esxi"
      version = "~> 1.9"
    }

    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}
```

## Deployment uitvoeren

Voer eerst Terraform init uit:

```bash
terraform init
```

Controleer daarna wat Terraform gaat aanmaken:

```bash
terraform plan
```

Deploy daarna de VM:

```bash
terraform apply
```

Na het uitvoeren van `terraform apply` wordt de Ubuntu VM aangemaakt op ESXi.

## Inventory voor Ansible

Terraform maakt automatisch een `inventory.ini` bestand aan. Hierin staat de Ubuntu VM met de variabele `app_name=demoapp`.

Voorbeeld:

```ini
[ubuntu]
192.168.1.100 app_name=demoapp ansible_user=student
```

## Ansible gebruiken

Met Ansible kun je de VM configureren nadat Terraform de VM heeft aangemaakt.

Test eerst of Ansible verbinding kan maken met de VM:

```bash
ansible all -i inventory.ini -m ping
```

Als dit goed werkt, krijg je een succesvolle ping-response terug.

Daarna kun je een playbook uitvoeren:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

## Voorbeeld van een inventory.ini

```ini
[hosts]
app1 ansible_host=<IP/FQDN server 1>
app2 ansible_host=<IP/FQDN server 2>
```

## Samenvatting

Met Terraform wordt de Ubuntu VM automatisch aangemaakt op ESXi. Terraform maakt ook een Ansible inventory aan. Daarna kan Ansible worden gebruikt om de VM verder te configureren.
