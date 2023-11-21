<!-- BEGIN_TF_DOCS -->
# Default example

This deploys the module in its simplest form.

```hcl
#----------All Required Provider Section----------- 

terraform {
  required_version = ">= 1.5"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = ">= 3.0, < 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.5.0, < 4.0.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# This ensures we have unique CAF compliant names for our resources.
module "naming" {
  source  = "Azure/naming/azurerm"
  version = "0.3.0"
  suffix  = ["agw"]
}

# This allows us to randomize the region for the resource group.
module "regions" {
  source  = "Azure/regions/azurerm"
  version = ">= 0.3.0"

}

# This allows us to randomize the region for the resource group.
resource "random_integer" "region_index" {
  min = 0
  max = length(module.regions.regions) - 1

}

module "application-gateway" {
  source     = "../../"
  depends_on = [azurerm_log_analytics_workspace.log_analytics_workspace, azurerm_key_vault.keyvault]

  # pre-requisites resources input required for the module

  public_ip_name             = "${module.naming.public_ip.name_unique}-pip"
  resource_group_name        = azurerm_resource_group.rg-group.name
  location                   = azurerm_resource_group.rg-group.location
  vnet_name                  = azurerm_virtual_network.vnet.name
  subnet_name_frontend       = azurerm_subnet.frontend.name
  subnet_name_backend        = azurerm_subnet.backend.name
  log_analytics_workspace_id = azurerm_log_analytics_workspace.log_analytics_workspace.id
  enable_telemetry    = var.enable_telemetry

  # provide Application gateway name 
  app_gateway_name = module.naming.application_gateway.name_unique


  tags = {
    environment = "dev"
    owner       = "application_gateway"
    project     = "AVM"
  }

  sku = {
    # Accpected value for names Standard_v2 and WAF_v2
    name = "WAF_v2"
    # Accpected value for tier Standard_v2 and WAF_v2
    tier = "WAF_v2"
    # Accpected value for capacity 1 to 10 for a V1 SKU, 1 to 100 for a V2 SKU
    capacity = 0 # Set the initial capacity to 0 for autoscaling
  }

  autoscale_configuration = {
    min_capacity = 2
    max_capacity = 15
  }

  # frontend configuration block for the application gateway
  # frontend_ip_configuration_name = "app-gateway-feip"
  private_ip_address = "10.90.1.5" // IP address from backend subnet

  # Backend configuration for the application gateway
  backend_address_pools = [
    {
      name = "Pool1"
      #fqdns        = ["example1.com", "example2.com"]
      ip_addresses = ["10.90.2.4", "10.90.2.5", "10.90.2.6", "10.90.2.7"]
    },
    {
      name = "Pool2"
      # fqdns        = ["contoso.com", "app1.contoso.com"]
      ip_addresses = ["10.90.2.4", "10.90.2.5", "10.90.2.6", "10.90.2.7"]
    },
    {
      name = "Pool3-https"
      #fqdns        = ["example1.com", "example2.com"]
      ip_addresses = ["10.90.2.4", "10.90.2.5", "10.90.2.6", "10.90.2.7"]
    },
    # Add more pools as needed
  ]

  # Http Listerners configuration for the application gateway
  http_listeners = [
    {
      name                   = "http-listener-80"
      frontend_port_name     = null
      protocol               = "Http"
      frontend_ip_assocation = "public"
    },
    {
      name                   = "http-listener2-81"
      frontend_port_name     = null
      protocol               = "Http"
      frontend_ip_assocation = "both"
    },
    {
      name                   = "http-listener-443"
      frontend_port_name     = "frontend-port-443"
      protocol               = "Https"
      frontend_ip_assocation = "public"
      ssl_certificate_name   = "app-gateway-cert"
    }

    # Add more http listeners as needed
  ]

  # Backend http settings configuration for the application gateway

  backend_http_settings = [
    {
      name                  = "port1-80"
      port                  = 80
      protocol              = "Http"
      cookie_based_affinity = "Disabled"
    },
    {
      name                  = "port2-81"
      port                  = 81
      protocol              = "Http"
      cookie_based_affinity = "Disabled"
    },
    {
      name                  = "http-settings-443"
      port                  = 443
      protocol              = "Https"
      cookie_based_affinity = "Disabled"
      # Define other attributes as needed
    }
    # Add more http settings as needed
  ]

  # Routing rules configuration for the backend pool
  request_routing_rules = [
    {
      name                      = "Rule1"
      rule_type                 = "Basic"
      http_listener_name        = null
      backend_address_pool_name = null
      priority                  = 9
    },
    {
      name                      = "Rule2"
      rule_type                 = "Basic" //"PathBasedRouting"
      http_listener_name        = null
      backend_address_pool_name = null
      priority                  = 10
    },
    {
      name                        = "Rule3-443"
      rule_type                   = "Basic" //"PathBasedRouting"
      http_listener_name          = null    //var.http_listeners[1].name
      backend_address_pool_name   = null    //var.backend_address_pools[1].name
      priority                    = 11
      redirect_configuration_name = "app1-redirect-config"
    },
    # Add more rules as needed
  ]


  # SSL Certificate Block
  ssl_certificates = [{
    name = "app-gateway-cert"
    key_vault_secret_id = azurerm_key_vault_certificate.ssl_cert_id.secret_id
  }]


  # HTTP to HTTPS Redirection Configuration for

  redirection_configurations = [
    {
      name                 = "app1-redirect-config"
      redirect_type        = "Permanent"
      target_url           = "https://contoso.com/"
      include_path         = true
      include_query_string = true
    }
    # Add more redirection configurations as needed
  ]

  # waf_policy_name             = "waf-policy" # Name of the exisitng WAF policy
  enable_classic_rule = true //applicable only for WAF_v2 SKU. this will enable WAF standard policy

  waf_configuration = [
    {
      enabled          = true
      firewall_mode    = "Prevention"
      rule_set_type    = "OWASP"
      rule_set_version = "3.1"
    }
  ]

  # redirect_configuration_name    = "app-gateway-rdrcfg"

  # Optional Input
  
  zone_redundant = ["1", "2", "3"] #["1", "2", "3"] # Zone redundancy for the application gateway

  identity_ids = [{
    identity_ids = [
      azurerm_user_assigned_identity.appag_umid.id
  ] }]

  
}

```

