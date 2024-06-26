[cloud_user@ip-10-0-1-168 ~]$ tree .
.
|-- instance.tf
`-- modularisedtfcode
    |-- main.tf
    |-- modules
    |   |-- ec2_instance
    |   |   |-- main.tf
    |   |   |-- outputs.tf
    |   |   `-- variables.tf
    |   |-- internet_gateway
    |   |   |-- main.tf
    |   |   |-- outputs.tf
    |   |   `-- variables.tf
    |   |-- route_table
    |   |   |-- main.tf
    |   |   |-- outputs.tf
    |   |   `-- variables.tf
    |   |-- security_group
    |   |   |-- main.tf
    |   |   |-- outputs.tf
    |   |   `-- variables.tf
    |   |-- subnet
    |   |   |-- main.tf
    |   |   |-- outputs.tf
    |   |   `-- variables.tf
    |   `-- vpc
    |       |-- main.tf
    |       |-- outputs.tf
    |       `-- variables.tf
    |-- outputs.tf
    `-- varibles.tf

=================================================
=================================================

[cloud_user@ip-10-0-1-168 ~]$ cat instance.tf 
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "kk_aws_vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "my-first-tf-node"
  }
}

resource "aws_internet_gateway" "kk_gw" {
  vpc_id = aws_vpc.kk_aws_vpc.id

  tags = {
    Name = "my-first-tf-node"
  }
}

resource "aws_route_table" "kk_aws_route_table" {
  vpc_id = aws_vpc.kk_aws_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.kk_gw.id
  }

  tags = {
    Name = "my-first-tf-node"
  }
}

resource "aws_subnet" "kk_aws_subnet" {
  vpc_id     = aws_vpc.kk_aws_vpc.id
  cidr_block = "10.0.1.0/24"

  tags = {
    Name = "my-first-tf-node"
  }
}

resource "aws_route_table_association" "kk_subnet_association" {
  subnet_id      = aws_subnet.kk_aws_subnet.id
  route_table_id = aws_route_table.kk_aws_route_table.id
}

resource "aws_security_group" "kk_instance_sg" {
  name        = "instance-sg"
  description = "Security group for instance with internet access"

  vpc_id = aws_vpc.kk_aws_vpc.id

  ingress {
    description = "Allow SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "my-first-tf-node"
  }
}

resource "aws_instance" "kk_vm" {
  ami           = "ami-0b3aef6bc281a13b2"
  subnet_id     = aws_subnet.kk_aws_subnet.id
  instance_type = "t3.micro"

  vpc_security_group_ids = [aws_security_group.kk_instance_sg.id]

  tags = {
    Name = "my-first-tf-node"
  }
}


=================================================
=================================================


[cloud_user@ip-10-0-1-168 modularisedtfcode]$ cat main.tf 
provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source = "./modules/vpc"

  cidr_block       = var.vpc_cidr_block
  instance_tenancy = "default"
  name             = "my-first-tf-node"
}

module "internet_gateway" {
  source = "./modules/internet_gateway"

  vpc_id = module.vpc.vpc_id
  name   = "my-first-tf-node"
}

module "route_table" {
  source = "./modules/route_table"

  vpc_id           = module.vpc.vpc_id
  route_cidr_block = "0.0.0.0/0"
  gateway_id       = module.internet_gateway.internet_gateway_id
  name             = "my-first-tf-node"
}

module "subnet" {
  source = "./modules/subnet"

  vpc_id         = module.vpc.vpc_id
  cidr_block     = var.subnet_cidr_block
  route_table_id = module.route_table.route_table_id
  name           = "my-first-tf-node"
}

module "security_group" {
  source = "./modules/security_group"

  vpc_id      = module.vpc.vpc_id
  name        = "instance-sg"
  description = "Security group for instance with internet access"
}

module "ec2_instance" {
  source = "./modules/ec2_instance"

  ami                = var.instance_ami
  subnet_id          = module.subnet.subnet_id
  instance_type      = var.instance_type
  security_group_ids = [module.security_group.security_group_id]
  name               = "my-first-tf-node"
}


---

[cloud_user@ip-10-0-1-168 modularisedtfcode]$ cat varibles.tf 
variable "vpc_cidr_block" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "subnet_cidr_block" {
  description = "CIDR block for the subnet"
  default     = "10.0.1.0/24"
}

variable "instance_ami" {
  description = "AMI ID for the EC2 instance"
  default     = "ami-0b3aef6bc281a13b2"
}

variable "instance_type" {
  description = "Instance type for the EC2 instance"
  default     = "t3.micro"
}
---


[cloud_user@ip-10-0-1-168 modularisedtfcode]$ cat outputs.tf 
output "vpc_id" {
  description = "ID of the created VPC"
  value       = module.vpc.vpc_id
}

output "subnet_id" {
  description = "ID of the created subnet"
  value       = module.subnet.subnet_id
}

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = module.ec2_instance.public_ip
}
---
=================================================
=================================================

[cloud_user@ip-10-0-1-168 modularisedtfcode]$ ls
main.tf  modules  outputs.tf  varibles.tf
[cloud_user@ip-10-0-1-168 modularisedtfcode]$ cd modules/
[cloud_user@ip-10-0-1-168 modules]$ ls
ec2_instance  internet_gateway  route_table  security_group  subnet  vpc
[cloud_user@ip-10-0-1-168 modules]$ tree .
.
|-- ec2_instance
|   |-- main.tf
|   |-- outputs.tf
|   `-- variables.tf
|-- internet_gateway
|   |-- main.tf
|   |-- outputs.tf
|   `-- variables.tf
|-- route_table
|   |-- main.tf
|   |-- outputs.tf
|   `-- variables.tf
|-- security_group
|   |-- main.tf
|   |-- outputs.tf
|   `-- variables.tf
|-- subnet
|   |-- main.tf
|   |-- outputs.tf
|   `-- variables.tf
`-- vpc
    |-- main.tf
    |-- outputs.tf
    `-- variables.tf

