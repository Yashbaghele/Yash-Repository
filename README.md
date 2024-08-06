resource "aws_vpc" "VPC-3-tier" {
  cidr_block = "192.168.0.0/16"

  tags = {
    Name = "VPC-3-tier"
  }
}

resource "aws_subnet" "public" {
  vpc_id = aws_vpc.VPC-3-tier.id
  cidr_block = "192.168.1.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = true
  tags = {
    Name = "Public-Subnet-Nginx"
  }
}

resource "aws_subnet" "private-tom" {
  vpc_id = aws_vpc.VPC-3-tier.id
  cidr_block = "192.168.2.0/24"
  availability_zone = "ap-south-1b"
  map_public_ip_on_launch = false
  tags = {
    Name = "Private-Subnet-Tomcat"
  }
}

resource "aws_subnet" "private-db" {
  vpc_id = aws_vpc.VPC-3-tier.id
  cidr_block = "192.168.3.0/24"
  availability_zone = "ap-south-1a"
  map_public_ip_on_launch = false
  tags = {
    Name = "Private-Subnet-Database"
  }
}

resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.VPC-3-tier.id
  tags = {
    Name = "IGW-3-tier"
  }

}

resource "aws_route_table" "RT-public" {
  vpc_id = aws_vpc.VPC-3-tier.id
  tags = {
    Name = "RT-Public"
  }

  route  {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
}

resource "aws_route_table" "RT-private" {
  vpc_id = aws_vpc.VPC-3-tier.id
  tags = {
    Name = "RT-private"
  }

  route  {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.NAT-3-tier.id
    
  }
  
}

resource "aws_eip" "eip" {
  domain = "vpc"
}

resource "aws_nat_gateway" "NAT-3-tier" {
 allocation_id = aws_eip.eip.id
 subnet_id = aws_subnet.public.id

}

resource "aws_route_table_association" "RT-public" {
  subnet_id = aws_subnet.public.id
  route_table_id = aws_route_table.RT-public.id
}

resource "aws_route_table_association" "RT-private-tom" {
  subnet_id = aws_subnet.private-tom.id
  route_table_id = aws_route_table.RT-private.id
}

resource "aws_route_table_association" "RT-private-db" {
  subnet_id = aws_subnet.private-db.id
  route_table_id = aws_route_table.RT-private.id
}

resource "aws_security_group" "sg" {
  name = "tf-sg"
  vpc_id = aws_vpc.VPC-3-tier.id

  ingress  {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]

  } 

  ingress {
    from_port = 8080
    to_port   = 8080
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port  = 80
    to_port    = 80
    protocol   = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port = 3306
    to_port   = 3306
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }  
  
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

}

resource "aws_instance" "vm-Nginx" {
  ami = "ami-068e0f1a600cd311c"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.sg.id]
  key_name = "linux.pem"

  tags = {
    Name ="Nginx-Server-Public"
  }
}

resource "aws_instance" "vm-tom" {
  ami = "ami-068e0f1a600cd311c"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.private-tom.id
  vpc_security_group_ids = [aws_security_group.sg.id]
  key_name = "linux.pem"

  tags = {
    Name = "Tomcat-Server-Private"
  }
}

resource "aws_instance" "vm-db" {
  ami = "ami-068e0f1a600cd311c"
  instance_type = "t2.micro"
  subnet_id = aws_subnet.private-db.id
  vpc_security_group_ids = [aws_security_group.sg.id]
  key_name = "linux.pem"

  tags = {
    Name = "Database-Server-Private"
  }
  
}

resource "aws_db_subnet_group" "db-subnet" {
  name = "db-subnet"
  subnet_ids = [aws_subnet.private-tom.id,aws_subnet.private-db.id]
}

resource "aws_db_instance" "rds" {
  allocated_storage = 20
  db_name =  "database1"
  engine =  "mariadb"
  engine_version = "10.11.8"
  username = "admin"
  password = "Passwd123$"
  instance_class = "db.t3.micro"
  skip_final_snapshot = true
  db_subnet_group_name = aws_db_subnet_group.db-subnet.name

  vpc_security_group_ids = [aws_security_group.sg.id]

  tags = {
    Name ="DB-Instance"
  }
  
}


  










