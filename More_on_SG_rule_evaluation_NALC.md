
---

## 🔐 Security Group Rule Evaluation — How AWS Applies Rules
AWS Security Groups follow a very specific behavior model. Understanding this is crucial for VPC Peering.

### 🔸 1. **No Rule Order**
Security Groups do **not** process rules in order. There is:
- no priority
- no first-match logic
- no last-match logic

AWS simply checks:
> “Does **any** rule allow this traffic?”
If yes → allow. If no → deny.

### 🔸 2. **Allow‑Only Firewall Model**
Security Groups contain **only allow rules**. There are **no deny rules**. Anything not explicitly allowed is denied.

### 🔸 3. **Stateful Behavior**
If outbound is allowed for a connection, the **return traffic is automatically allowed**, even if inbound rules do not explicitly allow it.

### 🔸 4. **Multiple SGs Merge Together**
If an EC2 has several SGs attached, AWS merges all rules into one giant permissions set.

### ✅ Summary
- SGs are **stateful**
- SGs are **allow-only**
- Rule **order does not matter**
- If **any** rule matches → traffic allowed
- If **no** rule matches → traffic denied

This behavior ensures that your VPC Peering setup works as long as each side has **at least one** inbound/outbound rule permitting the traffic.



# Visual Security Group (SG) Flow — VPC Peering Path

This diagram shows **how packets traverse** from the public EC2 to the private EC2 through **VPC Peering**, and **where SGs** and **route tables** apply. It also highlights the **stateful** nature of SGs.

## 🗺️ Flowchart (Mermaid)
```mermaid
flowchart TB
  %% VPC boundaries
  subgraph DEF_VPC[Default VPC</b>CIDR 172.31.0.0/16]
    PUB[Amazon EC2</b>nautilus-public-ec2]
    SG_OUT[AWS Security Group</b>Outbound rules</b>Allow to 10.1.0.0/16]
    RT_DEF[AWS Route Table</b>Route 10.1.0.0/16 → peering]
  end

  subgraph PRIV_VPC[nautilus-private-vpc</b>CIDR 10.1.0.0/16]
    RT_PRIV[AWS Route Table</b>Route 172.31.0.0/16 → peering]
    SG_IN[AWS Security Group</b>Inbound rules</b>Allow SSH or ICMP</b>Source public EC2 SG]
    PRIV[Amazon EC2</b>nautilus-private-ec2]
  end

  PCX[(Amazon VPC Peering</b>nautilus-vpc-peering)]

  %% Data path
  PUB --> SG_OUT
  SG_OUT --> RT_DEF
  RT_DEF --> PCX
  PCX --> RT_PRIV
  RT_PRIV --> SG_IN
  SG_IN --> PRIV

  %% Stateful return note
  WHY_STATEFUL[SGs are stateful</b>Replies automatically allowed</b>No extra inbound needed for responses]
  SG_OUT -. dashed .- WHY_STATEFUL
  SG_IN -. dashed .- WHY_STATEFUL

  %% Styling
  classDef vpc fill:#eef6ff,stroke:#3b82f6,color:#0b2e5b
  classDef ec2 fill:#fef9c3,stroke:#eab308,color:#3b2f04
  classDef rt fill:#dcfce7,stroke:#16a34a,color:#052e16
  classDef sg fill:#fde2e2,stroke:#ef4444,color:#4b0f0f
  classDef pcx fill:#e9d5ff,stroke:#7c3aed,color:#2e1065
  classDef why fill:#f5f5f5,stroke:#737373,color:#111827,stroke-dasharray: 5 5

  class DEF_VPC,PRIV_VPC vpc
  class PUB,PRIV ec2
  class RT_DEF,RT_PRIV rt
  class SG_OUT,SG_IN sg
  class PCX pcx
  class WHY_STATEFUL why
```

## 🔄 Sequence View (Mermaid)
```mermaid
sequenceDiagram
  participant U as Public EC2</b>nautilus-public-ec2
  participant SGU as SG outbound</b>allow to 10.1.0.0/16
  participant RTU as Route table</b>Default VPC
  participant PCX as VPC peering</b>nautilus-vpc-peering
  participant RTP as Route table</b>Private VPC
  participant SGP as SG inbound</b>allow SSH or ICMP from public EC2 SG
  participant P as Private EC2</b>nautilus-private-ec2

  U->>SGU: Packet to 10.1.1.x
  SGU->>RTU: Outbound allowed
  RTU->>PCX: Route via peering
  PCX->>RTP: Forward to private VPC
  RTP->>SGP: Deliver to inbound SG
  SGP->>P: Permit SSH or ICMP
  P-->>U: Response allowed statefully
```