=================================================
=================================================

[cloud_user@ip-10-0-1-168 modules]$ cat ec2_instance/main.tf 
resource "aws_instance" "my_instance" {
  ami           = var.ami
  subnet_id     = var.subnet_id
  instance_type = var.instance_type

  vpc_security_group_ids = var.security_group_ids

  tags = {
    Name = var.name
  }
}
---


[cloud_user@ip-10-0-1-168 modules]$ cat ec2_instance/variables.tf 
variable "ami" {
  description = "ID of the AMI for the instance"
}

variable "subnet_id" {
  description = "ID of the subnet to launch the instance"
}

variable "instance_type" {
  description = "Type of instance to launch"
}

variable "security_group_ids" {
  description = "List of security group IDs to attach to the instance"
}

variable "name" {
  description = "Name tag for resources"
}
---


[cloud_user@ip-10-0-1-168 modules]$ cat ec2_instance/outputs.tf 
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.my_instance.id
}

output "public_ip" {
  description = "Public IP address of the EC2 instance (if applicable)"
  value       = aws_instance.my_instance.public_ip
}
---
=================================================
=================================================

---
[cloud_user@ip-10-0-1-168 modules]$ cat internet_gateway/main.tf 
resource "aws_internet_gateway" "my_igw" {
  vpc_id = var.vpc_id

  tags = {
    Name = var.name
  }
}
---

[cloud_user@ip-10-0-1-168 modules]$ cat internet_gateway/variables.tf 
variable "vpc_id" {
  description = "ID of the VPC to attach the internet gateway"
}

variable "name" {
  description = "Name tag for resources"
}
---


[cloud_user@ip-10-0-1-168 modules]$ cat internet_gateway/outputs.tf 
output "internet_gateway_id" {
  description = "ID of the internet gateway"
  value       = aws_internet_gateway.my_igw.id
}

---
=================================================
=================================================


[cloud_user@ip-10-0-1-168 modules]$ cat route_table/main.tf 
resource "aws_route_table" "my_route_table" {
  vpc_id = var.vpc_id

  route {
    cidr_block = var.route_cidr_block
    gateway_id = var.gateway_id
  }

  tags = {
    Name = var.name
  }
}
---
[cloud_user@ip-10-0-1-168 modules]$ cat route_table/variables.tf 
variable "vpc_id" {
  description = "ID of the VPC to associate the route table"
}

variable "route_cidr_block" {
  description = "CIDR block for the route in the route table"
}

variable "gateway_id" {
  description = "ID of the gateway (internet gateway, NAT gateway, etc.)"
}

variable "name" {
  description = "Name tag for resources"
}
---

[cloud_user@ip-10-0-1-168 modules]$ cat route_table/outputs.tf 
output "route_table_id" {
  description = "ID of the route table"
  value       = aws_route_table.my_route_table.id
}
---
=================================================
=================================================


