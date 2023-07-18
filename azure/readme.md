### Sample terraform module
Module is used to peer Azure VNETs

```javascript
main.tf 
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


modules/vnet-peerer/main.tf
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
