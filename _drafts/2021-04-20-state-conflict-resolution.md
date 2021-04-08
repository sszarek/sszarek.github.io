---
layout: post
title: Provisioning Azure resources with Terraform - Part 3 - Resolving conflicts in state
date: 2021-04-20 22:01:19 +0100
tags:
  - Azure 
  - IaC 
  - Terraform
author: Stefan Szarek
---

## Lineage and serial number
Lets take a look at the Terraform state similar to one that would be created after running scripts from previous article. It will look more or less like that:

```json
{
  "version": 4,
  "terraform_version": "0.14.2",
  "serial": 8,
  "lineage": "0c07f967-e88a-0a90-5c30-b0c001ed8543",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "azurerm_resource_group",
      "name": "rg",
      "provider": "provider[\"registry.terraform.io/hashicorp/azurerm\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "/subscriptions/XXX/resourceGroups/terraform_example_app",
            "location": "centralus",
            "name": "terraform_example_app",
            "tags": null,
            "timeouts": null
          },
          "sensitive_attributes": [],
          "private": "YYY"
        }
      ]
    }
  ]
}
```

Last time I told you about how this file maps resources defined in script with physical ones, this time I would like to focus on aspects related to state consistency:
* Lineage is unique identifier assigned when creating the state file. If I would delete my local state file and recreate it, Terraform would generate new value. This allows Terraform to determine if two states were generated in the same moment of time.
* Serial is monotonically increasing value. It gets increased whenever you run operation which is changing the state - apply or destroy. This allows Terraform to determine which state file was modified most recently.

As you probably suspect these values are used for determining if Terraform can safely update remote state stored in backend. Having such mechanism is required 

