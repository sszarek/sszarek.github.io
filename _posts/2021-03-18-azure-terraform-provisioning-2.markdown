---
layout: post
title: "Provisioning Azure resources with Terraform - Part 2 - Backends"
date: 2021-03-18 22:01:19 +0100
tags:
  - Azure 
  - IaC 
  - Terraform
author: "Stefan Szarek"
---
## Intro
In the first article about Terraform I have focused on basics required to start working with the tool. Today I would like to tell you about feature called backends which is essentially way of storing state file.

## Local backend
By default Terraform is using local backend which is simply writing state file to local file system. This provider does not require providing any additional configuration. But for the sake of learning, lets try to change the name of the file used for storing the state. Add following block to main.tf file which we have created in previous article.

{% highlight hcl %}
terraform {
  backend "local" {
    path = "my-state-file.tfstate"
  }
}
{% endhighlight %}

The above code instructs Terraform to configure local backend in such way that it stores state file in *my-state-file.tfstate*. Now if you would try running terraform plan you would get information about need of running terraform init first to initialize the backend. In fact terraform init needs to be called after all backend configuration changes.

If you have not done it yet - run terraform init now. You should see information as on screenshot below.

![Initializing Backend](/assets/azure-terraform-provisioning-2/backend-init.png)

Here Terraform detected that configuration of local backend changed (name of the state file was set to *my-state-file.tfstate*), now we need to make a decision whether we would like to copy existing state to the new backend - we will choose "yes". This option will also be available in more advanced scenarios - e.g. when migrating from local to remote state - I will show you that later on.
Now that we have discussed basic configuration of the local backend and how to initialize it, lets talk about feature which is called state locking.

## Locking
State locking is a feature exposed by some of the backends. When running command which is modifying the state (apply, destroy), Terraform locks the state file. The lock is held until command finishes its execution - during that time, no other Terraform process can modify the state file. State locking is essentially way of protecting consistency of the state file.

Lets see how it works in practice. Open two console windows and navigate to directory with Terraform scripts we have created in the previous post. When done, run terraform apply command in both of them but do not confirm. You should get output similar to one on the screenshot below.

![Initializing Backend](/assets/azure-terraform-provisioning-2/state-locking.png)

## Lineage and serial number
Lets take a loo k at the Terraform state similar to one that would be created after running scripts from previous article. It will look more or less like that:

{% highlight json %}
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
{% endhighlight %}

Last time I told you about how this file maps resources defined in script with physical ones, this time I would like to focus on aspects related to state consistency:
* Lineage is unique identifier assigned when creating the state file. If I would delete my local state file and recreate it, Terraform would generate new value. This allows Terraform to determine if two states were generated in the same moment of time.
* Serial is monotonically increasing value. It gets increased whenever you run operation which is changing the state - apply or destroy. This allows Terraform to determine which state file was modified most recently.

As you probably suspect these values are used for determining if Terraform can safely update remote state stored in backend. Having such mechanism is required 