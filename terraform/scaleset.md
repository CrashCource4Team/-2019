# Create an Azure virtual machine scale set with a custom image using Terraform

### Create the directory structure
```mkdir vmss```

```cd vmss```

### Create the variables definitions file
Create a file named variables.tf:

```code variables.tf```

Paste the following code into the editor:
```
variable "location" {
 description = "The location where resources will be created"
}

variable "tags" {
 description = "A map of the tags to use for the resources that are deployed"
 type        = "map"

 default = {
   environment = "codelab"
 }
}

variable "resource_group_name" {
 description = "The name of the resource group in which the resources will be created"
 default     = "myResourceGroup"
}
```

### Create the output definitions file

Create a file named output.tf:

```code output.tf```

Paste the following code into the editor to expose the fully qualified domain name (FQDN) for the virtual machines:

```
output "vmss_public_ip" {
     value = azurerm_public_ip.vmss.fqdn
 }
 ```

 ### Define the network infrastructure in a template

 In this section, you create the following network infrastructure in a new Azure resource group:

    One virtual network (VNET) with the address space of 10.0.0.0/16
    One subnet with the address space of 10.0.2.0/24
    Two public IP addresses. One used by the virtual machine scale set load balancer, the other used to connect to the SSH jumpbox.

Create a file named vmss.tf to describe the virtual machine scale set infrastructure.

```code vmss.tf```

Paste the following code to the end of the file to expose the fully qualified domain name (FQDN) for the virtual machines:

```
resource "azurerm_resource_group" "vmss" {
 name     = var.resource_group_name
 location = var.location
 tags     = var.tags
}

resource "random_string" "fqdn" {
 length  = 6
 special = false
 upper   = false
 number  = false
}

resource "azurerm_virtual_network" "vmss" {
 name                = "vmss-vnet"
 address_space       = ["10.0.0.0/16"]
 location            = var.location
 resource_group_name = azurerm_resource_group.vmss.name
 tags                = var.tags
}

resource "azurerm_subnet" "vmss" {
 name                 = "vmss-subnet"
 resource_group_name  = azurerm_resource_group.vmss.name
 virtual_network_name = azurerm_virtual_network.vmss.name
 address_prefix       = "10.0.2.0/24"
}

resource "azurerm_public_ip" "vmss" {
 name                         = "vmss-public-ip"
 location                     = var.location
 resource_group_name          = azurerm_resource_group.vmss.name
 allocation_method = "Static"
 domain_name_label            = random_string.fqdn.result
 tags                         = var.tags
}
```

### Provision the network infrastructure

Initialize Terraform:

```terraform init```

Run the following command to deploy the defined infrastructure in Azure.

```terraform apply```

Terraform prints the output as defined in the output.tf file.

### Add a virtual machine scale set with a custom image

Open the vmss.tf configuration file:

```code vmss.tf```
Go to the end of the file and paste the following code:

```
resource "azurerm_lb" "vmss" {
 name                = "vmss-lb"
 location            = var.location
 resource_group_name = azurerm_resource_group.vmss.name

 frontend_ip_configuration {
   name                 = "PublicIPAddress"
   public_ip_address_id = azurerm_public_ip.vmss.id
 }

 tags = var.tags
}

resource "azurerm_lb_backend_address_pool" "bpepool" {
 resource_group_name = azurerm_resource_group.vmss.name
 loadbalancer_id     = azurerm_lb.vmss.id
 name                = "BackEndAddressPool"
}

resource "azurerm_lb_probe" "vmss" {
 resource_group_name = azurerm_resource_group.vmss.name
 loadbalancer_id     = azurerm_lb.vmss.id
 name                = "ssh-running-probe"
 port                = var.application_port
}

resource "azurerm_lb_rule" "lbnatrule" {
   resource_group_name            = azurerm_resource_group.vmss.name
   loadbalancer_id                = azurerm_lb.vmss.id
   name                           = "http"
   protocol                       = "Tcp"
   frontend_port                  = var.application_port
   backend_port                   = var.application_port
   backend_address_pool_id        = azurerm_lb_backend_address_pool.bpepool.id
   frontend_ip_configuration_name = "PublicIPAddress"
   probe_id                       = azurerm_lb_probe.vmss.id
}

data "azurerm_image" "dataimage" {
    name = "Image_name"
    resource_group_name = "RG_name"
}

resource "azurerm_virtual_machine_scale_set" "vmss" {
 name                = "vmscaleset"
 location            = var.location
 resource_group_name = azurerm_resource_group.vmss.name
 upgrade_policy_mode = "Manual"

 sku {
   name     = "Standard_DS1_v2"
   tier     = "Standard"
   capacity = 2
 }

 storage_profile_image_reference {
   id = "/subscriptions/Subscription_id/resourceGroups/RG_name/providers/Microsoft.Compute/images/Image_name"
 }

 storage_profile_os_disk {
   name              = ""
   caching           = "ReadWrite"
   create_option     = "FromImage"
   managed_disk_type = "Standard_LRS"
 }

 storage_profile_data_disk {
   lun          = 0
   caching        = "ReadWrite"
   create_option  = "Empty"
   disk_size_gb   = 10
 }

 os_profile {
   computer_name_prefix = "vmlab"
   admin_username       = var.admin_user
   admin_password       = var.admin_password
   custom_data          = file("web.conf")
 }

 os_profile_linux_config {
   disable_password_authentication = false
 }

 network_profile {
   name    = "terraformnetworkprofile"
   primary = true

   ip_configuration {
     name                                   = "IPConfiguration"
     subnet_id                              = azurerm_subnet.vmss.id
     load_balancer_backend_address_pool_ids = [azurerm_lb_backend_address_pool.bpepool.id]
     primary = true
   }
 }

 tags = var.tags
}
```

### Create a file named web.conf to serve as the cloud-init configuration for the virtual machines that are part of the scale set.

```code web.conf```

Paste the following code into the editor:

```
#cloud-config
packages:
 - nginx
  ```

Open the variables.tf configuration file

```code variables.tf```

Customize the deployment by pasting the following code to the end of the file:

```
variable "application_port" {
   description = "The port that you want to expose to the external load balancer"
   default     = 80
}

variable "admin_user" {
   description = "User name to use as the admin account on the VMs that will be part of the VM Scale Set"
   default     = "azureuser"
}

variable "admin_password" {
   description = "Default password for admin account"
}
```

Create a Terraform plan to visualize the virtual machine scale set deployment. (You need to specify a password of your choosing, as well as the location for your resources.):

```terraform plan```

Deploy the new resources in Azure:

```terraform apply```