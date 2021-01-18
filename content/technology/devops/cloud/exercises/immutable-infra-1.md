---
type: exercise
slug: immutable-infra-1
guid: https://pasztor.at/blog/immutable-infra-1/
authors:
    - janos
categories:
- Technology
date: "2019-05-15T00:00:00Z"
publishDate: "2019-05-15T00:00:00Z"
summary: Recently I rebuilt the infrastructure that hosts this website following the principles of immutable infrastructure. Let's see how that works!
images:
- posts/immutable-infra-1.png
preview: posts/immutable-infra-1.jpg
tags:
- DevOps
- Cloud
title: Immutable Infrastructure in Practice — Part 1
---

For over a year I [ran my own CDN for this website](/blog/building-your-own-cdn). However, I had to amend the article about the CDN because a year later I had a number of issues with it. If you are interested, go click and have a read.

As I had to get rid of the CDN, I needed to rebuild my infrastructure. This time around I decided to set a different challenge for myself. I wanted to rebuild my infrastructure on a single server following the principles of 
[immutable infrastructure](/blog/immutable-infrastructure). The reason is reproducibility and testability. I want to be able to test updates automatically before applying them instead of simply rolling updates on existing servers.

In other words, the server would be built in such a way that every time I needed to do an update a new server would be installed and the old server would be deleted. This, of course, presents a unique challenge. Any data generated will need to be stored in some sort of a persistent storage.

{{% tip %}}
**Note:** This article will use Exoscale as a cloud provider. The company I worked for at the time of writing (A1 Digital) is a majority shareholder of Exoscale. This blog is my own and does not represent A1 Digital or Exoscale in any way and the thoughts expressed here are my own. The contents of these article can be replicated on any IaaS cloud provider. Later in this article I will describe how to port this system to AWS.
{{% /tip %}}

## Designing the system

![An illustration of the infrastructure being built. The content is in the content bucket on the object storage. From there it is fetched into the content folder which is in use by both nginx and PHP. The incoming user traffic is sent to nginx, which may ask PHP. Traefik also sends requests to Grafana, which may query Prometheus. Traefik is checked by UptimeRobot, which pushes alerts to VictorOps. Traefik and Prometheus are backed up into the Backup Bucket. Grafana also sends alerts to VictorOps.](posts/website-architecture.svg)

