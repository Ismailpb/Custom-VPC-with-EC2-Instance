# Terraform_VPC

Here is a simple project on how to create a custom vpc with ec2 instances via terraform. In this project we are creating 3 ec2 instances in our own custom VPC. One instance for database one for webserver and another one is for bastion server. For the security purpose we are creating DB server in private network.

## Prerequisites for this project
- Need a IAM user access with attached policies for the creation of VPC and EC2.
- Knowledge to the working principles of each AWS services especially VPC, EC2 and IP Subnetting.


## Features

- All Ec2 resource creations are done using terraform
- Each subnet CIDR block created automatically using subnetbit Function 
- AWS informations are defined using tfvars file and can easily changed 
- We can easily migrate the  infrastrucre to another region by changing provider details.
- All the resource names are appended with project name so we can easily identify the resources

## Terraform installtion 
You can easily download terraform using the following  official documentation provided by terraform.

https://www.terraform.io/downloads.html

Sample installation steps

```sh 
wget https://releases.hashicorp.com/terraform/0.15.3/terraform_0.15.3_linux_amd64.zip
unzip terraform_0.15.3_linux_amd64.zip 
ls -l
-rwxr-xr-x 1 root root 79991413 May  6 18:03 terraform  <<=======
-rw-r--r-- 1 root root 32743141 May  6 18:50 terraform_0.15.3_linux_amd64.zip
mv terraform /usr/bin/
which terraform 
/usr/bin/terraform
```
The next is configuration of AWS provider. So, we are creating a file named provider.tf for this purpose and mentioned our region name, IAM access key and seceret key in that file.

```sh 
provider "aws" {
  region     = region
  access_key = *************
  secret_key = ************
}
```
Now the next step is to create VPC
```sh 
terraform init
```
### Fetching AZ Names
```
data "aws_availability_zones" "az" {
    
  state = "available"

}
```
### VPC Creation
```
resource "aws_vpc" "vpc" {
    
  cidr_block           = var.vpc_cidr
  instance_tenancy     = "default"
  enable_dns_support   = true
  enable_dns_hostnames = true  
  tags = {
    Name = "${var.project}-vpc"
    project = var.project
  }
  lifecycle {
    create_before_destroy = true
  }
}
```

Once the VPC is created, we can now proceed with the craetion of Internet Gateway(IGW)

### Attaching Internet GateWay
 
```
resource "aws_internet_gateway" "igw" {
    
  vpc_id = aws_vpc.vpc.id
  tags = {
    Name = "${var.project}-igw"
    project = var.project
  }
    
}

```
In the next part, we need to subnets and here am going to create 3 public subnets and 3 private subnets

### Creating Subnets Public1

```
resource "aws_subnet" "public1" {
    
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = cidrsubnet(var.vpc_cidr,3,0)                        
  map_public_ip_on_launch  = true
  availability_zone        = data.aws_availability_zones.az.names[0]
  tags = {
    Name = "${var.project}-public1"
    project = var.project
  }
}
```
### Creating Subnets Public2
```
resource "aws_subnet" "public2" {
    
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = cidrsubnet(var.vpc_cidr,3,1)
  map_public_ip_on_launch  = true
  availability_zone        = data.aws_availability_zones.az.names[1]
  tags = {
    Name = "${var.project}-public2"
    project = var.project
  }
}
```
### Creating Subnets Public3
```
resource "aws_subnet" "public3" {
    
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = cidrsubnet(var.vpc_cidr,3,2)
  map_public_ip_on_launch  = true
  availability_zone        = data.aws_availability_zones.az.names[2]
  tags = {
    Name = "${var.project}-public3"
    project = var.project
  }
}
```
### Creating Subnets Private1
```
resource "aws_subnet" "private1" {
    
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = cidrsubnet(var.vpc_cidr,3,3)
  map_public_ip_on_launch  = false
  availability_zone        = data.aws_availability_zones.az.names[0]
  tags = {
    Name = "${var.project}-private1"
    project = var.project
  }
}
```
### Creating Subnets Private2
```
resource "aws_subnet" "private2" {
    
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = cidrsubnet(var.vpc_cidr,3,4)
  map_public_ip_on_launch  = false
  availability_zone        = data.aws_availability_zones.az.names[1]
  tags = {
    Name = "${var.project}-private2"
    project = var.project
  }
}
```
### Creating Subnets Private3
```
resource "aws_subnet" "private3" {
    
  vpc_id                   = aws_vpc.vpc.id
  cidr_block               = cidrsubnet(var.vpc_cidr,3,5)
  map_public_ip_on_launch  = false
  availability_zone        = data.aws_availability_zones.az.names[2]
  tags = {
    Name = "${var.project}-private3"
    project = var.project
  }
}
```
In order to configure private route table, we need to setup NAT Gateway and Elastic IP 
### Creating Nat GateWay
```
resource "aws_nat_gateway" "nat" {
    
  allocation_id = aws_eip.eip.id
  subnet_id     = aws_subnet.public2.id

  tags     = {
    Name    = "${var.project}-nat"
    project = var.project
  }

}
```
### Elastic IP Allocation
```
resource "aws_eip" "eip" {
  vpc      = true
  tags     = {
    Name    = "${var.project}-nat-eip"
    project = var.project
  }
}
```