[cloud_user@ip-10-0-1-168 modules]$ cat security_group/main.tf 
resource "aws_security_group" "my_security_group" {
  name        = var.name
  description = var.description
  vpc_id      = var.vpc_id

  ingress {
    description = "Allow SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = var.name
  }
}
---

[cloud_user@ip-10-0-1-168 modules]$ cat security_group/variables.tf 
variable "vpc_id" {
  description = "ID of the VPC to create the security group"
}

variable "name" {
  description = "Name tag for resources"
}

variable "description" {
  description = "Description for the security group"
}

---

[cloud_user@ip-10-0-1-168 modules]$ cat security_group/outputs.tf 
output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.my_security_group.id
}
---

=================================================
=================================================

[cloud_user@ip-10-0-1-168 modules]$ cat subnet/main.tf 
resource "aws_subnet" "my_subnet" {
  vpc_id            = var.vpc_id
  cidr_block        = var.cidr_block
  availability_zone = "us-east-1a"  # Specify the desired availability zone

  tags = {
    Name = var.name
  }
}

resource "aws_route_table_association" "my_subnet_association" {
  subnet_id      = aws_subnet.my_subnet.id
  route_table_id = var.route_table_id
}
[cloud_user@ip-10-0-1-168 modules]$ cat subnet/variables.tf 
variable "vpc_id" {
  description = "ID of the VPC to associate the subnet"
}

variable "cidr_block" {
  description = "CIDR block for the subnet"
  default     = "10.0.1.0/24"
}

variable "name" {
  description = "Name tag for resources"
  default     = "my-first-tf-node"
}

variable "route_table_id" {
  description = "ID of the route table to associate with the subnet"
}

[cloud_user@ip-10-0-1-168 modules]$ cat subnet/outputs.tf 
output "subnet_id" {
  description = "ID of the subnet"
  value       = aws_subnet.my_subnet.id
}

output "subnet_cidr_block" {
  description = "CIDR block of the subnet"
  value       = aws_subnet.my_subnet.cidr_block
}

output "subnet_availability_zone" {
  description = "Availability Zone of the subnet"
  value       = aws_subnet.my_subnet.availability_zone
}

=================================================
=================================================


[cloud_user@ip-10-0-1-168 modules]$ cat vpc/main.tf 
resource "aws_vpc" "my_vpc" {
  cidr_block       = var.cidr_block
  instance_tenancy = var.instance_tenancy

  tags = {
    Name = var.name
  }
}
[cloud_user@ip-10-0-1-168 modules]$ cat vpc/variables.tf 
variable "cidr_block" {
  description = "CIDR block for the VPC"
}

variable "instance_tenancy" {
  description = "Tenancy of instances in the VPC"
  default     = "default"
}

variable "name" {
  description = "Name tag for resources"
}

[cloud_user@ip-10-0-1-168 modules]$ cat vpc/outputs.tf 
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.my_vpc.id
}

=================================================
=================================================

[cloud_user@ip-10-0-1-168 modularisedtfcode]$ terraform init
Initializing modules...

Initializing the backend...

Initializing provider plugins...
- Using previously-installed hashicorp/aws v5.55.0

The following providers do not have any version constraints in configuration,
so the latest version was installed.

To prevent automatic upgrades to new major versions that may contain breaking
changes, we recommend adding version constraints in a required_providers block
in your configuration, with the constraint strings suggested below.

* hashicorp/aws: version = "~> 5.55.0"

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.

If you ever set or change modules or backend configuration for Terraform,
rerun this command to reinitialize your working directory. If you forget, other
commands will detect it and remind you to do so if necessary.

=================================================
=================================================

