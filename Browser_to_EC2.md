
# Configuring a Public VPC with an EC2 Instance for Internet Access

> Goal: Reach an **EC2 instance directly from your browser via the public internet** (e.g., serve Nginx on port 80).

---

## 1) Architecture Diagrams (Mermaid) — **Corrected**

### A. Layered Components and Relationships (no special shapes that break parsing)
```mermaid

flowchart LR
  Internet[Internet]

  subgraph VPC["VPC</br>10.0.0.0/16"]
    IGW[IGW</br>Internet Gateway]
    RT[Route Table</br>Default route: 0.0.0.0/0 -> IGW]
    Subnet[Public Subnet</br>10.0.1.0/24]
    SG[Security Group</br>allow 80 and 443</br>SSH from your IP]
    NACL[Network ACL</br>inbound 80 and 443</br>outbound ephemeral]
    EC2[EC2 Instance</br>Public or Elastic IP]
    App["Web Server: Nginx/Apache/Node </br> Listening on 80 or 443"]

    IGW --> RT
    RT --> Subnet
    Subnet --> EC2
    SG -. allows 80/443 .-> EC2
    NACL --> Subnet
    EC2 --> App
  end

  Internet --> IGW
  App --> Internet

  classDef vpc fill:#E0F7FA,stroke:#006064,stroke-width:2px,color:#004d40;
  classDef subnet fill:#FFF3E0,stroke:#EF6C00,stroke-width:2px,color:#6D4C41;
  classDef ec2 fill:#E8F5E9,stroke:#2E7D32,stroke-width:2px,color:#1B5E20;
  classDef igw fill:#E3F2FD,stroke:#1565C0,stroke-width:2px,color:#0D47A1;
  classDef rt fill:#EDE7F6,stroke:#5E35B1,stroke-width:2px,color:#311B92;
  classDef sg fill:#FCE4EC,stroke:#C2185B,stroke-width:2px,color:#880E4F;
  classDef nacl fill:#F3E5F5,stroke:#8E24AA,stroke-width:2px,color:#4A148C;

  class VPC vpc;
  class Subnet subnet;
  class EC2 ec2;
  class IGW igw;
  class RT rt;
  class SG sg;
  class NACL nacl;
```
### B. Request Path (Sequence)
```mermaid
sequenceDiagram
  participant B as Browser
  participant I as Internet
  participant G as IGW
  participant R as Route Table
  participant S as Public Subnet
  participant E as EC2 (Public IP)
  participant A as Web App (Nginx/Apache)

  B->>I: HTTP GET /
  I->>G: Forward packet to VPC (public IP)
  G->>R: Enter VPC consult routing 
  R->>S: Route 0.0.0.0/0 -> IGW (association ensures path)
  S->>E: Deliver to instance
  E->>A: App receives request on 80/443
  A-->>B: Response returns via public IP path

  Note over E: SG must allow 80/443 (and 22 if SSH)
  Note over S: NACL permits inbound 80/443 and outbound ephemeral ports
```

---

## 2) Why Each Component Matters (the essential reasoning)

| Component | Why We Need It |
|----------|----------------|
| **VPC** | The isolated network where EC2 lives. |
| **Public Subnet** | A subnet is *public* only if its route table has `0.0.0.0/0 -> IGW`. |
| **Internet Gateway (IGW)** | Enables two-way internet connectivity for public IPs in the VPC. |
| **Route Table** | Holds the default outbound route to the IGW; associated with the public subnet. |
| **Public IP / Elastic IP** | Required so the internet can return traffic to the EC2 (routable address). |
| **Security Group (SG)** | Virtual firewall; must allow inbound `80/443` for web and typically `22` for SSH. |
| **Network ACL (NACL)** | Subnet-level stateless filter; must permit inbound 80/443 and outbound ephemeral. |
| **Web Server / App** | Something listening on `80/443` (e.g., Nginx/Apache/Node) to respond to browser requests. |

---

## 3) Step-by-Step Setup (Console)

1. **VPC**: `10.0.0.0/16`  
2. **Public Subnet**: `10.0.1.0/24` (associate with the public route table)  
3. **IGW**: Create and attach to VPC  
4. **Route Table**: Add `0.0.0.0/0 -> IGW` and associate to the subnet  
5. **EC2**: Launch in public subnet, **enable Public IP** (or Elastic IP)  
6. **SG**: Allow inbound `80/443`; SSH `22` from **your IP**  
7. **App**: Install Nginx/Apache and start the service  

---

## 4) Testing & Troubleshooting
- `curl -I http://<PUBLIC-IP>`
- Verify SG, NACL, route table association, and that the app is listening on `0.0.0.0:80`.

---

## 5) Optional: Minimal Terraform
```hcl
# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "my-public-vpc" }
}

# Public subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  tags = { Name = "public-subnet-1" }
}

# Internet Gateway
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "my-vpc-igw" }
}

# Route table + default route
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "public-rt" }
}

resource "aws_route" "default" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.igw.id
}

resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group for web
resource "aws_security_group" "web_sg" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["<YOUR_IP/32>"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 instance
resource "aws_instance" "web" {
  ami                           = "ami-xxxxxxxx"   # choose a valid AMI in your region
  instance_type                 = "t2.micro"
  subnet_id                     = aws_subnet.public.id
  vpc_security_group_ids        = [aws_security_group.web_sg.id]
  associate_public_ip_address   = true
  key_name                      = "my-key"         # optional for SSH
  tags = { Name = "public-web" }
}
```

---

### Final Checklist
- [ ] VPC + public subnet
- [ ] IGW attached
- [ ] Route table with `0.0.0.0/0 -> IGW` and association to subnet
- [ ] EC2 has public/Elastic IP
- [ ] SG allows `80/443` (SSH `22` from your IP)
- [ ] NACL permits inbound `80/443` & outbound ephemeral
- [ ] Web server is running

If all are true → **Browser → EC2 works**.
