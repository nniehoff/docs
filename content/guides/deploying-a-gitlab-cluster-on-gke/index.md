---
title: "Deploying a GitLab Cluster on GCP/GKE"
date: 2019-05-28
tags: ["gcp", "gke", "gitlab", "ci"]
---

This guide explains how to deploy a production-grade GitLab cluster running on GKE using our GCP open-source modules.

You could also use the marketplace: https://console.cloud.google.com/marketplace/details/gitlab-public/gitlab?filter=solution-type:k8s.
however we recommend our solution for higher availability.

In order to follow this guide you will need:

1. A GCP account with billing enabled. There is a [free tier](https://cloud.google.com/free/) that includes \$300 of free credit over a 12 month period.
2. [Terraform](https://learn.hashicorp.com/terraform/getting-started/install.html) v0.10.3 or later installed locally.
3. A recent version of the [gcloud](https://cloud.google.com/sdk/gcloud/) command-line tool.
4. [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl) v1.10.11 or higher
5. [Helm](https://docs.helm.sh/using_helm/#installing-helm) v2.11.0

## Architecture

In addition we use a Cloud SQL database for better durability and availability.

Our deployment will include:

- The core GitLab components

We use Kubergrunt to securely install Helm on our GKE cluster.

We deploy a Cloud SQL for PostgreSQL instance with a private IP.

**Note:** You cannot connect to the private Cloud SQL instance from outside Google Cloud Platform.

## Deploying a GKE Cluster

Before we can install GitLab, we need to first deploy a [Google Kubernetes Engine](https://cloud.google.com/kubernetes-engine/)
cluster. By using our open-source [GKE module](https://github.com/gruntwork-io/terraform-google-gke) we can easily
deploy a production-grade GKE cluster.

First, let’s create a new directory to work in:

```bash
$ mkdir -p google-ci-gitlab
$ cd google-ci-gitlab
```

Then create a `main.tf` file and copy the following code:

```hcl
provider "google" {
  version = "~> 2.7.0"
  project = "${var.project}"
  region  = "${var.region}"
}

provider "google-beta" {
  version = "~> 2.7.0"
  project = "${var.project}"
  region  = "${var.region}"
}

# ---------------------------------------------------------------------------------------------------------------------
# DEPLOY A PRIVATE CLUSTER IN GOOGLE CLOUD PLATFORM
# ---------------------------------------------------------------------------------------------------------------------

module "gke_cluster" {
  # Use a recent version of the gke-cluster module
  source = "git::git@github.com:gruntwork-io/terraform-google-gke.git//modules/gke-cluster?ref=v0.1.2"

  name = "${var.cluster_name}"

  project  = "${var.project}"
  location = "${var.location}"
  network  = "${module.vpc_network.network}"

  # We're deploying the cluster in the 'public' subnetwork to allow outbound internet access
  # See the network access tier table for full details:
  # https://github.com/gruntwork-io/terraform-google-network/tree/master/modules/vpc-network#access-tier
  subnetwork = "${module.vpc_network.public_subnetwork}"

  # When creating a private cluster, the 'master_ipv4_cidr_block' has to be defined and the size must be /28
  master_ipv4_cidr_block = "${var.master_ipv4_cidr_block}"

  # This setting will make the cluster private
  enable_private_nodes = "true"

  # To make testing easier, we keep the public endpoint variable available. In production, we highly recommend restricting access to only within the network boundary, requiring your users to use a bastion host or VPN.
  disable_public_endpoint = "false"

  # With a private cluster, it is highly recommended to restrict access to the cluster master
  # However, for example purposes we will allow all inbound traffic.
  master_authorized_networks_config = [{
    cidr_blocks = [{
      cidr_block   = "0.0.0.0/0"
      display_name = "all-for-testing"
    }]
  }]

  cluster_secondary_range_name = "${module.vpc_network.public_subnetwork_secondary_range_name}"
}

# ---------------------------------------------------------------------------------------------------------------------
# CREATE A NODE POOL
# ---------------------------------------------------------------------------------------------------------------------

resource "google_container_node_pool" "node_pool" {
  provider = "google-beta"

  name     = "private-pool"
  project  = "${var.project}"
  location = "${var.location}"
  cluster  = "${module.gke_cluster.name}"

  initial_node_count = "1"

  autoscaling {
    min_node_count = "1"
    max_node_count = "5"
  }

  management {
    auto_repair  = "true"
    auto_upgrade = "true"
  }

  node_config {
    image_type   = "COS"
    machine_type = "${var.node_machine_type}"

    labels = {
      private-pools-example = "true"
    }

    # Add a private tag to the instances. See the network access tier table for full details:
    # https://github.com/gruntwork-io/terraform-google-network/tree/master/modules/vpc-network#access-tier
    tags = [
      "${module.vpc_network.private}",
      "private-pool-example",
    ]

    disk_size_gb = "30"
    disk_type    = "pd-standard"
    preemptible  = false

    service_account = "${module.gke_service_account.email}"

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform",
    ]
  }

  lifecycle {
    ignore_changes = ["initial_node_count"]
  }

  timeouts {
    create = "30m"
    update = "30m"
    delete = "30m"
  }
}

# ---------------------------------------------------------------------------------------------------------------------
# CREATE A CUSTOM SERVICE ACCOUNT TO USE WITH THE GKE CLUSTER
# ---------------------------------------------------------------------------------------------------------------------

module "gke_service_account" {
  # Use a recent version of the gke-service-account module
  source = "git::git@github.com:gruntwork-io/terraform-google-gke.git//modules/gke-service-account?ref=v0.1.2"

  name        = "${var.cluster_service_account_name}"
  project     = "${var.project}"
  description = "${var.cluster_service_account_description}"
}

# ---------------------------------------------------------------------------------------------------------------------
# ALLOW THE CUSTOM SERVICE ACCOUNT TO PULL IMAGES FROM THE GCR REPO
# ---------------------------------------------------------------------------------------------------------------------

resource "google_storage_bucket_iam_member" "member" {
  bucket = "artifacts.${var.project}.appspot.com"
  role   = "roles/storage.objectViewer"
  member = "serviceAccount:${module.gke_service_account.email}"
}

# ---------------------------------------------------------------------------------------------------------------------
# CREATE CLOUD SQL DATABASE INSTANCE WITH PRIVATE IP
# ---------------------------------------------------------------------------------------------------------------------

module "postgres" {
  # Use a recent version of the cloud-sql module
  source = "git::git@github.com:gruntwork-io/terraform-google-sql.git//modules/cloud-sql?ref=v0.1.1"

  project = "${var.project}"
  region  = "${var.region}"
  name    = "${local.db_instance_name}"
  db_name = "${var.db_name}"

  engine       = "${var.postgres_version}"
  machine_type = "${var.db_machine_type}"

  # Indicate that we want to create a failover replica for high availability
  enable_failover_replica = true

  # These together will construct the master_user privileges, i.e.
  # 'master_user_name'@'master_user_host' IDENTIFIED BY 'master_user_password'.
  # These should typically be set as the environment variable TF_VAR_db_master_user_password, etc.
  # so you don't check these into source control."
  master_user_password = "${var.db_master_user_password}"

  master_user_name = "${var.db_master_user_name}"
  master_user_host = "%"

  # Pass the private network link to the module
  private_network = "${module.vpc_network.network}"

  # Wait for the vpc connection to complete
  dependencies = ["${google_service_networking_connection.private_vpc_connection.network}"]

  custom_labels = {
    test-id = "postgres-private-ip-example"
  }
}

# ---------------------------------------------------------------------------------------------------------------------
# CREATE A NETWORK TO DEPLOY THE CLUSTER TO
# ---------------------------------------------------------------------------------------------------------------------

module "vpc_network" {
  source = "git::git@github.com:gruntwork-io/terraform-google-network.git//modules/vpc-network?ref=v0.1.1"

  name_prefix = "${var.cluster_name}-network-${random_string.suffix.result}"
  project     = "${var.project}"
  region      = "${var.region}"

  cidr_block           = "${var.vpc_cidr_block}"
  secondary_cidr_block = "${var.vpc_secondary_cidr_block}"
}

# Reserve global internal address range for the peering
resource "google_compute_global_address" "private_ip_address" {
  provider      = "google-beta"
  name          = "${local.private_ip_name}"
  purpose       = "VPC_PEERING"
  address_type  = "INTERNAL"
  prefix_length = 16
  network       = "${module.vpc_network.network}"
}

# Establish VPC network peering connection using the reserved address range
resource "google_service_networking_connection" "private_vpc_connection" {
  provider                = "google-beta"
  network                 = "${module.vpc_network.network}"
  service                 = "servicenetworking.googleapis.com"
  reserved_peering_ranges = ["${google_compute_global_address.private_ip_address.name}"]
}

locals {
  db_instance_name = "${format("%s-%s", var.db_name_prefix, random_string.suffix.result)}"
  private_ip_name  = "private-ip-${random_string.suffix.result}"
}

# Use a random suffix to prevent overlap in network names
resource "random_string" "suffix" {
  length  = 4
  special = false
  upper   = false
}
```

The `main.tf` file is responsible for creating all of the GCP resources. After that let's create both the `outputs.tf`
and `variables.tf` files:

**outputs.tf**

```hcl
output "cluster_endpoint" {
  description = "The IP address of the cluster master."
  sensitive   = true
  value       = "${module.gke_cluster.endpoint}"
}

output "client_certificate" {
  description = "Public certificate used by clients to authenticate to the cluster endpoint."
  value       = "${module.gke_cluster.client_certificate}"
}

output "client_key" {
  description = "Private key used by clients to authenticate to the cluster endpoint."
  sensitive   = true
  value       = "${module.gke_cluster.client_key}"
}

output "cluster_ca_certificate" {
  description = "The public certificate that is the root of trust for the cluster."
  sensitive   = true
  value       = "${module.gke_cluster.cluster_ca_certificate}"
}

output "db_master_instance_name" {
  description = "The name of the database instance"
  value       = "${module.postgres.master_instance_name}"
}

output "db_master_ip_addresses" {
  description = "All IP addresses of the instance as list of maps, see https://www.terraform.io/docs/providers/google/r/sql_database_instance.html#ip_address-0-ip_address"
  value       = "${module.postgres.master_ip_addresses}"
}

output "db_master_private_ip" {
  description = "The private IPv4 address of the master instance"
  value       = "${module.postgres.master_private_ip_address}"
}

output "db_master_instance" {
  description = "Self link to the master instance"
  value       = "${module.postgres.master_instance}"
}

output "db_master_proxy_connection" {
  description = "Instance path for connecting with Cloud SQL Proxy. Read more at https://cloud.google.com/sql/docs/mysql/sql-proxy"
  value       = "${module.postgres.master_proxy_connection}"
}

output "db_name" {
  description = "Name of the default database"
  value       = "${module.postgres.db_name}"
}

output "db" {
  description = "Self link to the default database"
  value       = "${module.postgres.db}"
}
```

**variables.tf**

```hcl
# ---------------------------------------------------------------------------------------------------------------------
# REQUIRED PARAMETERS
# These variables are expected to be passed in by the operator.
# ---------------------------------------------------------------------------------------------------------------------

variable "project" {
  description = "The project ID where all resources will be launched."
}

variable "location" {
  description = "The location (region or zone) of the GKE cluster."
}

variable "region" {
  description = "The region for the network. If the cluster is regional, this must be the same region. Otherwise, it should be the region of the zone."
}

variable "db_master_user_name" {
  description = "The username part for the default user credentials, i.e. 'master_user_name'@'master_user_host' IDENTIFIED BY 'master_user_password'. This should typically be set as the environment variable TF_VAR_master_user_name so you don't check it into source control."
}

variable "db_master_user_password" {
  description = "The password part for the default user credentials, i.e. 'master_user_name'@'master_user_host' IDENTIFIED BY 'master_user_password'. This should typically be set as the environment variable TF_VAR_master_user_password so you don't check it into source control."
}

# ---------------------------------------------------------------------------------------------------------------------
# OPTIONAL PARAMETERS
# These parameters have reasonable defaults.
# ---------------------------------------------------------------------------------------------------------------------

variable "cluster_name" {
  description = "The name of the Kubernetes cluster."
  default     = "example-cluster"
}

variable "cluster_service_account_name" {
  description = "The name of the custom service account used for the GKE cluster. This parameter is limited to a maximum of 28 characters."
  default     = "example-cluster-sa"
}

variable "cluster_service_account_description" {
  description = "A description of the custom service account used for the GKE cluster."
  default     = "Example GKE Cluster Service Account managed by Terraform"
}

variable "node_machine_type" {
  description = "The machine type to use for GKE cluster nodes, see https://cloud.google.com/kubernetes-engine/pricing for more details"
  default     = "n1-standard-1"
}

# Note, after a db instance name is used, it cannot be reused for up to one week.
variable "db_name_prefix" {
  description = "The name prefix for the database instance. Will be appended with a random string. Use lowercase letters, numbers, and hyphens. Start with a letter."
}

variable "postgres_version" {
  description = "The engine version of the database, e.g. `POSTGRES_9_6`. See https://cloud.google.com/sql/docs/db-versions for supported versions."
  default     = "POSTGRES_9_6"
}

variable "db_machine_type" {
  description = "The machine type to use, see https://cloud.google.com/sql/pricing for more details"
  # See https://cloud.google.com/sql/docs/postgres/create-instance#machine-types
  default     = "db-custom-1-3840"
}

variable "db_name" {
  description = "Name for the db"
  default     = "gitlab"
}

variable "master_ipv4_cidr_block" {
  description = "The IP range in CIDR notation (size must be /28) to use for the hosted master network. This range will be used for assigning internal IP addresses to the master or set of masters, as well as the ILB VIP. This range must not overlap with any other ranges in use within the cluster's network."
  default     = "10.5.0.0/28"
}

# For the example, we recommend a /16 network for the VPC. Note that when changing the size of the network,
# you will have to adjust the 'cidr_subnetwork_width_delta' in the 'vpc_network' -module accordingly.
variable "vpc_cidr_block" {
  description = "The IP address range of the VPC in CIDR notation. A prefix of /16 is recommended. Do not use a prefix higher than /27."
  default     = "10.3.0.0/16"
}

# For the example, we recommend a /16 network for the secondary range. Note that when changing the size of the network,
# you will have to adjust the 'cidr_subnetwork_width_delta' in the 'vpc_network' -module accordingly.
variable "vpc_secondary_cidr_block" {
  description = "The IP address range of the VPC's secondary address range in CIDR notation. A prefix of /16 is recommended. Do not use a prefix higher than /27."
  default     = "10.4.0.0/16"
}

```

**Note:** Be sure to fill in any required variables that don't have a default value.

Now we can use Terraform to create the resources:

1. Run `terraform init`.
2. Run `terraform plan`.
3. If the plan looks good, run `terraform apply`.

Terraform will begin to create the GCP resources. This process can take up to XX minutes.

## Installing GitLab

We install GitLab by following the recommended way using a Helm chart. The chart contains all of the required components
to run GitLab and can easily scale to large deployments.

More information about the chart is explain on the GitLab cloud native Helm chart page. [GitLab cloud native Helm Chart | GitLab](https://docs.gitlab.com/charts/)

**Note:** Some GitLab features are currently not available when using the Helm chart:

- GitLab Pages
- GitLab Geo
- No in-cluster HA database
- Smartcard authentication

We can check the latest versions using the following command:

```bash
$ helm search -l gitlab/gitlab | head -n4
NAME                    CHART VERSION   APP VERSION     DESCRIPTION
gitlab/gitlab           1.9.0           11.11.0         Web-based Git-repository manager with wiki and issue-trac...
gitlab/gitlab           1.8.4           11.10.4         Web-based Git-repository manager with wiki and issue-trac...
gitlab/gitlab           1.8.3           11.10.3         Web-based Git-repository manager with wiki and issue-trac...
```

**Note**: The GitLab chart doesn't have the same version number as GitLab itself. For more information please refer to the [GitLab version mappings](https://docs.gitlab.com/charts/installation/version_mappings.html) document.

Please read the GitLab [Helm Deployment Guide](https://docs.gitlab.com/charts/installation/deployment.html) for more information on customizing
the GitLab installation.

## Deploying the Dockerized App

To deploy our Dockerized App on the GKE cluster, we can use the `kubectl` CLI tool to create a
[Kubernetes Pod](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/). A Pod is the smallest deployable
object in the Kubernetes object model and will contain only our `simple-web-app` Docker image.

First, we must configure `kubectl` to use the newly created cluster:

```
$ gcloud container clusters get-credentials example-private-cluster
```

**Note**: Be sure to substitute `example-private-cluster` with the name of your GKE cluster.

Use the `kubectl run` command to create a [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
named `simple-web-app-deploy` on your cluster:

```bash
$ kubectl run simple-web-app-deploy --image=gcr.io/${PROJECT_ID}/simple-web-app:v1 --port 8080
```

To see the Pod created by the last command, you can run:

```bash
$ kubectl get pods
```

The output should look similar to the following:

```bash
NAME                                     READY     STATUS             RESTARTS   AGE
simple-web-app-deploy-7fb787c449-vgtf6   0/1       ContainerCreating  0          7s
```

Now we need to expose the app to the public internet.

## Attaching a Load Balancer

So far we have deployed the dockerized app, but it is not currently accessible from the public internet. This is because
we have not assigned an external IP address or load balancer to the Pod. We can easily achieve this, by running the
following command:

```bash
$ kubectl expose deployment simple-web-app-deploy --type=LoadBalancer --port 80 --target-port 8080
```

**Note:** GKE assigns the external IP address to the Service resource, not the Deployment.

## Cleaning Up

In order to save costs, we recommend you destroy any infrastructure you've created by following this guide.

First, delete the Kubernetes Service:

```bash
$ kubectl delete service simple-web-app-deploy
```

This will destroy the Load Balancer created during the previous step.

Then remove GitLab using the Helm chart:

```bash
$ helm delete gitlab
```

Next, to destroy the GKE cluster, you can simply invoke the `terraform destroy` command:

```bash
$ terraform destroy
```

**Note**: This is a destructive command that will forcibly terminate and destroy your GKE cluster!