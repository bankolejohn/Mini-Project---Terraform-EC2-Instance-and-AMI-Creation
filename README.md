# Mini Project: Terraform EC2 Instance and AMI Creation

## Table of Contents

1.  [Project Overview](#1-project-overview "null")
    
2.  [Objectives](#2-objectives "null")
    
3.  [Prerequisites](#3-prerequisites "null")
    
4.  [Project Tasks](#4-project-tasks "null")
    
    *   [Task 1: Confirm the Prerequisites](#task-1-confirm-the-prerequisites "null")
        
    *   [Task 2: Developing the Terraform Script](#task-2-developing-the-terraform-script "null")
        
    *   [Task 3: Executing the Terraform Script](#task-3-executing-the-terraform-script "null")
        
    *   [Task 4: Confirm Resources](#task-4-confirm-resources "null")
        
    *   [Task 5: Clean Up](#task-5-clean-up "null")
        
5.  [Observations and Challenges](#5-observations-and-challenges "null")
    
6.  [Conclusion](#6-conclusion "null")
    

## 1\. Project Overview

This mini-project guides you through automating the creation of an Amazon Web Services (AWS) EC2 instance and subsequently creating a custom Amazon Machine Image (AMI) from that instance, all using **Terraform**. This project emphasizes core Infrastructure as Code (IaC) principles, enabling repeatable and consistent infrastructure deployments.

## 2\. Objectives

By completing this project, you will:

*   Learn how to write basic Terraform configuration files.
    
*   Understand how to use Terraform to automate the creation of an EC2 instance on AWS.
    
*   Learn how to use Terraform to automate the creation of an AMI from an already created EC2 instance on AWS.
    

## 3\. Prerequisites

To successfully complete this project, ensure you have the following installed and configured on your local machine:

*   **AWS Account:** An active and functional AWS account.
    
*   **AWS CLI:** The AWS Command Line Interface installed and configured with credentials that have sufficient permissions to create EC2 instances, Security Groups, and AMIs. Terraform will use these locally configured AWS CLI credentials to communicate with your AWS account.
    
*   **Terraform:** Terraform installed on your computer.
    

## 4\. Project Tasks

### Task 1: Confirm the Prerequisites

Before diving into Terraform, let's confirm your local environment is ready.

1.  **Login into your AWS Account:**
    
    *   Open your web browser and log in to the AWS Management Console to confirm your account is functional.
        
2.  **Confirm AWS CLI Installation:**
    
    *   Open your terminal or command prompt and run:
        
            aws --version
            
        
    *   You should see an output similar to this:
        
            aws-cli/2.x.x Python/3.x.x <OS>/<Architecture> botocore/2.x.x
            
        
3.  **Confirm AWS CLI Configuration:**
    
    *   Run `aws configure list` to confirm your AWS CLI is configured with your credentials and default settings:
        
            aws configure list
            
        
    *   You should see an output similar to this (values will be masked and specific to your setup):
        
                  Name                    Value             Type    Location
                  ----                    -----             ----    --------
               profile                <not set>             None    None
            access_key     ****************ABCD shared-credentials-file
            secret_key     ****************WXYZ shared-credentials-file
                region                us-east-1      config-file    ~/.aws/config
              output                     json      config-file    ~/.aws/config
            
        
4.  **Verify AWS CLI Authentication:**
    
    *   Run `aws sts get-caller-identity` to verify that the AWS CLI can successfully authenticate to your AWS Account and confirm the identity you're operating as:
        
            aws sts get-caller-identity
            
        
    *   You should see an output similar to this:
        
            {
                "UserId": "AIDAXxxxxxxxxxxxxxxxx:YourUserName",
                "Account": "123456789012",
                "Arn": "arn:aws:iam::123456789012:user/YourUserName"
            }
            
        
5.  **Confirm Terraform Installation:**
    
    *   Run `terraform --version` to confirm Terraform is installed and accessible in your PATH:
        
            terraform --version
            
        
    *   You should see an output similar to this:
        
            Terraform v1.x.x
            on <OS>_<Architecture>
            
        

### Task 2: Developing the Terraform Script to create EC2 Instance and AMI from it

This task involves setting up your Terraform project directory and writing the configuration file (`main.tf`).

1.  **Create a new directory** for this Terraform project and navigate into it:
    
        mkdir terraform-ec2-ami
        cd terraform-ec2-ami
        
    
2.  **Create a Terraform configuration file:** `nano main.tf` (or your preferred editor).
    
        nano main.tf
        
    
3.  **Copy and paste the following script** into `main.tf`.
    
    **IMPORTANT:**
    
    *   **Replace `"us-east-1"`** with your desired AWS Account region if different.
        
    *   **You MUST replace `"your-key-pair-name"` with the actual name of an existing SSH key pair in your AWS account.** This is critical for you to be able to SSH into the created EC2 instance. If you don't have one, create it in the EC2 console.
        
    *   The `aws_security_group` resource is included to allow SSH access. **Be cautious with `0.0.0.0/0` in `cidr_blocks` for production environments**, as it allows access from any IP address. For production, you should restrict this to your specific IP range.
        

    # Configure the AWS Provider
    # Ensure your AWS credentials are configured (e.g., via AWS CLI, environment variables, or ~/.aws/credentials)
    provider "aws" {
      region = "us-east-1" # Change this to your desired AWS region (e.g., "eu-west-1", "ap-southeast-2")
    }
    
    # Data source to get the latest Ubuntu 20.04 LTS AMI
    # This ensures you always use the most recent image for your specified region.
    data "aws_ami" "ubuntu_focal_20_04" {
      most_recent = true
      owners      = ["099720109477"] # Canonical's AWS account ID for Ubuntu AMIs
    
      filter {
        name   = "name"
        # Matches Ubuntu 20.04 LTS (Focal Fossa) server images, HVM, SSD backed.
        values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
      }
    
      filter {
        name   = "virtualization-type"
        values = ["hvm"]
      }
    
      filter {
        name   = "architecture"
        values = ["x86_64"]
      }
    }
    
    # Resource: AWS Security Group for SSH Access
    # This security group allows inbound SSH (port 22) traffic from anywhere.
    # WARNING: For production environments, restrict cidr_blocks to specific IP ranges.
    resource "aws_security_group" "ssh_access" {
      name        = "terraform-ssh-access-sg"
      description = "Allow SSH inbound traffic for Terraform-created instance"
    
      ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"] # Be cautious with 0.0.0.0/0 in production
      }
    
      egress {
        from_port   = 0
        to_port     = 0
        protocol    = "-1" # Allows all outbound traffic
        cidr_blocks = ["0.0.0.0/0"]
      }
    
      tags = {
        Name = "Terraform-SSH-SG"
      }
    }
    
    # Resource: AWS EC2 Instance
    resource "aws_instance" "my_ec2_spec" {
      # Use the ID of the latest Ubuntu 20.04 AMI found by the data source
      ami           = data.aws_ami.ubuntu_focal_20_04.id
      instance_type = "t3.micro" # Changed from t2.micro to t3.micro for the latest instance type
    
      # Link to the security group created above
      vpc_security_group_ids = [aws_security_group.ssh_access.id]
    
      # --- IMPORTANT: Replace "your-key-pair-name" with an actual key pair name in your AWS account ---
      # You need an SSH key pair to connect to the instance.
      key_name = "your-key-pair-name"
    
      tags = {
        Name = "Terraform-created-EC2-Instance"
      }
    }
    
    # Resource: AWS AMI Creation from the EC2 Instance
    resource "aws_ami_from_instance" "my_ec2_spec_ami" {
      name               = "my-ec2-ami-from-terraform" # Updated name to be more distinct
      description        = "My AMI created from my EC2 Instance with Terraform script"
      source_instance_id = aws_instance.my_ec2_spec.id
    
      tags = {
        Name = "Terraform-AMI-Snapshot"
      }
    }
    
    # Output the public IP address of the created EC2 instance
    output "instance_public_ip" {
      description = "The public IP address of the EC2 instance"
      value       = aws_instance.my_ec2_spec.public_ip
    }
    
    # Output the ID of the created AMI
    output "ami_id" {
      description = "The ID of the AMI created from the EC2 instance"
      value       = aws_ami_from_instance.my_ec2_spec_ami.id
    }