---

### Key Takeaways
- **SGs are instance firewalls**: outbound on sender and inbound on receiver must allow the traffic.
- **Route tables provide the path**: each VPC must know the other’s CIDR via the peering connection.
- **Stateful SGs**: return traffic is automatically allowed once the initial outbound is permitted.



# Visual Security Group (SG) Flow — VPC Peering Path

This diagram shows **how packets traverse** from the public EC2 to the private EC2 through **VPC Peering**, and **where SGs** and **route tables** apply. It also highlights the **stateful** nature of SGs.

## 🗺️ Flowchart (Mermaid)
```mermaid
flowchart TB
  %% VPC boundaries
  subgraph DEF_VPC[Default VPC</b>CIDR 172.31.0.0/16]
    PUB[Amazon EC2</b>nautilus-public-ec2]
    SG_OUT[AWS Security Group</b>Outbound rules</b>Allow to 10.1.0.0/16]
    RT_DEF[AWS Route Table</b>Route 10.1.0.0/16 → peering]
  end

  subgraph PRIV_VPC[nautilus-private-vpc</b>CIDR 10.1.0.0/16]
    RT_PRIV[AWS Route Table</b>Route 172.31.0.0/16 → peering]
    SG_IN[AWS Security Group</b>Inbound rules</b>Allow SSH or ICMP</b>Source public EC2 SG]
    PRIV[Amazon EC2</b>nautilus-private-ec2]
  end

  PCX[(Amazon VPC Peering</b>nautilus-vpc-peering)]

  %% Data path
  PUB --> SG_OUT
  SG_OUT --> RT_DEF
  RT_DEF --> PCX
  PCX --> RT_PRIV
  RT_PRIV --> SG_IN
  SG_IN --> PRIV

  %% Stateful return note
  WHY_STATEFUL[SGs are stateful</b>Replies automatically allowed</b>No extra inbound needed for responses]
  SG_OUT -. dashed .- WHY_STATEFUL
  SG_IN -. dashed .- WHY_STATEFUL

  %% Styling
  classDef vpc fill:#eef6ff,stroke:#3b82f6,color:#0b2e5b
  classDef ec2 fill:#fef9c3,stroke:#eab308,color:#3b2f04
  classDef rt fill:#dcfce7,stroke:#16a34a,color:#052e16
  classDef sg fill:#fde2e2,stroke:#ef4444,color:#4b0f0f
  classDef pcx fill:#e9d5ff,stroke:#7c3aed,color:#2e1065
  classDef why fill:#f5f5f5,stroke:#737373,color:#111827,stroke-dasharray: 5 5

  class DEF_VPC,PRIV_VPC vpc
  class PUB,PRIV ec2
  class RT_DEF,RT_PRIV rt
  class SG_OUT,SG_IN sg
  class PCX pcx
  class WHY_STATEFUL why
```

## 🔄 Sequence View (Mermaid)
```mermaid
sequenceDiagram
  participant U as Public EC2</b>nautilus-public-ec2
  participant SGU as SG outbound</b>allow to 10.1.0.0/16
  participant RTU as Route table</b>Default VPC
  participant PCX as VPC peering</b>nautilus-vpc-peering
  participant RTP as Route table</b>Private VPC
  participant SGP as SG inbound</b>allow SSH or ICMP from public EC2 SG
  participant P as Private EC2</b>nautilus-private-ec2

  U->>SGU: Packet to 10.1.1.x
  SGU->>RTU: Outbound allowed
  RTU->>PCX: Route via peering
  PCX->>RTP: Forward to private VPC
  RTP->>SGP: Deliver to inbound SG
  SGP->>P: Permit SSH or ICMP
  P-->>U: Response allowed statefully
```

---

### Key Takeaways
- **SGs are instance firewalls**: outbound on sender and inbound on receiver must allow the traffic.
- **Route tables provide the path**: each VPC must know the other’s CIDR via the peering connection.
- **Stateful SGs**: return traffic is automatically allowed once the initial outbound is permitted.



