# AWS VPC Peering - Student Project

**Name:** Dilli Prathap  
**Batch:** 1.1  
**Course:** EWB Courses - AWS Solutions Architect Associate  
**Region:** Asia Pacific (Mumbai) `ap-south-1`  
**Report date:** 30 October 2026  

A hands-on build of **two private networks (VPCs) in AWS connected with VPC Peering**. The project shows how a bastion host in a public subnet is used to reach private application servers, and how routing and security groups control which traffic may cross the peering link.

> The full PDF version of this report is also in this repository: [`DilliPrathap_EWB_VPC_Peering_Report.pdf`](DilliPrathap_EWB_VPC_Peering_Report.pdf)

---

## 1. Architecture

```
                         Internet
                            |
                  [ Internet Gateway ewb-igw-a ]
                            |
 +------------------------ VPC A 10.10.0.0/16 ---------------+      +------- VPC B 10.20.0.0/16 -------+
 |  Public subnet  ewb-a-public   10.10.1.0/24                |      |                                  |
 |     Bastion host  10.10.1.10  (public IPv4)                |      |  Private subnet ewb-b-private    |
 |                                                            |      |      10.20.1.0/24                |
 |  Private subnet ewb-a-private  10.10.2.0/24                |<====>|      App-B  10.20.1.10           |
 |     App-A  10.10.2.10  (no public IP)                      | pcx  |  (no internet gateway)           |
 +------------------------------------------------------------+      +----------------------------------+
```

- Only the **bastion** has a public IP. App servers are private.
- **VPC B has no internet gateway.**
- The two VPC ranges **do not overlap**, which is required for peering.

## 2. Address plan and resources

| Resource | CIDR / IP | AWS ID |
|---|---|---|
| ewb-vpc-a | 10.10.0.0/16 | vpc-0dc3eaaa48f8bd4ba |
| ewb-a-public | 10.10.1.0/24 | subnet-063e2fa071b39b60d |
| ewb-a-private | 10.10.2.0/24 | subnet-0d9f52d2a3d963220 |
| ewb-igw-a | attached to VPC A | igw-0ea88add97112536f |
| ewb-rt-a-public | 0.0.0.0/0 to IGW | rtb-0a0b81ff85754de10 |
| ewb-rt-a-private | local only | rtb-01f82e7a496913f21 |
| ewb-vpc-b | 10.20.0.0/16 | vpc-0a2d862236d5b99f8 |
| ewb-b-private | 10.20.1.0/24 | subnet-026d9fda11eea3237 |
| ewb-rt-b-private | local only | rtb-05280c9b35580f8ee |
| ewb-a-b-peer | VPC A <-> VPC B | pcx-0ee2feeb51a3dda9a |
| ewb-sg-app-a | SSH + ICMP echo | sg-062179dbba0becb82 |
| ewb-sg-app-b | SSH + ICMP echo | sg-01dbc66e1827a21a9 |
| ewb-bastion | 10.10.1.10 / public 43.205.229.108 | i-01a30032838ce244d |
| ewb-app-a | 10.10.2.10 | i-0ab37d64ad8477813 |

Tags used: `Project = EWB-VPC-Peering`, `Student = Dilli-Prathap`.

---

## 3. Build steps with evidence

### Step 1 - Create VPC A
Created `ewb-vpc-a` with CIDR `10.10.0.0/16`.

![VPC A created](01-vpc-a.jpg)

### Step 2 - Create the subnets in VPC A
Public subnet `ewb-a-public` (10.10.1.0/24) and private subnet `ewb-a-private` (10.10.2.0/24).

![Public subnet](02-subnet-public.jpg)
![Private subnet](03-subnet-private.jpg)

### Step 3 - Internet gateway
Created `ewb-igw-a` and attached it to VPC A only. Tags are visible in the screenshot.

![Internet gateway attached](04-igw-attached.jpg)

### Step 4 - Route tables for VPC A
- **Public route table:** local route plus `0.0.0.0/0` to the internet gateway, associated with `ewb-a-public`.
- **Private route table:** local route only (no internet route), associated with `ewb-a-private`.

![Public route table](05-route-table-public.jpg)
![Private route table](06-route-table-private.jpg)

### Step 5 - Create VPC B and its route table
`ewb-vpc-b` uses `10.20.0.0/16`. Its private subnet `ewb-b-private` is associated with `ewb-rt-b-private`.

