# Terraform-with-ansible

Here we are creating a Classic Load Balancer(CLB) with an auto-scaling group using terraform and provisioning using ansible. While performing provisioning we use serial keywords, this is used to reduce cost and down time. By using this serial keyword, the instance won't get deleted or recreated while performing the code update, but changes will get applied to all instances. Serial is the number of hosts that Ansible should be running side by side.

## Prerequisites

- An instance with installed terraform and ansible
- Valid AWS IAM user credentials with required access

## Create project directory

First we have to create a directory for our project (myproject), Lets create a file for declaring the variables.

```bash
mkdir myproject
cd myproject
```

**Create a varriable.tf file**

```bash
variable "project" {}
variable "region" {}
variable "access_key" {}
variable "secret_key" {}
```

**Create a provider.tf file**

```bash
 provider "aws" {
  region     = var.region
  access_key = var.access_key
  secret_key = var.secret_key
 }
 ```
 
 **Create a variable.tfvars**
 
 ```bash
 project = "your-project-name"
region = "ap-south-1"
access_key = "your-secret-key"
secret_key = "your-access-key"
```

**Create a setup.sh**

Here I'm using this script to add under the user data while creating a launch configuration.
```bash
#!/bin/bash

echo "ClientAliveInterval 60" >> /etc/ssh/sshd_config
echo "LANG=en_US.utf-8" >> /etc/environment
echo "LC_ALL=en_US.utf-8" >> /etc/environment

yum install httpd php git -y
git clone https://github.com/radin-lawrence/aws-elb-site.git /var/website/
cp -r /var/website/* /var/www/html/
chown -R apache:apache /var/www/html/*
systemctl restart httpd.service
systemctl enable httpd.service

```

**Create datasource.tf**


Data sources in Terraform are used to get information about resources external to Terraform, and use them to set up your Terraform resources.

```bash

Example Usage
data "aws_instances" "ec" {
  instance_tags = {
    Name = "${var.project}"
  }
```
**Create main.tf file**

```bash
# ------------------------
# Import ssh key pair
# -----------------------

resource "aws_key_pair" "mykey" {
  key_name   = "testkey"
  public_key = file("testkey.pub")
}

# -----------------------------------------
# Security Group For Webserver Access
# ------------------------------------------

resource "aws_security_group" "webserver" {
  name        = "${var.project}-webserver"
  description = "Allow 80 and 443"

   ingress {
    description      = ""
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

   ingress {
    description      = ""
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  ingress {
    description      = ""
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags = {
    Name = "${var.project}-webserver"
  }
}


# -------------------------------
# Classic Loadbalncer
# -------------------------------

resource "aws_elb" "clb" {
  name               = "myclb"
  availability_zones = ["ap-south-1a", "ap-south-1b"]
  security_groups = [aws_security_group.webserver.id]

  listener {
    instance_port     = 80
    instance_protocol = "http"
    lb_port           = 80
    lb_protocol       = "http"
  }

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 5
    target              = "HTTP:80/health.html"
    interval            = 10
  }

  cross_zone_load_balancing   = true
  idle_timeout                = 60
  connection_draining         = true
  connection_draining_timeout = 5

  tags = {
    Name = "${var.project}-elb"
  }
}

# ------------------------------------------
# Launch Configuration
# ------------------------------------------

resource "aws_launch_configuration" "lc" {
  name_prefix   = "${var.project}-lc"
  image_id = "ami-0a3277ffce9146b74"
  instance_type = "t2.micro"
  key_name = aws_key_pair.mykey.id
  security_groups= [aws_security_group.webserver.id]
  user_data = file("script.sh")
  lifecycle {
    create_before_destroy = true
  }
}

# -------------------------------------
# AutoScaling Group
# --------------------------------------

resource "aws_autoscaling_group" "asg" {
  name                      = "${var.project}-asg"
  max_size                  = 2
  min_size                  = 2
  health_check_grace_period = 120
  health_check_type         = "ELB"
  desired_capacity          = 2
  force_delete              = true
  launch_configuration      = aws_launch_configuration.lc.name
  availability_zones        = ["ap-south-1a","ap-south-1b" ]
  load_balancers            = [aws_elb.clb.name]
  wait_for_elb_capacity     = 2

  tag {
    key                 = "Name"
    value               = "${var.project}"
    propagate_at_launch = true
  }

  tag {
    key                 = "project"
    value               = "${var.project}"
    propagate_at_launch = true
  }
}


```
**Create output.tf**

output "url" {
  value = "http://{aws_elb.clb.dns_name}"
 }
 
 ouutput "ip"
   value = data.aws_instances.ec.



