![](Acvtivity%20image/screenshot.jpeg)


#Create a vpc to define your data center inside aws platform 
resource "aws_vpc" "my_vpc" {
  cidr_block       = "10.0.0.0/16"
  instance_tenancy = "default"

  tags = {
    Name = "Production"
  }
}

#Create a subnet inside vpc
resource "aws_subnet" "public1" {
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "Pub1-sub"
  }
}


resource "aws_subnet" "private1" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "Priv1-sub"
  }
}

resource "aws_subnet" "public2" {
  vpc_id                  = aws_vpc.my_vpc.id
  cidr_block              = "10.0.3.0/24"
  availability_zone       = "us-east-1b"
  map_public_ip_on_launch = true

  tags = {
    Name = "Pub2-sub"
  }
}

resource "aws_subnet" "private2" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.4.0/24"
  availability_zone = "us-east-1b"

  tags = {
    Name = "Priv2-sub"
  }
}

#Create a route table inside the custom vpc
resource "aws_route_table" "pub" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "pub-route"
  }
}

#Create route table association to seperate my public subnet from private subnet
resource "aws_route_table_association" "public1" {
  subnet_id      = aws_subnet.public1.id
  route_table_id = aws_route_table.pub.id
}

resource "aws_route_table_association" "public2" {
  subnet_id      = aws_subnet.public2.id
  route_table_id = aws_route_table.pub.id
}

#Create internet gateway inside the vpc
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "pub-IGW"
  }
}

#Create a security group for this vpc
resource "aws_security_group" "allow_traffic" {
  name        = "traffic"
  description = "Allow inbound traffic"
  vpc_id      = aws_vpc.my_vpc.id

  ingress {
    description = "HTTPS Traffic"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP Traffic"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "TLS from VPC"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web_pub"
  }
}

#Launch instance to the vpc defined 
resource "aws_instance" "web-server" {
  ami                         = "ami-00874d747dde814fa"
  instance_type               = "t2.micro"
  availability_zone           = "us-east-1a"
  key_name                    = "server"
  subnet_id                   = aws_subnet.public1.id
  associate_public_ip_address = true
  security_groups             = [aws_security_group.allow_traffic.id]

  provisioner "file" {
    source      = "script.sh"
    destination = "/home/ubuntu/script.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x script.sh",
      "sudo /home/ubuntu/script.sh args"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    host        = self.public_ip
    private_key = file("server.pem")
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ipaddress.txt"
  }

  tags = {
    Name = "web1"
  }
}

resource "aws_instance" "web-server2" {
  ami                         = "ami-00874d747dde814fa"
  instance_type               = "t2.micro"
  availability_zone           = "us-east-1b"
  key_name                    = "server"
  subnet_id                   = aws_subnet.public2.id
  associate_public_ip_address = true
  security_groups             = [aws_security_group.allow_traffic.id]

  provisioner "file" {
    source      = "script2.sh"
    destination = "/home/ubuntu/script2.sh"
  }

  provisioner "remote-exec" {
    inline = [
      "chmod +x script2.sh",
      "sudo /home/ubuntu/script2.sh args"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    host        = aws_instance.web-server2.public_ip
    private_key = file("server.pem")
  }

  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ipaddress.txt"
  }

  tags = {
    Name = "web2"
  }
}
