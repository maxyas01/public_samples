### 1. Sample terraform module used for Azure VNET peering together with key implementation details
```terraform
#main.tf 
...
module "vnet-peerings" {
  source  = "./modules/vnet-peerer"
  rg_name = data.azurerm_resource_group.sandboxrg.name
  vnets_to_integrate = {
    hub-spoke1 = {
      a = { id = "${module.vnet-hub.id}", name = "${module.vnet-hub.name}", allow_forwarded_traffic = false },
      b = { id = "${module.vnet-spoke1.id}", name = "${module.vnet-spoke1.name}" }
    },
    hub-spoke2 = {
      a = { id = "${module.vnet-hub.id}", name = "${module.vnet-hub.name}" },
      b = { id = "${module.vnet-spoke2.id}", name = "${module.vnet-spoke2.name}", allow_gateway_transit = true }
    }
  }
}
...

#modules/vnet-peerer/main.tf
#use of can() function enables handling of optional paramerers
....
resource "azurerm_virtual_network_peering" "lhs-rhs" {
  for_each            = var.vnets_to_integrate
  name                = "peering_${each.value.a.name}_to_${each.value.b.name}"
  resource_group_name = var.rg_name


  virtual_network_name         = each.value.a.name
  remote_virtual_network_id    = each.value.b.id
  allow_forwarded_traffic      = can(each.value.a.allow_forwarded_traffic) ? each.value.a.allow_forwarded_traffic : true
  allow_virtual_network_access = can(each.value.a.allow_virtual_network_access) ? each.value.a.allow_virtual_network_access : true
  allow_gateway_transit        = can(each.value.a.allow_gateway_transit) ? each.value.a.allow_gateway_transit : false

}
...
```
### 2. Sample terraform module used for Azure private DNS zones together with key implementation details
In this example a number of different private dns zones are required to be created, they also integrate with different VNETs
```terraform
#main.tf 
...
module "private_dns_zones" {
  source  = "./modules/private_dns"
  rg_name = data.azurerm_resource_group.sandboxrg.name

  create_kv_zone     = true
  create_apim_zone   = true
  create_sqlsrv_zone = true
  vnets_to_integrate = [module.vnet-hub.id, module.vnet-spoke1.id, module.vnet-spoke2.id]

}
module "second_set_pdns" {
  source  = "./modules/private_dns"
  rg_name = data.azurerm_resource_group.sandboxrg.name

  create_sa_blob_zone = true
  create_webapp_zone  = true
  vnets_to_integrate  = [module.vnet-hub.id]
}
....

# modules/private_dns/main.tf
....
locals {
#cartesian product is created to enable subsequent use of for_each in the azurerm_private_dns_zone_virtual_network_link resource block
  dns_zone_names = concat(
    [for dns_zone in azurerm_private_dns_zone.kv_dnsz : dns_zone.name],
    [for dns_zone in azurerm_private_dns_zone.apim_dnsz : dns_zone.name],
    [for dns_zone in azurerm_private_dns_zone.web_dnsz : dns_zone.name],
    [for dns_zone in azurerm_private_dns_zone.sqlsrv_dnsz : dns_zone.name],
    [for dns_zone in azurerm_private_dns_zone.sa_blob_dnsz : dns_zone.name]
  )

  dns_vnet_combinations = [
    for dns_zone in local.dns_zone_names : [
      for vnet_id in var.vnets_to_integrate :
      [
        dns_zone,
        split("/", vnet_id)[length(split("/", vnet_id)) - 1], # Extract last segment
        vnet_id,
      ]
    ]
  ]
}
....
```