Next we need to route the subnets for that we need to create the public and private route table and association

### RouteTable Creation public
```
resource "aws_route_table" "public" {
    
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags     = {
    Name    = "${var.project}-route-public"
    project = var.project
  }
}
```
### RouteTable Creation Private
```
resource "aws_route_table" "private" {
    
  vpc_id = aws_vpc.vpc.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.nat.id
  }

  tags     = {
    Name    = "${var.project}-route-private"
    project = var.project
  }
}
```
### RouteTable Association Subnet Public1  rtb public
```
resource "aws_route_table_association" "public1" {
  subnet_id      = aws_subnet.public1.id
  route_table_id = aws_route_table.public.id
}
```
### RouteTable Association Subnet Public2  rtb public
```
resource "aws_route_table_association" "public2" {
  subnet_id      = aws_subnet.public2.id
  route_table_id = aws_route_table.public.id
}
```
### RouteTable Association Subnet Public3  rtb public
```
resource "aws_route_table_association" "public3" {
  subnet_id      = aws_subnet.public3.id
  route_table_id = aws_route_table.public.id
}
```
### RouteTable Association Subnet Private1  rtb public
```
resource "aws_route_table_association" "private1" {
  subnet_id      = aws_subnet.private1.id
  route_table_id = aws_route_table.private.id
}
```
### RouteTable Association Subnet private2  rtb public
```
resource "aws_route_table_association" "private2" {
  subnet_id      = aws_subnet.private2.id
  route_table_id = aws_route_table.private.id
}
```
### RouteTable Association Subnet private3  rtb public
```
resource "aws_route_table_association" "private3" {
  subnet_id      = aws_subnet.private3.id
  route_table_id = aws_route_table.private.id
}
```

Now the creation of VPC is completed.


```sh
 terraform validate
```
- After successful validation, plan the build architecture and confirm the changes

```sh
 terraform plan
```
- Apply the changes to the AWS architecture

Then need to apply the below command

```sh
 terraform apply
```
### Creating Public Key

Here we are going to create a public key to access the instance. We can use the below codes in the files "02-01-infra-instance.tf"
```
resource "aws_key_pair"  "key" {
 
  key_name   = "terraform"
  public_key = file("terraform.pub")
  tags     = {
    Name    = "${var.project}-key"
    project = var.project
  }
}
```
Next we are going to create 3 security groups to assign to 3 instances.

### Creating Security Group - bastion
```
resource "aws_security_group" "bastion" {
    
  name        = "${var.project}-bastion"
  description = "Allows 22 traffic"
  vpc_id      = aws_vpc.vpc.id
  ingress     = [
      
  {
    description      = ""
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = [ "0.0.0.0/0" ]
    ipv6_cidr_blocks = [ "::/0" ]
    prefix_list_ids  = []
    security_groups  = []
    self             = false 
  }

  ]
      
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags     = {
    Name    = "${var.project}-bastion"
    project = var.project
  }
}
```

