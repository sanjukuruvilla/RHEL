## Enabling GUI for RHEL9 with RDP connection via AWS CLI

### Table of Contents

1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Tasks](#tasks)
    - [1. Create an IAM User Group](#1-create-an-iam-user-group)
    - [2. Create an IAM User and Add to the 'cloud_shell_deployment' Group](#2-create-an-iam-user-and-add-to-the-cloud_shell_deployment-group)
    - [3. Create AWS VPC at useast-1](#3-create-aws-vpc-at-useast-1)
    - [4. Creating network access control list (NACL)](#4-creating-network-access-control-list-nacl-which-can-be-filtered-traffic-at-network-level)
    - [5. Create 1 private and 1 public subnet](#5-create-1-private-and-1-public-subnet)
    - [6. Create 1 public security group and 1 private security group](#6-create-1-public-security-group-and-1-private-security-group)
    - [7. Create 1 Private Route Table](#7-create-1-private-route-table)
    - [8. Create 1 Public Route Table](#8-create-1-public-route-table)
    - [9. Create 1 Internet Gateway with Public IP for Public Subnet](#9-create-1-internet-gateway-with-public-ip-for-public-subnet-publicsubnet_01-and-associate-it-with-public-route-table-publicrt_01)
    - [10. Create an Elastic IP for NAT Gateway](#10-create-an-elastic-ip-for-nat-gateway)
    - [11. Create 1 NAT Gateway for private subnet](#11-create-1-nat-gateway-for-private-subnet-and-associate-it-with-elastic-ip)
    - [12. Create EC2 Instances at Public subnet](#12-create-ec2-instances-at-public-subnet-publicsubnet_01)
    - [13. Install Packages Related to Desktop or MATE Package/Distro](#13-install-packages-related-to-desktop-or-mate-packagedistro)
    - [14. Adding user and adding the user to wheel group](#14-adding-user-and-adding-the-user-to-wheel-group)
    - [15. Installing the services required for enabling RDP connection](#15-installing-the-services-required-for-enabling-rdp-connection)
    - [16. Enable GUI Package for RHEL9](#16-enable-gui-package-for-rhel9)
    - [17. RDP connection to the RHEL 9 server as GUI](#17-rdp-connection-to-the-rhel-9-server-as-gui)
    - [18. Post-Connection Configuration](#18-post-connection-configuration)
    - [19. Troubleshooting](#19-troubleshooting)
    - [20. Conclusion](#20-conclusion)
4. [Contributions](#Contributions)
5. [License](#license)

### Introduction

This guide provides detailed steps to enable a graphical user interface (GUI) for Red Hat Enterprise Linux 9 (RHEL9) instances on AWS using Remote Desktop Protocol (RDP) connection. All tasks are performed using AWS CLI commands, allowing for automation and ease of deployment.

### Prerequisites

Before proceeding, ensure you have:

- An AWS account with appropriate permissions.
- AWS CLI installed and configured on your local machine.

### Tasks

#### 1. Create an IAM User Group

```bash
# Create an IAM user group named 'cloud_shell_deployment'
aws iam create-group --group-name [Group Name]
```

#### 2. Create an IAM User and Add to the 'cloud_shell_deployment' Group

```bash
# Create an IAM user and add them to the 'cloud_shell_deployment' group
aws iam add-user-to-group --user-name [Username] --group-name [Group Name]
```

#### 3. Create AWS VPC at useast-1

```bash
# Create an AWS VPC in the us-east-1 (N.Virginia) region
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region us-east-1
```

#### 4. Creating network access control list (NACL) which can be filter traffic at network level

```bash
# Create a network access control list (NACL)
aws ec2 create-network-acl --vpc-id [VPC ID] --network-acl-name [ACL Name]
```

#### Additional Steps:[Optional]

```
### Configure Access Control List (ACL) Rules for Security

Before proceeding, ensure that appropriate Access Control List (ACL) rules are configured to control traffic to and from your VPC. You can use the `aws ec2 create-network-acl` command to create a new ACL and `aws ec2 create-network-acl-entry` to add rules to it.

```bash
# Create a new network ACL
aws ec2 create-network-acl --vpc-id <YOUR_VPC_ID> --network-acl-name <YOUR_ACL_NAME>

# Add inbound and outbound rules to the ACL as needed
# Example: Add an inbound rule allowing SSH traffic
aws ec2 create-network-acl-entry --network-acl-id <YOUR_NACL_ID> --rule-number 100 --protocol tcp --rule-action allow --egress false --cidr-block 0.0.0.0/0 --port-range From=22,To=22
```

Replace `<YOUR_VPC_ID>`, `<YOUR_ACL_NAME>`, and `<YOUR_NACL_ID>` with your actual values.

---

Create a default outbound rule allowing all traffic:

```bash
aws ec2 create-network-acl-entry --network-acl-id your-nacl-id --rule-number 100 --protocol -1 --rule-action allow --egress true --cidr-block 0.0.0.0/0
```

Create a default inbound rule allowing all traffic:

```bash
aws ec2 create-network-acl-entry --network-acl-id your-nacl-id --rule-number 100 --protocol -1 --rule-action allow --egress false --cidr-block 0.0.0.0/0
```

#### 5. Create 1 private and 1 public subnet

```bash
# Create a public subnet
aws ec2 create-subnet --vpc-id [VPC ID] --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --region us-east-1

# Create a private subnet
aws ec2 create-subnet --vpc-id [VPC ID] --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --region us-east-1
```

#### 6. Create Security Groups

```bash
# Create a public security group
aws ec2 create-security-group --group-name Pubsg_01 --description "Public Security Group" --vpc-id [VPC ID] --region us-east-1

# Create a private security group
aws ec2 create-security-group --group-name privatesg_01 --description "Private Security Group" --vpc-id [VPC ID] --region us-east-1
```

**Additional Configuration:**

Ensure both security groups have inbound rules allowing traffic for HTTP, HTTPS, RDP, Oracle database, and SSH from any IPv4 address.

**Public Security Group Inbound Rules:**

```bash
# Allow HTTP traffic
$ aws ec2 authorize-security-group-ingress --group-id sg-0bd8036c9a4601f9a --protocol tcp --port 80 --cidr 0.0.0.0/0
# Allow HTTPS traffic
$ aws ec2 authorize-security-group-ingress --group-id sg-0bd8036c9a4601f9a --protocol tcp --port 443 --cidr 0.0.0.0/0
# Allow RDP traffic
$ aws ec2 authorize-security-group-ingress --group-id sg-0bd8036c9a4601f9a --protocol tcp --port 3389 --cidr 0.0.0.0/0
# Allow Oracle database traffic
$ aws ec2 authorize-security-group-ingress --group-id sg-0bd8036c9a4601f9a --protocol tcp --port 1521 --cidr 0.0.0.0/0
# Allow SSH traffic
$ aws ec2 authorize-security-group-ingress --group-id sg-0bd8036c9a4601f9a --protocol tcp --port 22 --cidr 0.0.0.0/0
```

**Private Security Group Configuration:**

Repeat the above steps to update the private security group.

#### 7. Create 1 Private Route Table

```bash
# Create a private route table
aws ec2 create-route-table --vpc-id [VPC ID] --region us-east-1
```


#### 8. Create 1 Public Route Table

```bash
# Create a public route table
aws ec2 create-route-table --vpc-id [VPC ID] --region us-east-1

# Associate public subnet with public route table
$ aws ec2 associate-route-table --subnet-id <PUBLIC_SUBNET_ID> --route-table-id <PUBLIC_ROUTE_TABLE_ID> --region us-east-1

```



#### 9. Create 1 Internet Gateway with Public IP for Public Subnet "Publicsubnet_01" and Associate it with Public Route Table "Publicrt_01"

```bash
# Create an internet gateway
aws ec2 create-internet-gateway --region us-east-1

# Edit the public routable routes and attach the internet gateway with destination: 0.0.0.0/0 target: internet gateway
$ aws ec2 create-route --route-table-id <PUBLIC_ROUTE_TABLE_ID> --destination-cidr-block 0.0.0.0/0 --gateway-id <INTERNET_GATEWAY_ID>

```



#### 10. Create an Elastic IP for NAT Gateway

```bash
# Allocate an Elastic IP address
aws ec2 allocate-address --region us-east-1
```



#### 11. Create 1 NAT Gateway (use public subnet ID) and Associate it with Elastic IP

```bash

#Note: The NAT gateway should be in public subnet where the internetgateway is present and Edit the private routable routes and aƩach the natgateway with destination: 0.0.0.0/0 target :
natgateway

# Create a NAT gateway
aws ec2 create-nat-gateway --subnet-id [Public Subnet ID] --allocation-id [Elastic IP Allocation ID] --region us-east-1

# Edit the private routable routes and attach the NAT gateway with destination 0.0.0.0/0 target: natgateway
$ aws ec2 create-route --route-table-id <PRIVATE_ROUTE_TABLE_ID> --destination-cidr-block 0.0.0.0/0 --nat-gateway-id <NAT_GATEWAY_ID>

```


#### 12. Create EC2 Instances at Public subnet "Publicsubnet_01"

```bash
# Launch EC2 instances
aws ec2 run-instances --image-id [Image ID] --instance-type [Instance Type] --key-name [Key Name] --vpc-id [VPC ID] --subnet-id [Public Subnet ID] --security-group-ids [Security Group IDs] --associate-public-ip-address
```


#### 13. Install Packages Related to Desktop or MATE Package/Distro

```bash
# Install packages relevant to the desktop environment
sudo dnf groupinstall "Server with GUI" -y
```

#### 14. Adding user and adding the user to wheel group

```bash
# Create a new user
sudo useradd -m -s /bin/bash sanju

# Set a password for the user
sudo passwd sanju

# Add the user to the wheel group
sudo usermod -aG wheel sanju
```

#### 15. Installing the services required for enabling RDP connection

```bash
# Install XRDP and TigerVNC server
sudo dnf install xrdp tigervnc-server -y

# Start the XRDP service
sudo systemctl start xrdp.service

# Enable XRDP service to start on boot
sudo systemctl enable xrdp.service

# Start the firewall service
sudo systemctl start firewalld.service

# Enable firewall service to start on boot
sudo systemctl enable firewalld.service

# Add port 3389 for RDP to the firewall
sudo firewall-cmd --permanent --add-port=3389/tcp

# Reload firewall settings
sudo firewall-cmd --reload
```

#### 16. Enable GUI Package for RHEL9

```bash
# Set graphical.target as the default booting target
sudo systemctl set-default graphical.target

# Reboot the system to apply changes
sudo systemctl reboot
```

#### 17. RDP connection to the RHEL 9 server as GUI

Now, open an RDP tool on your Windows system and provide the public IPv4 address of the instance. Login using the username and password you have created earlier. In this case, the username is 'sanju', and the password is what you've set during user creation.

Ensure you wait for some time as the initial loading might be slow.

#### 13. Install Packages Related to Desktop or MATE Package/Distro

```bash
# Switch to root user
sudo su -

# Register with subscription manager or ensure subscription to Red Hat
# subscription-manager register [Provide username and password]

# Clean all unrelated or broken installation files and dependencies
dnf clean all

# Update services and dependencies
dnf update -y
```

#### 14. Adding user and adding the user to wheel group

```bash
# Add a new user with a home directory and set bash as the default shell
useradd -m -s /bin/bash sanju

# Set a password for the user
passwd sanju

# Add the user to the 'wheel' group for administrative privileges
usermod -aG wheel sanju
```

#### 15. Installing the services required for enabling RDP connection

```bash
# Install XRDP and TigerVNC server
dnf install xrdp tigervnc-server -y

# Start the XRDP service
systemctl start xrdp.service

# Enable the XRDP service to start on boot
systemctl enable xrdp.service

# Start the firewall service
systemctl start firewalld.service

# Enable the firewall service to start on boot
systemctl enable firewalld.service

# Add custom port 3389 for TCP protocol
firewall-cmd --permanent --add-port=3389/tcp

# Reload the firewall
firewall-cmd --reload
```

#### 16. Enable GUI Package for RHEL9

```bash
# Install the default GUI for RHEL9
dnf groupinstall "server with GUI" -y

# Set the default booting target to graphical
systemctl set-default graphical.target

# Reboot to reflect the settings
systemctl reboot
```

#### 17. RDP connection to the RHEL 9 server as GUI.

Now open an RDP tool on Windows and provide the public IPv4 address of the instance. Login using the user you have created, providing the username and password.

```plaintext
Username: Sanju
Password: [Your Chosen Password]
```

#### 18. Post-Connection Configuration

Once you have successfully connected to the RHEL 9 server via RDP, you may need to perform additional configuration tasks such as:

- Setting up user preferences and profiles.
- Configuring network settings.
- Installing additional software or packages as per your requirements.
- Customizing the desktop environment.
- Setting up security measures such as firewall rules and user access controls.

#### 19. Troubleshooting

In case you encounter any issues during the setup or connection process, you can troubleshoot by:

- Checking the system logs for error messages (`/var/log/messages`, `/var/log/syslog`).
- Verifying network connectivity and firewall configurations.
- Reviewing the RDP server configuration settings.
- Checking for any system updates or patches that may be required.
- Consulting online forums or documentation for solutions to common problems.

#### 20. Conclusion

Congratulations! You have successfully enabled GUI for RHEL 9 using RDP connection via AWS CLI. Your RHEL 9 server is now accessible through a graphical interface, providing a more user-friendly experience for managing your AWS environment.

### Contributions

If you have any feedback or suggestions for improving this guide, or if you encounter any issues that need to be addressed, please feel free to:

- Submit a pull request on GitHub to contribute to the guide.
- Open an issue to report any bugs or request additional features.
- Provide feedback directly to the author or maintainers of this guide.

Thank you for using this guide, and we hope it has been helpful in achieving your goals!

---

Feel free to add any specific troubleshooting steps, conclusion remarks, or feedback and contribution sections as needed. Let me know if there's anything else you'd like to include!