![VPC B created](07-vpc-b.jpg)
![VPC B route table association](08-route-table-b-private.jpg)

### Step 6 - VPC peering connection
Requested `ewb-a-b-peer` from VPC A (requester) to VPC B (accepter), then accepted it. Status changed from *Pending acceptance* to **Active**.

![Peering requested](09-peering-requested.jpg)
![Peering active](10-peering-active.jpg)

### Step 7 - Security groups
Both application security groups allow **SSH (TCP 22)** and **ICMP Echo Request** only. No rule opens the servers to the whole internet.

![Security group App-A](11-sg-app-a.jpg)
![Security group App-B](12-sg-app-b.jpg)

### Step 8 - EC2 instances
- `ewb-bastion` in the public subnet with a public IPv4 address.
- `ewb-app-a` in the private subnet with **no public IP**.
Both passed 3/3 status checks.

![Bastion running](13-bastion-running.jpg)
![App-A running](14-app-a-running.jpg)

### Note on Application server B
When launching `ewb-app-b`, AWS displayed an account validation message for the region and blocked new launches until validation completes (it states this can take up to 4 hours). This is an AWS account-level restriction, not a VPC, subnet or security-group configuration issue.

![App-B launch message](15-app-b-launch-message.jpg)

---

## 4. How the routing works (design explanation)

| Item | Why it is needed |
|---|---|
| Local routes | Every route table has the VPC CIDR as `local`, so hosts in one VPC reach each other. |
| `ewb-rt-a-private`: 10.20.0.0/16 -> peering | Forward path. Without it, traffic from App-A to 10.20.1.10 has no matching route and is dropped. |
| `ewb-rt-b-private`: 10.10.0.0/16 -> peering | Return path. App-B needs a route back to 10.10.2.10 to reply. |
| `ewb-rt-a-public`: 10.20.0.0/16 -> peering | With SSH ProxyJump the bastion itself connects to App-B. The more specific /16 wins over `0.0.0.0/0`. |
| Active peering is not enough | Peering is only the link; route tables and security groups decide what may use it. |
| SSH source on App-B = bastion private IP | App-B sees 10.10.1.10, not the laptop's public IP. |
| ICMP Echo rules | Each app server accepts ping only from the other; security groups are stateful so replies are allowed. |
| No IGW on VPC B | Peering is non-transitive. VPC B cannot use VPC A's internet gateway. |

Route design for the peering:

| Route table | Destination | Target |
|---|---|---|
| ewb-rt-a-private | 10.10.0.0/16, 10.20.0.0/16 | local, pcx-0ee2feeb51a3dda9a |
| ewb-rt-b-private | 10.20.0.0/16, 10.10.0.0/16 | local, pcx-0ee2feeb51a3dda9a |
| ewb-rt-a-public | 10.10.0.0/16, 0.0.0.0/0, 10.20.0.0/16 | local, igw-0ea88add97112536f, pcx-0ee2feeb51a3dda9a |

---

## 5. Cleanup

After the work, all resources were removed to avoid charges: instances terminated, security groups deleted, peering connection deleted (with its routes), subnets and route tables deleted, internet gateway detached.

![Bastion termination](16-cleanup-bastion-terminate.jpg)
![Instances terminated](17-cleanup-instances-terminated.jpg)
![Security groups deleted](18-cleanup-security-groups.jpg)
![Peering deleted](19-cleanup-peering-deleted.jpg)
![Peering status Deleted](20-cleanup-peering-status.jpg)
![Subnets deleted](21-cleanup-subnets.jpg)
![Route tables deleted](22-cleanup-route-tables.jpg)
![Internet gateway detached](23-cleanup-igw-detached.jpg)

---

## 6. What I learned

- VPC CIDR ranges must not overlap if the VPCs are to be peered.
- A peering connection must be **accepted** and routes added **on both sides** before traffic can flow.
- Public vs private subnets differ only by their route table (a route to an internet gateway).
- A bastion host lets private servers be administered without giving them public IPs.
- Security groups are stateful and should allow only the specific sources needed.
- Peering is not transitive.
- Tag resources and delete them in the right order afterwards (instances, security groups, peering, subnets, route tables, internet gateway, VPCs).

## 7. Repository contents

| File | Description |
|---|---|
| `README.md` | This report |
| `DilliPrathap_EWB_VPC_Peering_Report.pdf` | Full PDF report |
| `01-...jpg` to `23-...jpg` | AWS Console screenshots used above |
