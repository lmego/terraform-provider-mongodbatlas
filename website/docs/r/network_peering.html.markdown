---
layout: "mongodbatlas"
page_title: "MongoDB Atlas: network_peering"
sidebar_current: "docs-mongodbatlas-resource-network-peering"
description: |-
    Provides a Network Peering resource.
---

# mongodbatlas_network_peering

`mongodbatlas_network_peering` provides a Network Peering Connection resource. The resource lets you create, edit and delete network peering connections. The resource requires your Project ID.  Ensure you have first created a Network Container.  See the network_container resource and examples below.

~> **GCP AND AZURE ONLY:** Connect via Peering Only mode is deprecated, so no longer needed.  See [disable Peering Only mode](https://docs.atlas.mongodb.com/reference/faq/connection-changes/#disable-peering-mode) for details and [private_ip_mode](https://www.terraform.io/docs/providers/mongodbatlas/r/private_ip_mode.html) to disable.

~> **AZURE ONLY:** To create the peering request with an Azure VNET, you must grant Atlas the following permissions on the virtual network.
    Microsoft.Network/virtualNetworks/virtualNetworkPeerings/read
    Microsoft.Network/virtualNetworks/virtualNetworkPeerings/write
    Microsoft.Network/virtualNetworks/virtualNetworkPeerings/delete
    Microsoft.Network/virtualNetworks/peer/action
For more information see https://docs.atlas.mongodb.com/security-vpc-peering/

-> **Create a Whitelist:** Ensure you whitelist the private IP ranges of the subnets in which your application is hosted in order to connect to your Atlas cluster.  See the project_ip_whitelist resource.

-> **NOTE:** Groups and projects are synonymous terms. You may find **group_id** in the official documentation.


## Example Usage

### Global configuration for the following examples
```hcl
locals {
  project_id        = <your-project-id>

  # needed for GCP only
  GCP_PROJECT_ID = <your-google-project-id>

  # needed for Azure Only
  AZURE_DIRECTORY_ID = <your-azure-directory-id>
  AZURE_SUBCRIPTION_ID = <Unique identifer of the Azure subscription in which the VNet resides>
  AZURE_RESOURCES_GROUP_NAME = <Name of your Azure resource group>
  AZURE_VNET_NAME = <Name of your Azure VNet>
}
```

### Example with AWS.

```hcl
resource "mongodbatlas_network_container" "test" {
  project_id       = local.project_id
  atlas_cidr_block = "10.8.0.0/21"
  provider_name    = "AWS"
  region_name      = "US_EAST_1"
}

resource "mongodbatlas_network_peering" "test" {
  accepter_region_name   = "us-east-1"
  project_id             = local.project_id
  container_id           = "507f1f77bcf86cd799439011"
  provider_name          = "AWS"
  route_table_cidr_block = "192.168.0.0/24"
  vpc_id                 = "vpc-abc123abc123"
  aws_account_id         = "abc123abc123"
}

# the following assumes an AWS provider is configured  
resource "aws_vpc_peering_connection_accepter" "peer" {
  vpc_peering_connection_id = "${mongodbatlas_network_peering.test.connection_id}"
  auto_accept = true
}

```

### Example with GCP

```hcl

resource "mongodbatlas_network_container" "test" {
  project_id       = "${local.project_id}"
  atlas_cidr_block = "192.168.192.0/18"
  provider_name    = "GCP"
}

resource "mongodbatlas_network_peering" "test" {
  project_id       = "${local.project_id}"
  atlas_cidr_block = "192.168.0.0/18"

  container_id   = "${mongodbatlas_network_container.test.container_id}"
  provider_name  = "GCP"
  gcp_project_id = "${local.GCP_PROJECT_ID}"
  network_name   = "default"
}

data "google_compute_network" "default" {
  name = "default"
}

resource "google_compute_network_peering" "peering" {
  name         = "peering-gcp-terraform-test"
  network      = "${data.google_compute_network.default.self_link}"
  peer_network = "https://www.googleapis.com/compute/v1/projects/${mongodbatlas_network_peering.test.atlas_gcp_project_id}/global/networks/${mongodbatlas_network_peering.test.atlas_vpc_name}"
}

resource "mongodbatlas_cluster" "test" {
  project_id   = "${local.project_id}"
  name         = "terraform-manually-test"
  num_shards   = 1
  disk_size_gb = 5

  replication_factor           = 3
  auto_scaling_disk_gb_enabled = true
  mongo_db_major_version       = "4.2"

  //Provider Settings "block"
  provider_name               = "GCP"
  provider_instance_size_name = "M10"
  provider_region_name        = "US_EAST_4"

  depends_on = ["google_compute_network_peering.peering"]
}

#  Private connection strings are not available w/ GCP until the reciprocal
#  connection changes to available (i.e. when the status attribute changes
#  to AVAILABLE on the 'mongodbatlas_network_peering' resource, which
#  happens when the google_compute_network_peering and and
#  mongodbatlas_network_peering make a reciprocal connection).  Hence
#  since the cluster can be created before this connection completes
#  you may need to run `terraform refresh` to obtain the private connection strings.  

```

### Example with Azure

```hcl

resource "mongodbatlas_network_container" "test" {
  project_id       = "${local.project_id}"
  atlas_cidr_block = "192.168.208.0/21"
  provider_name    = "AZURE"
  region           = "US_EAST_2"
}

resource "mongodbatlas_network_peering" "test" {
  project_id            = "${local.project_id}"
  atlas_cidr_block      = "192.168.0.0/21"
  container_id          = "${mongodbatlas_network_container.test.container_id}"
  provider_name         = "AZURE"
  azure_directory_id    = "${local.AZURE_DIRECTORY_ID}"
  azure_subscription_id = "${local.AZURE_SUBCRIPTION_ID}"
  resource_group_name   = "${local.AZURE_RESOURCES_GROUP_NAME}"
  vnet_name             = "${local.AZURE_VNET_NAME}"
}

resource "mongodbatlas_cluster" "test" {
  project_id = "${local.project_id}"
  name       = "terraform-manually-test"
  num_shards = 1

  replication_factor           = 3
  auto_scaling_disk_gb_enabled = true
  mongo_db_major_version       = "4.2"

  //Provider Settings "block"
  provider_name               = "AZURE"
  provider_disk_type_name     = "P4"
  provider_instance_size_name = "M10"
  provider_region_name        = "US_EAST_2"

  depends_on = ["mongodbatlas_network_peering.test"]
}

```

## Argument Reference

* `project_id` - (Required) The unique ID for the project to create the database user.
* `container_id` - (Required) Unique identifier of the Atlas VPC container for the region. You can create an Atlas VPC container using the Create Container endpoint. You cannot create more than one container per region. To retrieve a list of container IDs, use the Get list of VPC containers endpoint.
* `provider_name` - (Required) Cloud provider for this VPC peering connection. (Possible Values `AWS`, `AZURE`, `GCP`).
* `accepter_region_name` - (Optional | **AWS Required**) Specifies the region where the peer VPC resides. For complete lists of supported regions, see [Amazon Web Services](https://docs.atlas.mongodb.com/reference/amazon-aws/).
* `aws_account_id` - (Optional | **AWS Required**) Account ID of the owner of the peer VPC.
* `route_table_cidr_block` - (Optional | **AWS Required**) Peer VPC CIDR block or subnet.
* `vpc_id` - (Optional | **AWS Required**) Unique identifier of the peer VPC.
* `atlas_cidr_block` - (Required) CIDR block that Atlas uses for your clusters. Atlas uses the specified CIDR block for all other Network Peering connections created in the project. The Atlas CIDR block must be at least a /24 and at most a /21 in one of the following [private networks](https://tools.ietf.org/html/rfc1918.html#section-3).
* `azure_directory_id` - (Optional | **AZURE Required**) Unique identifier for an Azure AD directory.
* `azure_subscription_id` - (Optional | **AZURE Required**) Unique identifer of the Azure subscription in which the VNet resides.
* `resource_group_name` - (Optional | **AZURE Required**) Name of your Azure resource group.
* `vnet_name` - (Optional | **AZURE Required**) Name of your Azure VNet.
* `gcp_project_id` - (Optinal | **GCP Required**) GCP project ID of the owner of the network peer.
* `network_name` - (Optional | **GCP Required**) Name of the network peer to which Atlas connects.

## Attributes Reference

In addition to all arguments above, the following attributes are exported:

* `peer_id` - The Network Peering Container ID.
* `id` -	The Terraform's unique identifier used internally for state management.
* `connection_id` -  Unique identifier for the peering connection.
* `accepter_region_name` - Specifies the region where the peer VPC resides. For complete lists of supported regions, see [Amazon Web Services](https://docs.atlas.mongodb.com/reference/amazon-aws/).
* `aws_account_id` - Account ID of the owner of the peer VPC.
* `provider_name` - Cloud provider for this VPC peering connection. If omitted, Atlas sets this parameter to AWS. (Possible Values `AWS`, `AZURE`, `GCP`).
* `route_table_cidr_block` - Peer VPC CIDR block or subnet.
* `vpc_id` - Unique identifier of the peer VPC.
* `error_state_name` - Error state, if any. The VPC peering connection error state value can be one of the following: `REJECTED`, `EXPIRED`, `INVALID_ARGUMENT`.
* `status_name` - (AWS Only) The VPC peering connection status value can be one of the following: `INITIATING`, `PENDING_ACCEPTANCE`, `FAILED`, `FINALIZING`, `AVAILABLE`, `TERMINATING`.
* `atlas_cidr_block` - CIDR block that Atlas uses for your clusters.
* `azure_directory_id` - Unique identifier for an Azure AD directory.
* `azure_subscription_id` - Unique identifer of the Azure subscription in which the VNet resides.
* `resource_group_name` - Name of your Azure resource group.
* `vnet_name` - Name of your Azure VNet.
* `error_state` - Description of the Atlas error when `status` is `Failed`, Otherwise, Atlas returns `null`.
* `status` - (Azure/GCP Only) Status of the Atlas network peering connection.  Azure/GCP: `ADDING_PEER`, `AVAILABLE`, `FAILED`, `DELETING` GCP Only:  `WAITING_FOR_USER`.
* `gcp_project_id` - GCP project ID of the owner of the network peer.
* `atlas_gcp_project_id` - The Atlas GCP Project ID for the GCP VPC used by your atlas cluster that it is need to set up the reciprocal connection.
* `atlas_vpc_name` - The Atlas VPC Name is used by your atlas clister that it is need to set up the reciprocal connection.
* `network_name` - Name of the network peer to which Atlas connects.
* `error_message` - When `"status" : "FAILED"`, Atlas provides a description of the error.


## Import

Clusters can be imported using project ID and network peering peering id, in the format `PROJECTID-PEERID-PROVIDERNAME`, e.g.

```
$ terraform import mongodbatlas_network_peering.my_peering 1112222b3bf99403840e8934-5cbf563d87d9d67253be590a-AWS
```

See detailed information for arguments and attributes: [MongoDB API Network Peering Connection](https://docs.atlas.mongodb.com/reference/api/vpc-create-peering-connection/)
