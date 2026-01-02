
# AWS ALB → EC2 (Nginx) in VPC — Architecture & Why
**Topic:** Application Load Balancer (ALB) in front of EC2 (Nginx)  \
**Scope:** VPC, Subnets, ALB, Target Group, Security Groups, Route Tables, NACLs, IGW/NAT  \

---

## 1) Colored Mermaid Diagram — Fully Nested & Correct Placement

> ALB lives **inside public subnets**; EC2 lives **inside private subnets**; everything lives **inside the VPC**. Security Groups attach to ENIs of the ALB and EC2.

```mermaid
flowchart TB

  %% ---------- External ----------
  Internet[(Internet)]
  Browser[Browser / Client]

  %% ---------- VPC (everything inside) ----------
  subgraph VPC["VPC (10.0.0.0/16) — networking boundary"]
    direction TB

    %% ---- Network components in VPC ----
    IGW["Internet Gateway"]
    NAT["NAT Gateway (optional)"]
    RT_Public["Public Route Table\n0.0.0.0/0 → IGW"]
    RT_Private["Private Route Table\n0.0.0.0/0 → NAT"]
    NACL_Public["NACL (Public)"]
    NACL_Private["NACL (Private)"]

    %% ---- Public Subnets (ALB lives here) ----
    subgraph PublicLayer["Public Subnets (for ALB)"]
      direction TB
      Pub1["Public Subnet A (10.0.1.0/24)"]
      Pub2["Public Subnet B (10.0.2.0/24)"]

      subgraph ALB["Application Load Balancer (xfusion-alb)"]
        ALB_DNS["ALB DNS Name"]
        ALB_SG["Security Group: xfusion-sg\nInbound 80 from 0.0.0.0/0"]
        Listener80["Listener :80 → forward to target group"]
        TG["Target Group: xfusion-tg\nType: instance • Port 80\nHealth check: HTTP / (200–399)"]
      end
    end

    %% ---- Private Subnets (EC2 lives here) ----
    subgraph PrivateLayer["Private Subnets (for EC2)"]
      direction TB
      Priv1["Private Subnet A (10.0.11.0/24)"]
      Priv2["Private Subnet B (10.0.12.0/24)"]

      subgraph EC2Block["EC2 Layer"]
        EC2["EC2: xfusion-ec2\nNginx listening on :80"]
        EC2_SG["EC2 Security Group\nAllow :80 only from ALB SG"]
      end
    end
  end

  %% ---------- Flow Edges ----------
  Internet --> Browser
  Browser -->|HTTP :80| ALB_DNS

  %% ALB placement & routing
  ALB_DNS --- Pub1
  ALB_DNS --- Pub2
  Pub1 --> RT_Public
  Pub2 --> RT_Public
  RT_Public --> IGW
  NACL_Public --- Pub1
  NACL_Public --- Pub2

  %% EC2 placement & routing
  EC2 --- Priv1
  Priv1 --> RT_Private
  RT_Private --> NAT
  NACL_Private --- Priv1
  NACL_Private --- Priv2

  %% SG associations
  ALB_SG --- ALB_DNS
  EC2_SG --- EC2

  %% Listener routing
  Listener80 --> TG
  TG -->|HTTP :80| EC2

  %% Health checks (label kept simple)
  ALB_DNS -->|Health check GET /| EC2

  %% ---------- Styles ----------
  classDef vpc fill:#e6f2ff,stroke:#4a90e2,color:#1b3a57,stroke-width:1px,rx:8,ry:8;
  classDef publicSubnet fill:#fff7cc,stroke:#d4b106,color:#5a4d00,rx:8,ry:8;
  classDef privateSubnet fill:#e8f8e8,stroke:#52c41a,color:#1f4d1f,rx:8,ry:8;

  classDef alb fill:#ffe6e6,stroke:#ff4d4f,color:#8b0000,rx:8,ry:8;
  classDef ec2 fill:#efe6ff,stroke:#9254de,color:#2b1957,rx:8,ry:8;

  classDef sg fill:#f2f2f2,stroke:#8c8c8c,color:#333,stroke-dasharray: 3 2,rx:8,ry:8;
  classDef rt fill:#e0fbfc,stroke:#00a0a0,color:#084c61,rx:8,ry:8;
  classDef nacl fill:#ffe9d6,stroke:#fa8c16,color:#7a3d00,rx:8,ry:8;
  classDef gw fill:#edf2f7,stroke:#718096,color:#2d3748,rx:8,ry:8;
  classDef external fill:#fafafa,stroke:#bfbfbf,color:#333,rx:8,ry:8;

  class VPC vpc;
  class PublicLayer,Pub1,Pub2 publicSubnet;
  class PrivateLayer,Priv1,Priv2 privateSubnet;

  class ALB,ALB_DNS,Listener80,TG alb;
  class EC2Block,EC2 ec2;

  class ALB_SG,EC2_SG sg;
  class RT_Public,RT_Private rt;
  class NACL_Public,NACL_Private nacl;
  class IGW,NAT gw;
  class Internet,Browser external;
```

