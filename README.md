# AWS VPC Peering Project

**EWB Cloud Labs - Project 01**
Student: Anirudh A Zalki | Course: AWS Cloud Computing | Batch: Evening 1 | Date: 25-10-2026
Region: Asia Pacific (Sydney) - `ap-southeast-2`

## Overview

This project connects two isolated AWS VPCs with a VPC peering connection so that two private EC2 servers can talk to each other using private IPv4 addresses only. A single public bastion host in VPC A is used for administration over SSH (ProxyJump).

## Architecture

```
Laptop --SSH/22--> Internet Gateway --> Public subnet A (Bastion 10.10.1.10)
                                              |
                              VPC A 10.10.0.0/16
                                |                         \
                Private subnet A (App-A 10.10.2.10)   Peering (pcx)
                                                              \
                                              VPC B 10.20.0.0/16
                                              Private subnet B (App-B 10.20.1.10)
```

## IP plan

| Resource | Name | CIDR / IP |
|---|---|---|
| VPC A | ewb-vpc-a | 10.10.0.0/16 |
| Public subnet A | ewb-a-public | 10.10.1.0/24 |
| Private subnet A | ewb-a-private | 10.10.2.0/24 |
| VPC B | ewb-vpc-b | 10.20.0.0/16 |
| Private subnet B | ewb-b-private | 10.20.1.0/24 |
| Bastion | ewb-bastion | 10.10.1.10 + public IPv4 |
| App-A | ewb-app-a | 10.10.2.10 (no public IP) |
| App-B | ewb-app-b | 10.20.1.10 (no public IP) |

Other resources: internet gateway `ewb-igw-a`, route tables `ewb-rt-a-public`, `ewb-rt-a-private`, `ewb-rt-b-private`, security groups `ewb-sg-bastion`, `ewb-sg-app-a`, `ewb-sg-app-b`, peering connection `ewb-a-b-peer`, key pair `ewb-lab`.

## Routing

| Route table | Destination | Target |
|---|---|---|
| ewb-rt-a-public | 10.10.0.0/16 | local |
| ewb-rt-a-public | 0.0.0.0/0 | internet gateway |
| ewb-rt-a-public | 10.20.0.0/16 | peering connection |
| ewb-rt-a-private | 10.10.0.0/16 | local |
| ewb-rt-a-private | 10.20.0.0/16 | peering connection |
| ewb-rt-b-private | 10.20.0.0/16 | local |
| ewb-rt-b-private | 10.10.0.0/16 | peering connection |

## Security group rules (inbound)

| Group | Rule | Source |
|---|---|---|
| ewb-sg-bastion | SSH 22 | my public IP /32 |
| ewb-sg-app-a | SSH 22 | 10.10.1.10/32 |
| ewb-sg-app-a | ICMP Echo request | 10.20.1.10/32 |
| ewb-sg-app-b | SSH 22 | 10.10.1.10/32 |
| ewb-sg-app-b | ICMP Echo request | 10.10.2.10/32 |

Outbound rules stay at the default allow-all.

## Tests performed

| # | Test | Expected |
|---|---|---|
| 1 | App-A ping App-B before peering | No replies |
| 2 | App-A ping App-B after routes | 0% packet loss |
| 3 | App-B ping App-A after routes | 0% packet loss |
| 4 | `ssh ewb-app-b` through the bastion | Shell on App-B |
| 5 | Public IPv4 on App-A and App-B | None |
| 6 | Remove ICMP rule on App-B, then restore it | Ping fails, then recovers |

## SSH access

Keys stay on the laptop. `~/.ssh/config` uses the bastion as a jump host:

```
Host ewb-bastion
    HostName <BASTION_PUBLIC_IP>
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
Host ewb-app-a
    HostName 10.10.2.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
Host ewb-app-b
    HostName 10.20.1.10
    User ec2-user
    IdentityFile ~/.ssh/ewb-lab.pem
    IdentitiesOnly yes
    ProxyJump ewb-bastion
```

## Key concepts

- Peering needs non-overlapping CIDRs, a request that is accepted, and routes on both sides.
- Peering is one-to-one and non-transitive.
- Security groups are stateful; the destination still needs an inbound rule.
- Peering does not give a VPC access to another VPC's internet gateway.

## Troubleshooting notes

- Bastion SSH timeout: check the bastion's current public IP and that the security group source matches the public IP your SSH traffic actually uses.
- Ping fails after peering: check the pcx route on both private route tables, the Echo request rule, and the host's outbound rules.

## Cleanup

Terminate the instances, delete peer routes and the peering connection, security groups, route tables, subnets, the internet gateway and both VPCs, then delete the key pair. Verify by the `Project = EWB-VPC-Peering` tag.

## Files

- `EWB-Courses-AWS-VPC-Peering-Student-Project.pdf` - project workbook
- `EWB_VPC_Peering_Report_Checklist.docx` - report and checklist
- `README.md` - this file

No private keys, passwords or tokens are stored in this repository.