### Creating Security Group - frontend
```
resource "aws_security_group" "frontend" {   
  name        = "${var.project}-frontend"
  description = "Allows 80 from all,22 from bastion"
  vpc_id      = aws_vpc.vpc.id
  ingress     = [ 
      
  {
    description      = ""
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    security_groups  = [ aws_security_group.bastion.id ]
    prefix_list_ids  = []
    cidr_blocks      = []
    ipv6_cidr_blocks = []
    self             = false 
  },
  {
    description      = ""
    from_port        = 80
    to_port          = 80
    protocol         = "tcp"
    cidr_blocks      = [ "0.0.0.0/0" ]
    ipv6_cidr_blocks = [ "::/0" ]
    prefix_list_ids  = []
    security_groups  = []
    self             = false 
  },
  {
    description      = ""
    from_port        = 443
    to_port          = 443
    protocol         = "tcp"
    cidr_blocks      = [ "0.0.0.0/0" ]
    ipv6_cidr_blocks = [ "::/0" ]
    prefix_list_ids  = []
    security_groups  = []
    self             = false 
  }

  ]
      
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags     = {
    Name    = "${var.project}-frontend"
    project = var.project
  }
}
```


### Creating Security Group - backend
```
resource "aws_security_group" "backend" {
    
  name        = "${var.project}-backend"
  description = "Allows 3306 from frontend,22 from bastion"
  vpc_id      = aws_vpc.vpc.id
  ingress     = [ 
      
  {
    description      = ""
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    security_groups  = [ aws_security_group.bastion.id ]
    prefix_list_ids  = []
    cidr_blocks      = []
    ipv6_cidr_blocks = []
    self             = false 
  },
  {
    description      = ""
    from_port        = 3306
    to_port          = 3306
    protocol         = "tcp"
    security_groups  = [ aws_security_group.frontend.id ]
    prefix_list_ids  = []
    cidr_blocks      = []
    ipv6_cidr_blocks = []
    self             = false 
  }
  ]
      
  egress {
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }

  tags     = {
    Name    = "${var.project}-backend"
    project = var.project
  }
}
```

Once the public key and security groups were created, we can move to create Ec2 instances.

### Creating Ec2 Instance For Frontend

```
resource "aws_instance"  "frontend" {
    
  ami                          =  var.ami
  instance_type                =  var.type
  subnet_id                    =  aws_subnet.public1.id
  key_name                     =  aws_key_pair.key.id
  vpc_security_group_ids       =  [  aws_security_group.frontend.id ]
  user_data                    =  file("setup.sh")
  tags     = {
    Name    = "${var.project}-frontend"
    project = var.project
  }
    
}
```

### Creating Ec2 Instance For Backend
```
resource "aws_instance"  "backend" {
    
  ami                          =  var.ami
  instance_type                =  var.type
  subnet_id                    =  aws_subnet.private1.id
  key_name                     =  aws_key_pair.key.id
  vpc_security_group_ids       =  [  aws_security_group.backend.id  ]
  user_data                    =  file("setup.sh")
  tags     = {
    Name    = "${var.project}-backend"
    project = var.project
  }
    
}
```

### Creating Ec2 Instance For Bastion
```
resource "aws_instance"  "bastion" {
    
  ami                          =  var.ami
  instance_type                =  var.type
  subnet_id                    =  aws_subnet.public2.id
  key_name                     =  aws_key_pair.key.id
  vpc_security_group_ids       =  [  aws_security_group.bastion.id  ]
  user_data                    =  file("setup.sh")
  tags     = {
    Name    = "${var.project}-bastion"
    project = var.project
  }
    
}
```

### Setting Output

Here we are pushing the public IP's [Bastion Server,Frontend Server, Backend Server]  result to a file named "output.tf" so that we can easliy find out the public IP's of the servers.
```
output "frontend-public-ip" {
    
  value = aws_instance.frontend.public_ip
    
}


output "frontend-private-ip" {
    
  value = aws_instance.frontend.private_ip
    
}


output "bastion-public-ip" {
    
  value = aws_instance.bastion.public_ip
    
}



output "backend-public-ip" {
    
  value = aws_instance.backend.private_ip
    
}
```
Then we can use the below commands to complete the process

```
 terraform plan
```

And use
```
 terraform apply
```

To Obtain the output use the command as 

```
terraform output
```


⚙️ Connect with Me
 
  <a href="https://www.linkedin.com/in/ismail-pb-55373387/">
     <p> <img align="left" alt="Abhishek's LinkedIN" width="22px" src="https://raw.githubusercontent.com/peterthehan/peterthehan/master/assets/linkedin.svg" /> </p>
   </a>    