---

## 2) Sequence Diagram — Request Path & Health Checks (Parser-Safe)

```mermaid
sequenceDiagram
  autonumber
  participant U as Browser / User
  participant A as ALB (xfusion-alb + xfusion-sg)
  participant T as Target Group (xfusion-tg)
  participant E as EC2 (xfusion-ec2 + EC2 SG)

  U->>A: HTTP GET http://ALB_DNS/ (port 80)
  A->>T: Forward per listener :80
  T->>E: HTTP :80 to instance target
  E-->>A: HTTP 200 OK (Nginx default page)
  A-->>U: HTTP 200 OK

  Note over A,T: ALB health checks - GET / on :80 (expects 200–399)
  Note over E: Ensure Nginx listens on :80 and returns 200 for /
```

---

## 3) Why Each Component (Interview-Ready)

### VPC
- **Why?** Logical network boundary containing subnets, route tables, SGs, NACLs, ALB, EC2.
- **Key rule:** ALB, TG, and EC2 must be in the **same VPC** to route traffic correctly.

### Subnets
- **Public (ALB)**: Needed for internet-facing ALB; public route table must send `0.0.0.0/0 → IGW`.
- **Private (EC2)**: Keeps servers off the public internet; only ALB reaches EC2 on port 80.
- **Multi-AZ:** ALB requires **2+ subnets in different AZs** for availability.

### Internet Gateway (IGW)
- **Why?** Provides internet connectivity for public subnets so the ALB can receive traffic.

### NAT Gateway (optional)
- **Why?** Allows private EC2 instances to make **outbound** internet calls (updates, package installs) without being publicly reachable.

### Route Tables
- **Public RT:** `0.0.0.0/0 → IGW` for ALB in public subnets.
- **Private RT:** `0.0.0.0/0 → NAT` (optional) for EC2 outbound.

### NACLs (stateless, subnet firewalls)
- Ensure inbound 80 + outbound ephemeral on public subnets for ALB; similarly permit flows on private subnets.

### Security Groups (stateful)
- **ALB SG (`xfusion-sg`)**: Inbound **80** from `0.0.0.0/0`; outbound allow all.
- **EC2 SG**: Inbound **80** **from source = ALB SG**; outbound allow all.
- **Why source SG vs CIDR?** IPs change; referencing the ALB SG is robust and secure.

### Application Load Balancer (ALB)
- **Why ALB vs NLB?** Layer 7 features (HTTP routing, path/host-based rules, headers, health checks). Ideal for web apps.
- **Listener:** HTTP :80 → default action: **forward to `xfusion-tg`**.

### Target Group (`xfusion-tg`)
- **Type:** **instance** targets (your EC2).
- **Port/Protocol:** HTTP :80.
- **Health check:** `GET /` expecting 200–399; ALB routes only to **healthy** targets.
##### A Target Group is a logical container that stores a list of backend targets.
>These targets receive traffic from:
```
ALB (Application Load Balancer)
NLB (Network Load Balancer)
GWLB (Gateway Load Balancer)
Lambda Function (special case)
IP addresses
Multiple ports (with dynamic port mapping using ECS)

A Target Group defines:
✔ WHERE to send the traffic
✔ HOW to check health of targets
✔ WHAT protocol/port/target type to use
✔ Routing behavior, depending on Load Balancer type
```
>***Think of it like the “backend servers list.”***
> Because load balancers cannot directly talk to EC2, ECS tasks, or IPs.
> They need an intermediate layer that defines: Type of Target, Port of Target, Protocol , Health check rules
> Without TG: Load balancer would not know which backend is alive/unhealthy.
> Supported TG types: instance ( EC2) , ip ( pod , container, on-prem-IP) , lamda ( ALB-to_Lamda)
> Network Load Balancer (NLB)  : Supported TG types: instance ( EC2) , ip ( pod , container, on-prem-IP) , ALB ( rare, used for chained load balancers)

