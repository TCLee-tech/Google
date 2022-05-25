HashiCorp Terraform
https://www.terraform.io/

Terraform configuration files format. JSON recommended.
https://www.terraform.io/language

HashiCorp Terraform Community
https://www.terraform.io/community

Examples of Terraform use with GCP (github)
https://github.com/GoogleCloudPlatform/terraform-google-examples

Terraform Registry
https://registry.terraform.io/
info on provider for terraform {} block.

Custom OS images
 - with additional software installed or configurations applied
https://cloud.google.com/compute/docs/images/create-delete-deprecate-private-images

Packer
tool to create identical VM images for multiple platforms using single source template.
https://www.packer.io/
Packer build images for GCP
https://www.packer.io/plugins/builders/googlecompute

Provisioners
Terraform use them to upload files, run shell scripts, install and trigger software, e.g. config management tools.
https://www.terraform.io/language/resources/provisioners/syntax
Most provisioners require access to remote resource via SSH, expect nested connection block with details on how to connect:
https://www.terraform.io/language/resources/provisioners/connection

Modules can be loaded from local filesystem or remote sources, e.g. Terraform Registry, version control systems, HTTP URLs, cloud storage buckets, Terraform Cloud, Terraform Enterprise private module registries.
Terraform Modules
General info: https://www.terraform.io/language/modules
Modules from Terraform Registry: https://registry.terraform.io/browse/modules

Terraform state
Records relationship(bindings) between the cloud infrastructure objects and resource instances declared in configuration (.tf file).
Default: Stores the Terraform state file (terraform.tfstate) locally in the current working directory.
Recommended: Store remotely on a cloud server for team access.
Backends: Where Terraform's state snapshots are stored.
Some (not all) backends support locking of state.
Terraform can prevent multiple users run Terraform at the same time. When one user is performing an operation, Terraform locks access for all other users for all operations that can write state. Each terraform run begins with most updated state.
Benefits: 
	* Prevent corruption of state data. 
	* Keep sensitive info off local machine. State data is retrieved on demand and only stored in memory.
	* Remote operation of "terraform apply" can take time for large infrastructure. Remote state storage, locking and operation allows local machine shutdown.
https://www.terraform.io/language/state/remote
https://www.terraform.io/language/settings/backends
https://www.terraform.io/language/state/locking

Workspaces
Each terraform configuration is associated with one backend where persistent data (Terraform state) is stored. Each state is in a workspace. Initial workspace called default. Some backends can have multiple workspaces, i.e. multiple states. Effect is multiple instances of configuration.

Bringing existing infrastructure under Terraform’s control involves five main steps:
1. Identify the existing infrastructure to be imported. E.g. docker container.
2. Import the infrastructure into your Terraform state.
3. Write a Terraform configuration that matches that infrastructure.
4. Review the Terraform plan to ensure that the configuration matches the expected state and infrastructure.
5. Apply the configuration to update your Terraform state.
For step 3, can import Terraform state into docker.tf (terraform show -no-color > docker.tf) but must check resources and attributes to merge or remove.
Refer to list of required and optional attributes from provider documentation:
https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/resources/container#links

IaC best practice: Immutable infrastructure
https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure

Terraformer
tool to automate some steps associated with importing infrastructure to Terraform.
generates tf/json + tfstate files from existing infrastructure for all supported objects (reverse Terraform).
https://github.com/GoogleCloudPlatform/terraformer

Hashicorp on Google Cloud Marketplace:
https://console.cloud.google.com/marketplace/browse?q=Hashicorp&utm_source=Hashicorp&utm_medium=qwiklabs&utm_campaign=Qwiklabs%20to%20Marketplace
Hashicorp Learn - get started with cloud provider
https://learn.hashicorp.com/terraform
Terraform Community
https://www.terraform.io/community
Collection of examples for using Terraform with GCP
https://github.com/GoogleCloudPlatform/terraform-google-examples

Automating Infrastructure on Google Cloud with Terraform: Challenge Lab
Topics tested:
- Import existing infrastructure into your Terraform configuration.
- Build and reference your own Terraform modules.
- Add a remote backend to your configuration.
- Use and implement a module from the Terraform Registry.
- Re-provision, destroy, and update infrastructure.
- Test connectivity between the resources you've created.

Notes for the challenge lab.
terraform import command: terraform import [options] ADDRESS ID
ADDRESS refers to a valid resource address.
Each remote object must be bound to only one resource address. Be careful when you import existing object into Terraform manually. If Terraform creates the object, it guarantees this 1:1 mapping.
	- a resource address is made up of 2 parts: [module path] [resource spec]
	- [module path] is module.module_name[module index]
	- module keyword indicates child (non-root) module.
	- [module index] is optional; for selecting an instance when there are many; includes [] brackets.
	- e.g. module.foo[0].module.bar["a"]
	- [resource spec] is resource_type.resource_name[instance index]
	- e.g. aws_instance.web[3]
	- https://www.terraform.io/cli/state/resource-addressing
ID refers to the unique resource ID. For example, for VM instances, it is the instance ID.
For [options], see:
https://www.terraform.io/cli/commands/import#example-import-into-module

terraform init command: terraform init [options]
used to initialize working directory containing Terraform configuration files.
re-running already initialized backend will update working directory to use new backend settings.
Use -migrate-state. This option will copy existing state to the new backend.
For configuration with module blocks, re-running init will not change already installed modules. Will only install sources for any new modules added since last init.
Use -upgrade option to update all modules.