# Troubleshooting SG Issues in VPC Peering

Use this checklist when connectivity from `nautilus-public-ec2` to `nautilus-private-ec2` fails.

## ✅ Quick Checks
- **Peering status** is `Active` and both VPCs are in the **same region**.
- **CIDRs do not overlap** (Default VPC `172.31.0.0/16`, Private VPC `10.1.0.0/16`).
- **Correct route tables** are associated with the **right subnets**.
- **Private IP** is used for tests (peering does not carry public IP traffic).

## 🧱 Security Group Pitfalls
1) **Outbound blocked on public EC2**
   - Fix: allow outbound to `10.1.0.0/16` or keep default `allow all outbound`.

2) **Inbound blocked on private EC2**
   - Fix: allow **SSH (22)** or **ICMP** **from public EC2’s SG**.

3) **Wrong source in inbound rule**
   - Fix: reference **SG** (`sg-<public-ec2-sg-id>`) instead of public IP.

4) **Multiple SGs but missing the right one**
   - Fix: attach the SG that has the needed rules; SG rules merge but must exist.

## 🧭 Routing Pitfalls
1) **Missing route on either side**
   - Default VPC: `10.1.0.0/16 → pcx: nautilus-vpc-peering`
   - Private VPC: `172.31.0.0/16 → pcx: nautilus-vpc-peering`

2) **Wrong route table association**
   - Ensure `nautilus-private-subnet` uses the route table that contains the peering route.

3) **Overlapping CIDRs**
   - Peering cannot be established when CIDRs overlap.

## 🧪 Testing Pitfalls
1) **Testing with public IP**
   - Use `10.1.1.x`, not the public IP.

2) **OS firewall blocking** (iptables, firewalld, ufw)
   - Allow SSH and ICMP on the instance OS.

3) **DNS issues**
   - Test using IP first; resolve hostnames via `/etc/hosts` or Route 53 after connectivity works.

## 🧰 Additional Checks
- **NACLs**: confirm they are not denying ephemeral ports; NACLs are **stateless** with **ordered allow/deny**.
- **IAM or SSM**: if using Session Manager instead of SSH, verify SSM agent and IAM policies.
- **Key pairs**: ensure the right SSH key and user (`ec2-user` for Amazon Linux).

## 🧑‍⚕️ Minimal Fix Plan
1) Confirm peering **Active**
2) Add **both** routes
3) Public EC2 SG: outbound **allow** to `10.1.0.0/16`
4) Private EC2 SG: inbound **SSH or ICMP** from **public EC2 SG**
5) Test with `ping` and `ssh` using **private IP**




# NACL vs Security Group (SG) — Side‑by‑Side Comparison

| Aspect | Security Group (SG) | Network ACL (NACL) |
|---|---|---|
| **Layer** | Instance level (ENI) | Subnet level |
| **Association** | Attach to **ENI/EC2**, ALB, RDS etc | Attach to **subnet** |
| **Statefulness** | **Stateful** (responses auto‑allowed) | **Stateless** (responses must be explicitly allowed) |
| **Rule Types** | **Allow only** | **Allow and Deny** |
| **Order / Priority** | **No order**; any matching rule allows | **Ordered** by rule number; first match wins |
| **Defaults** | Inbound: deny all</b>Outbound: allow all | Inbound and Outbound both have default rules; evaluate order |
| **Logging** | Use **VPC Flow Logs** to observe | Use **VPC Flow Logs**; NACL itself has no dedicated logs |
| **Granularity** | Per‑instance behavior | Per‑subnet behavior |
| **Typical Use** | Control access to instances (e.g., SSH, app ports) | Additional subnet guardrails, blocklists, or compliance controls |
| **Peering Relevance** | Required to allow traffic at endpoints | Ensure not blocking ephemeral return ports across subnets |
| **Common Gotchas** | Forget outbound or inbound rules; wrong SG source | Forget return path rules; deny rules shadow allows; wrong order |

## Notes
- SGs and NACLs **work together**: SGs control instance access; NACLs can permit or deny at subnet edges.
- For VPC peering demos, most issues are SG or **missing routes**; NACLs matter if custom deny/allow lists are in place.