<!-- markdownlint-disable MD033 -->
## Requirements

The following requirements are needed by this module:

- <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) (>= 1.5)

- <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) (>= 3.0, < 4.0)

- <a name="requirement_random"></a> [random](#requirement\_random) (>= 3.5.0, < 4.0.0)

## Providers

The following providers are used by this module:

- <a name="provider_azurerm"></a> [azurerm](#provider\_azurerm) (>= 3.0, < 4.0)

- <a name="provider_random"></a> [random](#provider\_random) (>= 3.5.0, < 4.0.0)

## Resources

The following resources are used by this module:

- [azurerm_key_vault.keyvault](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault) (resource)
- [azurerm_key_vault_access_policy.appag_key_vault_access_policy](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_access_policy) (resource)
- [azurerm_key_vault_access_policy.key_vault_default_policy](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_access_policy) (resource)
- [azurerm_key_vault_certificate.ssl_cert_id](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_certificate) (resource)
- [azurerm_log_analytics_workspace.log_analytics_workspace](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/log_analytics_workspace) (resource)
- [azurerm_resource_group.rg-group](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group) (resource)
- [azurerm_subnet.backend](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet) (resource)
- [azurerm_subnet.frontend](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet) (resource)
- [azurerm_subnet.private-ip-test](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet) (resource)
- [azurerm_subnet.workload](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/subnet) (resource)
- [azurerm_user_assigned_identity.appag_umid](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/user_assigned_identity) (resource)
- [azurerm_virtual_network.vnet](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/virtual_network) (resource)
- [random_integer.region_index](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/integer) (resource)
- [azurerm_client_config.current](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/client_config) (data source)

<!-- markdownlint-disable MD013 -->
## Required Inputs

No required inputs.

## Optional Inputs

The following input variables are optional (have default values):

### <a name="input_enable_telemetry"></a> [enable\_telemetry](#input\_enable\_telemetry)

Description: This variable controls whether or not telemetry is enabled for the module.  
For more information see https://aka.ms/avm/telemetryinfo.  
If it is set to false, then no telemetry will be collected.

Type: `bool`

Default: `true`

## Outputs

The following outputs are exported:

### <a name="output_azurerm_key_vault_certificate_secret_id"></a> [azurerm\_key\_vault\_certificate\_secret\_id](#output\_azurerm\_key\_vault\_certificate\_secret\_id)

Description: n/a

### <a name="output_backend_subnet_id"></a> [backend\_subnet\_id](#output\_backend\_subnet\_id)

Description: ID of the Backend Subnet

### <a name="output_backend_subnet_name"></a> [backend\_subnet\_name](#output\_backend\_subnet\_name)

Description: Name of the Backend Subnet

### <a name="output_frontend_subnet_id"></a> [frontend\_subnet\_id](#output\_frontend\_subnet\_id)

Description: ID of the Frontend Subnet

### <a name="output_frontend_subnet_name"></a> [frontend\_subnet\_name](#output\_frontend\_subnet\_name)

Description: Name of the Frontend Subnet

### <a name="output_key_vault_id"></a> [key\_vault\_id](#output\_key\_vault\_id)

Description: ID of the Azure Key Vault

### <a name="output_log_analytics_workspace_id"></a> [log\_analytics\_workspace\_id](#output\_log\_analytics\_workspace\_id)

Description: ID of the Azure Log Analytics Workspace

### <a name="output_log_analytics_workspace_name"></a> [log\_analytics\_workspace\_name](#output\_log\_analytics\_workspace\_name)

Description: Name of the Azure Log Analytics Workspace

### <a name="output_private_ip_test_subnet_id"></a> [private\_ip\_test\_subnet\_id](#output\_private\_ip\_test\_subnet\_id)

Description: ID of the Private IP Test Subnet

### <a name="output_private_ip_test_subnet_name"></a> [private\_ip\_test\_subnet\_name](#output\_private\_ip\_test\_subnet\_name)

Description: Name of the Private IP Test Subnet

### <a name="output_resource_group_id"></a> [resource\_group\_id](#output\_resource\_group\_id)

Description: ID of the Azure Resource Group

### <a name="output_resource_group_name"></a> [resource\_group\_name](#output\_resource\_group\_name)

Description: Name of the Azure Resource Group

### <a name="output_self_signed_certificate_id"></a> [self\_signed\_certificate\_id](#output\_self\_signed\_certificate\_id)

Description: ID of the self-signed SSL certificate in Azure Key Vault

### <a name="output_virtual_network_id"></a> [virtual\_network\_id](#output\_virtual\_network\_id)

Description: ID of the Azure Virtual Network

### <a name="output_virtual_network_name"></a> [virtual\_network\_name](#output\_virtual\_network\_name)

Description: Name of the Azure Virtual Network

### <a name="output_workload_subnet_id"></a> [workload\_subnet\_id](#output\_workload\_subnet\_id)

Description: ID of the Workload Subnet

### <a name="output_workload_subnet_name"></a> [workload\_subnet\_name](#output\_workload\_subnet\_name)

Description: Name of the Workload Subnet

## Modules

The following Modules are called:

### <a name="module_application-gateway"></a> [application-gateway](#module\_application-gateway)

Source: ../../

Version:

### <a name="module_naming"></a> [naming](#module\_naming)

Source: Azure/naming/azurerm

Version: 0.3.0

### <a name="module_regions"></a> [regions](#module\_regions)

Source: Azure/regions/azurerm

Version: >= 0.3.0

<!-- markdownlint-disable-next-line MD041 -->
## Data Collection

The software may collect information about you and your use of the software and send it to Microsoft. Microsoft may use this information to provide services and improve our products and services. You may turn off the telemetry as described in the repository. There are also some features in the software that may enable you and Microsoft to collect data from users of your applications. If you use these features, you must comply with applicable law, including providing appropriate notices to users of your applications together with a copy of Microsoft’s privacy statement. Our privacy statement is located at <https://go.microsoft.com/fwlink/?LinkID=824704>. You can learn more about data collection and use in the help documentation and our privacy statement. Your use of the software operates as your consent to these practices.
<!-- END_TF_DOCS -->