### EC2 (`xfusion-ec2`)
- Runs Nginx on :80; reachable only via ALB.
- Prefer private subnets and avoid public IP exposure.

---

## 4) Minimal Checklist (to solve the assignment quickly)

- **ALB (xfusion-alb)**
  - Internet-facing; attach **xfusion-sg**; place in **two public subnets**.
  - Listener **HTTP :80** → forward to **xfusion-tg**.

- **Target Group (xfusion-tg)**
  - Type: **instance**; Port **80**; Health check **HTTP `/`**.
  - Register your EC2 instance target.

- **Security Groups**
  - **xfusion-sg (ALB):** Inbound TCP **80** from `0.0.0.0/0`.
  - **EC2 SG (default or custom):** Inbound TCP **80** from **source = xfusion-sg**.

- **Networking**
  - Public route table: `0.0.0.0/0 → IGW`.
  - Private route table: (optional) `0.0.0.0/0 → NAT`.
  - NACLs: allow required flows (80 inbound, ephemeral outbound).

- **Nginx**
  - Listens on **:80**; returns **200** for `/` (default page OK).

---

## 5) Common Pitfalls (and fixes)

- **ALB in private subnets** → must be public subnets with IGW route.
- **EC2 SG open to 0.0.0.0/0** → tighten to **source = ALB SG**.
- **Unhealthy targets** → ensure Nginx on :80, SG allows from ALB SG, NACLs permit flows, health path returns 200.
- **Wrong VPC** → ALB, TG, EC2 all in same VPC.
- **Single-AZ ALB** → use two subnets in different AZs.

---

## 6) Terraform Starter (names & ports you asked)

> Replace `vpc_id`, `public_subnet_ids`, and `ec2_instance_id` with your values.

```hcl
resource "aws_security_group" "xfusion_sg" {
  name        = "xfusion-sg"
  description = "Allow HTTP from the world to ALB"
  vpc_id      = var.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_lb_target_group" "xfusion_tg" {
  name        = "xfusion-tg"
  port        = 80
  protocol    = "HTTP"
  target_type = "instance"
  vpc_id      = var.vpc_id

  health_check {
    protocol = "HTTP"
    path     = "/"
    matcher  = "200-399"
  }
}

resource "aws_lb_target_group_attachment" "xfusion_attach" {
  target_group_arn = aws_lb_target_group.xfusion_tg.arn
  target_id        = var.ec2_instance_id
  port             = 80
}

resource "aws_lb" "xfusion_alb" {
  name               = "xfusion-alb"
  load_balancer_type = "application"
  internal           = false
  security_groups    = [aws_security_group.xfusion_sg.id]
  subnets            = var.public_subnet_ids
}

resource "aws_lb_listener" "xfusion_http" {
  load_balancer_arn = aws_lb.xfusion_alb.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.xfusion_tg.arn
  }
}

# Update default SG attached to EC2 to allow only ALB → EC2 HTTP
data "aws_security_group" "default_vpc_sg" {
  name   = "default"
  vpc_id = var.vpc_id
}

resource "aws_security_group_rule" "allow_http_from_alb_to_ec2" {
  type                     = "ingress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.xfusion_sg.id
  security_group_id        = data.aws_security_group.default_vpc_sg.id
}
```

---

## 7) Verification Steps

- **Get ALB DNS:** `aws elbv2 describe-load-balancers --names xfusion-alb --query 'LoadBalancers[0].DNSName' --output text`
- **Curl test:** `curl http://<ALB_DNS>` → Nginx default page.
- **Target health:** Check `xfusion-tg` shows **healthy**.

---

## 8) Credits & Notes

- Designed to be **parser-safe** for Mermaid on GitHub/VS Code.
- Color classes make tiers visually distinct for learning and interviews.
- Keep SG for EC2 restricted to **source = ALB SG** for defense-in-depth.

---

*© MohammadImran Khan — DevOps | AWS | Terraform*