[cloud_user@ip-10-0-1-168 modularisedtfcode]$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # module.ec2_instance.aws_instance.my_instance will be created
  + resource "aws_instance" "my_instance" {
      + ami                                  = "ami-0b3aef6bc281a13b2"
      + arn                                  = (known after apply)
      + associate_public_ip_address          = (known after apply)
      + availability_zone                    = (known after apply)
      + cpu_core_count                       = (known after apply)
      + cpu_threads_per_core                 = (known after apply)
      + disable_api_stop                     = (known after apply)
      + disable_api_termination              = (known after apply)
      + ebs_optimized                        = (known after apply)
      + get_password_data                    = false
      + host_id                              = (known after apply)
      + host_resource_group_arn              = (known after apply)
      + iam_instance_profile                 = (known after apply)
      + id                                   = (known after apply)
      + instance_initiated_shutdown_behavior = (known after apply)
      + instance_lifecycle                   = (known after apply)
      + instance_state                       = (known after apply)
      + instance_type                        = "t3.micro"
      + ipv6_address_count                   = (known after apply)
      + ipv6_addresses                       = (known after apply)
      + key_name                             = (known after apply)
      + monitoring                           = (known after apply)
      + outpost_arn                          = (known after apply)
      + password_data                        = (known after apply)
      + placement_group                      = (known after apply)
      + placement_partition_number           = (known after apply)
      + primary_network_interface_id         = (known after apply)
      + private_dns                          = (known after apply)
      + private_ip                           = (known after apply)
      + public_dns                           = (known after apply)
      + public_ip                            = (known after apply)
      + secondary_private_ips                = (known after apply)
      + security_groups                      = (known after apply)
      + source_dest_check                    = true
      + spot_instance_request_id             = (known after apply)
      + subnet_id                            = (known after apply)
      + tags                                 = {
          + "Name" = "my-first-tf-node"
        }
      + tags_all                             = {
          + "Name" = "my-first-tf-node"
        }
      + tenancy                              = (known after apply)
      + user_data                            = (known after apply)
      + user_data_base64                     = (known after apply)
      + user_data_replace_on_change          = false
      + vpc_security_group_ids               = (known after apply)

      + capacity_reservation_specification {
          + capacity_reservation_preference = (known after apply)

          + capacity_reservation_target {
              + capacity_reservation_id                 = (known after apply)
              + capacity_reservation_resource_group_arn = (known after apply)
            }
        }

      + cpu_options {
          + amd_sev_snp      = (known after apply)
          + core_count       = (known after apply)
          + threads_per_core = (known after apply)
        }

      + ebs_block_device {
          + delete_on_termination = (known after apply)
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + snapshot_id           = (known after apply)
          + tags                  = (known after apply)
          + tags_all              = (known after apply)
          + throughput            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = (known after apply)
          + volume_type           = (known after apply)
        }

      + enclave_options {
          + enabled = (known after apply)
        }

      + ephemeral_block_device {
          + device_name  = (known after apply)
          + no_device    = (known after apply)
          + virtual_name = (known after apply)
        }

      + instance_market_options {
          + market_type = (known after apply)

          + spot_options {
              + instance_interruption_behavior = (known after apply)
              + max_price                      = (known after apply)
              + spot_instance_type             = (known after apply)
              + valid_until                    = (known after apply)
            }
        }

      + maintenance_options {
          + auto_recovery = (known after apply)
        }

      + metadata_options {
          + http_endpoint               = (known after apply)
          + http_protocol_ipv6          = (known after apply)
          + http_put_response_hop_limit = (known after apply)
          + http_tokens                 = (known after apply)
          + instance_metadata_tags      = (known after apply)
        }

      + network_interface {
          + delete_on_termination = (known after apply)
          + device_index          = (known after apply)
          + network_card_index    = (known after apply)
          + network_interface_id  = (known after apply)
        }

      + private_dns_name_options {
          + enable_resource_name_dns_a_record    = (known after apply)
          + enable_resource_name_dns_aaaa_record = (known after apply)
          + hostname_type                        = (known after apply)
        }

      + root_block_device {
          + delete_on_termination = (known after apply)
          + device_name           = (known after apply)
          + encrypted             = (known after apply)
          + iops                  = (known after apply)
          + kms_key_id            = (known after apply)
          + tags                  = (known after apply)
          + tags_all              = (known after apply)
          + throughput            = (known after apply)
          + volume_id             = (known after apply)
          + volume_size           = (known after apply)
          + volume_type           = (known after apply)
        }
    }

  # module.internet_gateway.aws_internet_gateway.my_igw will be created
  + resource "aws_internet_gateway" "my_igw" {
      + arn      = (known after apply)
      + id       = (known after apply)
      + owner_id = (known after apply)
      + tags     = {
          + "Name" = "my-first-tf-node"
        }
      + tags_all = {
          + "Name" = "my-first-tf-node"
        }
      + vpc_id   = (known after apply)
    }

  # module.route_table.aws_route_table.my_route_table will be created
  + resource "aws_route_table" "my_route_table" {
      + arn              = (known after apply)
      + id               = (known after apply)
      + owner_id         = (known after apply)
      + propagating_vgws = (known after apply)
      + route            = [
          + {
              + carrier_gateway_id         = ""
              + cidr_block                 = "0.0.0.0/0"
              + core_network_arn           = ""
              + destination_prefix_list_id = ""
              + egress_only_gateway_id     = ""
              + gateway_id                 = (known after apply)
              + ipv6_cidr_block            = ""
              + local_gateway_id           = ""
              + nat_gateway_id             = ""
              + network_interface_id       = ""
              + transit_gateway_id         = ""
              + vpc_endpoint_id            = ""
              + vpc_peering_connection_id  = ""
            },
        ]
      + tags             = {
          + "Name" = "my-first-tf-node"
        }
      + tags_all         = {
          + "Name" = "my-first-tf-node"
        }
      + vpc_id           = (known after apply)
    }

  # module.security_group.aws_security_group.my_security_group will be created
  + resource "aws_security_group" "my_security_group" {
      + arn                    = (known after apply)
      + description            = "Security group for instance with internet access"
      + egress                 = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = "Allow all outbound traffic"
              + from_port        = 0
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "-1"
              + security_groups  = []
              + self             = false
              + to_port          = 0
            },
        ]
      + id                     = (known after apply)
      + ingress                = [
          + {
              + cidr_blocks      = [
                  + "0.0.0.0/0",
                ]
              + description      = "Allow SSH from anywhere"
              + from_port        = 22
              + ipv6_cidr_blocks = []
              + prefix_list_ids  = []
              + protocol         = "tcp"
              + security_groups  = []
              + self             = false
              + to_port          = 22
            },
        ]
      + name                   = "instance-sg"
      + name_prefix            = (known after apply)
      + owner_id               = (known after apply)
      + revoke_rules_on_delete = false
      + tags                   = {
          + "Name" = "instance-sg"
        }
      + tags_all               = {
          + "Name" = "instance-sg"
        }
      + vpc_id                 = (known after apply)
    }

  # module.subnet.aws_route_table_association.my_subnet_association will be created
  + resource "aws_route_table_association" "my_subnet_association" {
      + id             = (known after apply)
      + route_table_id = (known after apply)
      + subnet_id      = (known after apply)
    }

  # module.subnet.aws_subnet.my_subnet will be created
  + resource "aws_subnet" "my_subnet" {
      + arn                                            = (known after apply)
      + assign_ipv6_address_on_creation                = false
      + availability_zone                              = "us-east-1a"
      + availability_zone_id                           = (known after apply)
      + cidr_block                                     = "10.0.1.0/24"
      + enable_dns64                                   = false
      + enable_resource_name_dns_a_record_on_launch    = false
      + enable_resource_name_dns_aaaa_record_on_launch = false
      + id                                             = (known after apply)
      + ipv6_cidr_block_association_id                 = (known after apply)
      + ipv6_native                                    = false
      + map_public_ip_on_launch                        = false
      + owner_id                                       = (known after apply)
      + private_dns_hostname_type_on_launch            = (known after apply)
      + tags                                           = {
          + "Name" = "my-first-tf-node"
        }
      + tags_all                                       = {
          + "Name" = "my-first-tf-node"
        }
      + vpc_id                                         = (known after apply)
    }

  # module.vpc.aws_vpc.my_vpc will be created
  + resource "aws_vpc" "my_vpc" {
      + arn                                  = (known after apply)
      + cidr_block                           = "10.0.0.0/16"
      + default_network_acl_id               = (known after apply)
      + default_route_table_id               = (known after apply)
      + default_security_group_id            = (known after apply)
      + dhcp_options_id                      = (known after apply)
      + enable_dns_hostnames                 = (known after apply)
      + enable_dns_support                   = true
      + enable_network_address_usage_metrics = (known after apply)
      + id                                   = (known after apply)
      + instance_tenancy                     = "default"
      + ipv6_association_id                  = (known after apply)
      + ipv6_cidr_block                      = (known after apply)
      + ipv6_cidr_block_network_border_group = (known after apply)
      + main_route_table_id                  = (known after apply)
      + owner_id                             = (known after apply)
      + tags                                 = {
          + "Name" = "my-first-tf-node"
        }
      + tags_all                             = {
          + "Name" = "my-first-tf-node"
        }
    }

Plan: 7 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.


==============================================
==============================================