My blog is built using [Jekyll](https://jekyllrb.com/), an engine that generates static HTML files. This makes the whole ordeal much simpler as I can just simply generate content and copy it to the new server as it comes up. I don't need to worry about persisting data from one server to the next. If I was using [Wordpress](https://wordpress.org/), for example, I would have to build some sort of a redundant database on multiple servers. I would then have to perform a *rolling update*, replacing one server at a time.

So the content is not a problem. However, as I realized, monitoring data is. I am using [Prometheus](https://prometheus.io) as a primary monitoring and metrics collection system, and it would be nice if I didn't lose all my monitoring data when I update my servers. If you are wondering, yes, I am running Prometheus on the same server my web content is on, but I have an external monitoring setup too. We'll get to that in a minute.

If my cloud provider had support for network block storage, I could have put the prometheus data on such a volume. When I needed to replace the server, I could simply detach it from one server and attach it to the next. However, since a network block storage can only be attached to a single server at a time, I would have a downtime between the two servers.

Instead I opted to do what I call the *backup-and-restore* method. Every minute the server would back up the monitoring data to an object storage (think AWS S3) and when the new server comes up, it simply restores the monitoring data and continues where the other server left off. This would yield a maximum theoretical data loss of one minute, but I could seamlessly install the new server while the old one is still serving traffic.

To do the switchover, I opted to use a persistent IP address (elastic IP). This IP address would be pulled from one server to the next when the switchover happened.

As an external monitoring provider I'm using [Uptime Robot](https://uptimerobot.com/), and both Uptime Robot and Prometheus send the incidents to [VictorOps](https://victorops.com/), which is an incident management tool.

As for the webserver, and all the tools needed, I want to use [Docker](https://www.docker.com/). I could, of course, use a Docker registry, but I would have to either host it myself (yuck!) or pay for it separately. Instead, I opted to use [docker-compose](https://docs.docker.com/compose/) to build the containers right on the server. It's a little less efficient, but makes up for it in simplicity. 

To coordinate all this the infrastructure is built using [Terraform](https://www.terraform.io/), which is an excellent way to manage an immutable infrastructure.

## Terraform basics

The code, as usual, is [on GitHub](https://github.com/janoszen/pasztor.at-terraform), so you can go and
have a look at the complete project.

As first step we will set up an object storage backend for Terraform. This is needed because Terraform keeps a so-called statefile which stores which cloud resource (such as instances, object storage buckets, etc) corresponds to which resource in the Terraform config file. We store the state file on an object storage backend so I can continue using my Terraform setup in the event my computer crashes. If I didn't do this, the loss of my state file would mean I have to manually re-associate all existing cloud resources with the Terraform resources.

The backend configuration is quite simple:

```hcl-terraform
terraform {
  backend "s3" {
    bucket = "pasztor.at-terraform"
    key = "terraform.tfstate"
    region = "at-vie-1"
    endpoint = "https://sos-at-vie-1.exo.io"
    skip_credentials_validation = true
    skip_get_ec2_platforms = true
    skip_metadata_api_check = true
    skip_region_validation = true
    skip_requesting_account_id = true
  }
}
```

The good thing about the S3 backend in Terraform is that it supports multiple workspaces, so I can use the same state storage for my testing and my production environment. I can then run `terraform init` and supply the credentials:

```bash
terraform init \
    -backend-config="access_key=API KEY HERE" \
    -backend-config="secret_key=API SECRET HERE"
```

Now, as a next step I will set up my providers:

```hcl-terraform
provider "exoscale" {
  key = "${var.exoscale_key}"
  secret = "${var.exoscale_secret}"
}

// We use the AWS provider for object storage access
provider "aws" {
  region = "at-vie-1"
  skip_metadata_api_check = true
  skip_credentials_validation = true
  skip_region_validation = true
  skip_get_ec2_platforms = true
  skip_requesting_account_id = true
  endpoints {
    s3 = "https://sos-at-vie-1.exo.io"
    s3control = "https://sos-at-vie-1.exo.io"
  }
  access_key = "${var.exoscale_key}"
  secret_key = "${var.exoscale_secret}"
}

provider "tls" {

}

provider "uptimerobot" {
  api_key = "${var.uptimerobot_api_key}"
}
```

As you can see I am feeding the credentials via variables. I can use these variables to feed different credentials to my test system as the production. These variables are set up in the config as such:

```hcl-terraform
variable "exoscale_key" {}
variable "exoscale_secret" {}
//...
```

I, of course, don't always want to type in all the variable contents every time, so I supply them in a `tfvars` file. These files look like this:

```hcl-terraform
exoscale_key="API KEY HERE"
exoscale_secret="API SECRET HERE"
//...
```

I then run the Terraform config as such:

```bash
terraform apply -var-file=production.tfvars
```

## Object storage buckets

Object storage buckets, as mentioned before, are set up using the AWS provider with a few tweaks to support the Exoscale object storage:

```hcl-terraform
resource "aws_s3_bucket" "content" {
  bucket = "${var.content_bucket_name}"
  lifecycle {
    ignore_changes = [
      "object_lock_configuration"
    ]
  }
}
```

However, the bucket is empty at this point. Since we are talking immutable infrastructure, it is desired that running the Terraform configuration will provision the whole server. So we will set up a `null_resource` to run the Jekyll build and upload the content to the bucket:


```hcl-terraform
resource "null_resource" "site" {
  provisioner "local-exec" {
    environment {
      JEKYLL_ENV="prod"
    }
    working_dir = "../"
    command = "bundler exec jekyll build --future"
  }
  provisioner "local-exec" {
    working_dir = "../"
    command = "s3cmd sync --config=_terraform/s3.cfg --access_key=${var.exoscale_key} --secret_key=${var.exoscale_secret} --delete-removed ./_site/ s3://${var.content_bucket_name}"
  }
}
```

## Setting up the server

Now, the next trick is to set up the server. In Terraform this is quite simple:

```hcl-terraform
resource "exoscale_compute" "web" {
  display_name = "${var.server_hostname}"
  template = "${var.image}"
  size = "${var.instance_type}"
  disk_size = "${var.instance_disk}"
  key_pair = "${exoscale_ssh_keypair.initial.name}"
  state = "Running"
  zone = "at-vie-1"

  security_groups = ["${exoscale_security_group.web.name}"]
  ip6 = true

  user_data = <<EOF
#!/bin/bash

// Add server initialization script here
EOF

  //Create the new server before destroying the old one
  lifecycle {
    create_before_destroy = true
  }

  depends_on = [
    "null_resource.site"
  ]
}
```

Now, the initial server configuration can be passed in the `user_data` parameter. On Linux systems this data is usually processed by [cloud-init](https://cloudinit.readthedocs.io/en/latest/), an incredibly powerful
tool to provision servers.

In this case I'll pass a simple script to take care of the basics:

```bash
#!/bin/bash

#region Users
function create_user() {
  useradd -m -s /bin/bash $$1
  mkdir -p /home/$$1/.ssh
  echo "$$2" >/home/$$1/.ssh/authorized_keys
  chown -R $$1:$$1 /home/$$1
  gpasswd -a $$1 sudo
  gpasswd -a $$1 adm
}

sed -i -e 's/%sudo\s*ALL=(ALL:ALL)\s*ALL/%sudo ALL=(ALL:ALL) NOPASSWD:ALL/' /etc/sudoers
create_user janoszen "${var.ssh_key_janoszen}"
#endregion

# region Updates
DEBIAN_FRONTEND=noninteractive apt-get update
DEBIAN_FRONTEND=noninteractive apt-get -o Dpkg::Options::="--force-confnew" --force-yes -fuy upgrade
DEBIAN_FRONTEND=noninteractive apt-get -o Dpkg::Options::="--force-confnew" --force-yes -fuy dist-upgrade
DEBIAN_FRONTEND=noninteractive apt-get install -y rsync htop tcpdump tcpflow unzip mc
# endregion

# region Network
echo 'network: {config: disabled}' >/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
echo 'network:
    version: 2
    ethernets:
        eth0:
            dhcp4: true
            dhcp6: false
            addresses:
              - ${exoscale_ipaddress.vip.ip_address}/32
' >/etc/netplan/01-eth0.yaml
# endregion

# region SSH
echo '${tls_private_key.web-ecdsa.private_key_pem}' >/etc/ssh/ssh_host_ecdsa_key
echo '${tls_private_key.web-rsa.private_key_pem}' >/etc/ssh/ssh_host_rsa_key
rm /etc/ssh/ssh_host_dsa_key
rm /etc/ssh/ssh_host_ed25519_key
sed -i -e 's/#HostKey \/etc\/ssh\/ssh_host_dsa_key//' /etc/ssh/sshd_config
sed -i -e 's/#HostKey \/etc\/ssh\/ssh_host_ed25519_key//' /etc/ssh/sshd_config
sed -i -e 's/#HostKey \/etc\/ssh\/ssh_host_rsa_key/HostKey \/etc\/ssh\/ssh_host_rsa_key/' /etc/ssh/sshd_config
sed -i -e 's/#HostKey \/etc\/ssh\/ssh_host_ecdsa_key/HostKey \/etc\/ssh\/ssh_host_ecdsa_key/' /etc/ssh/sshd_config
sed -i -e 's/PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
echo '${tls_locally_signed_cert.web-ecdsa.cert_pem}' >/etc/ssh/ssh_host_ecdsa_cert
echo '${tls_locally_signed_cert.web-rsa.cert_pem}' >/etc/ssh/ssh_host_rsa_cert
echo '${tls_self_signed_cert.ca.cert_pem}' >/etc/ssh/ssh_ca_cert
echo 'HostCertificate /etc/ssh/ssh_host_ecdsa_cert' >>/etc/ssh/sshd_config
echo 'HostCertificate /etc/ssh/ssh_host_rsa_cert' >>/etc/ssh/sshd_config
sed -i -e 's/#Port 22/Port ${var.ssh_port}/' /etc/ssh/sshd_config
# endregion

# region Reboot
reboot --reboot
# endregion
```

As you can see I take care of a few things here. Creating my own user, running updates, configuring the extra IP address that I will be using for serving content, and finally passing a pre-generated SSH host key and changing the SSH port.

The pre-generated SSH key is important because I want to be able to SSH into the server without having to change my `known_hosts` file all the time. Terraform, thankfully, brings with it the tools to generate these keys:

```hcl-terraform
resource "tls_private_key" "web-ecdsa" {
  algorithm = "ECDSA"
}
```

There are more code bits about CA signing and what not, but we won't go into those here, feel free to look them up [in the Git repository](https://github.com/janoszen/pasztor.at/blob/master/_terraform/key.tf).

## Provisioning Docker

Now, I want to run Docker containers on my server. To do that I need to upload my whole Docker directory to the server. One way to do that would be to include it in the `user_data`, but there is a catch: I somehow need to wait until the server is completely provisioned before I switch over the IP. So instead, I opted to use provisioners.

To do that, as a first step I generated a temporary SSH key and registered it on Exoscale as an SSH key:

```hcl-terraform
resource "tls_private_key" "initial" {
  algorithm = "RSA"
}

resource "exoscale_ssh_keypair" "initial" {
  name = "${local.domain_name}"
  public_key = "${tls_private_key.initial.public_key_openssh}"
}
```

I then used this key to provision my instance, so the default `ubuntu` user would have this SSH key. This user is, of course, deleted at the end of the provisioning process, so the key really only works for a couple of minute. Using a temporary SSH key in this manner is useful because the provisioning process is independent of who is running the config, if I was working in a team, anyone could do it.

As mentioned, we are setting up a provisioner. I will spare you the details, suffice it to say, you can set up a file transfer via SSH in this way:

```hcl-terraform
resource "exoscale_compute" "web" {
  //...
  provisioner "file" {
    connection {
      agent = false
      type = "ssh"
      user = "ubuntu"
      port = "${var.ssh_port}"
      private_key = "${tls_private_key.initial.private_key_pem}"
      #TODO: replace with certificate when 0.12 comes out
      host_key = "${tls_private_key.web-rsa.public_key_openssh}"
    }
    source = "${data.archive_file.docker.output_path}"
    destination = "/srv/docker/docker.zip"
  }
}
```

Now, as you can see I'm referencing something called an `archive_file`. I did this to make the file transfer more efficient:

```hcl-terraform
data "archive_file" "docker" {
  type        = "zip"
  output_path = "${path.module}/docker.zip"

  source_dir = "docker/"
}
```

## Porting it to AWS

OK, I picked a relatively small cloud provider, as described. So how would you go about building this on AWS? And the answer is simpler than you may think. AWS, like Exoscale and many others, supports Infrastructure as a Service (IaaS), and even uses much of the same terminology. Instances are called
[EC2 instances](https://www.terraform.io/docs/providers/aws/d/instance.html), security groups are called [security groups](https://www.terraform.io/docs/providers/aws/d/security_group.html), elastic IPs are called [elastic IPs](https://www.terraform.io/docs/providers/aws/d/eip.html), etc.

However, unlike Exoscale the AWS networking model is a bit more complicated. In order to set up an instance you need to create a couple of pre-requisites with Terraform:

- A [VPC](https://www.terraform.io/docs/providers/aws/d/vpc.html)
- At least one [subnet](https://www.terraform.io/docs/providers/aws/d/subnet.html)
- An [internet gateway](https://www.terraform.io/docs/providers/aws/d/instances.html)
- A [routing table](https://www.terraform.io/docs/providers/aws/d/route_table.html)

It takes a bit of fiddling, but most of the things work just as well on any other provider.

{{% tip %}}
**Note:** Elastic IPs on AWS do not need to be added to the machine like on Exoscale, they are forwarded automatically.
{{% /tip %}}

## Next up: Docker

In the [next part of this series](/blog/immutable-infra-2) I will set up the whole Docker world on top of this server and explain how to work with Prometheus, Grafana, exporters and so on. [Keep on reading!](/blog/immutable-infra-2)
