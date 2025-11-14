# Network Load Balancer: Private vs Public Deployment Guide

## Overview

Network Load Balancers (NLB) can be deployed in both **private** and **public** subnets, just like Application Load Balancers. This guide explains the differences and provides configuration examples for both scenarios.

## When to Use Each Type

### Public Network Load Balancer (`tests/nlb-public/`)

**Use when:**
- ✅ You need to accept traffic from the internet
- ✅ Your application serves external clients
- ✅ You need a static public IP address for internet-facing services
- ✅ You're exposing TCP/TLS services to public users

**Characteristics:**
- Deployed in **public subnets** (with Internet Gateway route)
- Gets public IP addresses
- Accessible from the internet
- DNS records in public hosted zones

**Example use cases:**
- Public-facing APIs over TCP/TLS
- Game servers (TCP/UDP)
- VPN endpoints (TLS)
- IoT device connections
- Public database connections (with proper security)

### Private Network Load Balancer (`tests/nlb-private/`)

**Use when:**
- ✅ You need internal load balancing within your VPC
- ✅ Your application serves internal microservices
- ✅ You want to keep traffic within private networks
- ✅ You're connecting internal services across availability zones

**Characteristics:**
- Deployed in **private subnets** (no direct internet access)
- Gets only private IP addresses
- Only accessible from within VPC or via VPN/Direct Connect
- DNS records in private hosted zones

**Example use cases:**
- Internal microservices communication
- Database connection pooling
- Internal API gateways
- Cross-AZ service discovery
- Private PrivateLink endpoints

## Configuration Comparison

### Public NLB Configuration

```hcl
# tests/nlb-public/terragrunt.hcl

locals {
  subnet_tier = "public"  # ← Public subnets
  zone_name   = "example.com"  # Public DNS zone
}

inputs = {
  load_balancer_type = "network"

  subnet_tags = {
    tier        = "public"  # ← Selects public subnets
    environment = "prod"
  }

  # No security groups on NLB itself
  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [{
    name      = "app-nlb"
    zone_name = "example.com"  # Public DNS
    type      = "A"
  }]

  listeners = {
    tcp_443 = {
      port     = 443
      protocol = "TCP"
      forward  = { target_group_key = "app" }
    }
    tcp_80 = {
      port     = 80
      protocol = "TCP"
      forward  = { target_group_key = "app-http" }
    }
  }

  target_groups = {
    app = {
      protocol = "TCP"
      port     = 443
      health_check = {
        protocol = "HTTPS"
        path     = "/health"
      }
    }
    app-http = {
      protocol = "TCP"
      port     = 80
      health_check = {
        protocol = "HTTP"
        path     = "/health"
      }
    }
  }
}
```

### Private NLB Configuration

```hcl
# tests/nlb-private/terragrunt.hcl

locals {
  subnet_tier = "private"  # ← Private subnets
  zone_name   = "internal.example.com"  # Private DNS zone
}

inputs = {
  load_balancer_type = "network"

  subnet_tags = {
    tier        = "private"  # ← Selects private subnets
    environment = "prod"
  }

  # No security groups on NLB itself
  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [{
    name      = "app-internal"
    zone_name = "internal.example.com"  # Private DNS
    type      = "A"
  }]

  listeners = {
    tcp_443 = {
      port     = 443
      protocol = "TCP"
      forward  = { target_group_key = "app" }
    }
    tls_8443 = {
      port            = 8443
      protocol        = "TLS"  # TLS termination
      certificate_arn = "arn:aws:acm:..."
      forward         = { target_group_key = "app-tls" }
    }
  }

  target_groups = {
    app = {
      protocol = "TCP"
      port     = 443
      health_check = {
        protocol = "TCP"  # Simple TCP health check
      }
    }
    app-tls = {
      protocol = "TCP"
      port     = 443
      health_check = {
        protocol = "HTTPS"
        path     = "/internal/alive"
      }
    }
  }
}
```

## Key Differences

| Aspect | Public NLB | Private NLB |
|--------|-----------|-------------|
| **Subnet Type** | Public subnets | Private subnets |
| **IP Addresses** | Public + Private IPs | Private IPs only |
| **Internet Access** | ✅ Direct from internet | ❌ Via VPN/Direct Connect only |
| **DNS Zone** | Public hosted zone | Private hosted zone |
| **Use Case** | Internet-facing services | Internal services |
| **Accessibility** | From anywhere | From VPC/connected networks |
| **Common Ports** | 80, 443 (internet standard) | Any port (internal flexibility) |
| **TLS Certificates** | Public CA certificates | Internal or public certificates |

## Subnet Selection

The module automatically selects the correct subnets based on the `subnet_tags` parameter:

```hcl
subnet_tags = {
  tier        = "public"   # or "private"
  environment = "prod"
}
```

This queries AWS to find subnets with matching tags in your VPC.

## Security Considerations

### Public NLB Security

1. **No Security Groups on NLB**: NLB doesn't support security groups on the load balancer itself
2. **Target Security Groups**: Configure security groups on your EC2 instances to:
   - Allow traffic from the VPC CIDR (NLB preserves source IP)
   - Or allow traffic from specific IP ranges
3. **Network ACLs**: Use subnet-level NACLs for additional protection
4. **TLS Encryption**: Use TLS listeners for encrypted traffic
5. **Source IP Preservation**: NLB preserves client IP, enable proper logging

Example target instance security group:
```hcl
resource "aws_security_group" "nlb_targets" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # NLB preserves source IP
  }
}
```

### Private NLB Security

1. **VPC-Only Access**: Traffic stays within VPC boundaries
2. **Target Security Groups**: Allow traffic from VPC CIDR or specific subnets
3. **Internal TLS**: Still use TLS for sensitive internal communications
4. **IAM Policies**: Control who can access the NLB via AWS APIs

Example target instance security group:
```hcl
resource "aws_security_group" "nlb_targets_private" {
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # Your VPC CIDR
  }
}
```

## DNS Configuration

### Public NLB DNS

```hcl
dns_records = [{
  name      = "api"                    # Creates: api.example.com
  zone_name = "example.com"            # Public Route53 zone
  type      = "A"
}]
```

**Result:** `api.example.com` → Public NLB endpoint

### Private NLB DNS

```hcl
dns_records = [{
  name      = "api-internal"           # Creates: api-internal.internal.example.com
  zone_name = "internal.example.com"   # Private Route53 zone
  type      = "A"
}]
```

**Result:** `api-internal.internal.example.com` → Private NLB endpoint (only resolvable within VPC)

## Testing Your NLB

### Testing Public NLB

```bash
# Test from your local machine
curl https://api.example.com

# Test TCP connection
telnet api.example.com 443

# Test TLS connection
openssl s_client -connect api.example.com:443
```

### Testing Private NLB

```bash
# Must be done from within VPC (EC2 instance, VPN, or Direct Connect)

# SSH into bastion host or EC2 in VPC
ssh -i key.pem ec2-user@bastion-host

# From within VPC
curl https://api-internal.internal.example.com

# Test TCP connection
telnet api-internal.internal.example.com 443
```

## Example Architectures

### Public NLB Architecture

```
Internet
   │
   ├─────────────┐
   │             │
[NLB Public]    [NLB Public]
   │             │
   │   Public    │
   │   Subnets   │
   │             │
   └─────┬───────┘
         │
   NAT   │  Internet
   GW    │  Gateway
         │
   ──────┴────────  (VPC Boundary)
         │
   ──────┬────────
         │
    Private Subnet
         │
   ┌─────┴─────┐
[Target] [Target]
   EC2     EC2
```

### Private NLB Architecture

```
VPC / VPN / Direct Connect
   │
   ├─────────────┐
   │             │
[NLB Private]  [NLB Private]
   │             │
   │   Private   │
   │   Subnets   │
   │             │
   └─────┬───────┘
         │
   ──────┴────────  (No Internet Access)
         │
   ┌─────┴─────┐
[Target] [Target]
   EC2     EC2
```

## Health Check Strategies

### For Public NLB

```hcl
health_check = {
  protocol = "HTTPS"              # Verify HTTPS endpoint works
  path     = "/health"            # Public health endpoint
  matcher  = "200"                # Expect 200 OK
  interval = 30                   # Check every 30 seconds
  healthy_threshold   = 3         # 3 successful checks to mark healthy
  unhealthy_threshold = 2         # 2 failed checks to mark unhealthy
}
```

### For Private NLB

```hcl
health_check = {
  protocol = "TCP"                # Simple TCP check (faster)
  port     = "443"
  interval = 10                   # Check more frequently (internal)
  healthy_threshold   = 2         # Faster recovery
  unhealthy_threshold = 2         # Faster detection
}

# Or use HTTP for better visibility
health_check = {
  protocol = "HTTP"
  path     = "/internal/health"   # Internal health endpoint
  matcher  = "200"
}
```

## Migration Between Private and Public

### From Private to Public NLB

1. **Update subnet tier:**
   ```hcl
   subnet_tags = {
     tier = "public"  # Changed from "private"
   }
   ```

2. **Update DNS zone:**
   ```hcl
   dns_records = [{
     zone_name = "example.com"  # Changed from "internal.example.com"
   }]
   ```

3. **Update target security groups** to allow internet traffic

4. **Request TLS certificate** for public domain (if needed)

5. **Apply changes:**
   ```bash
   terraform destroy  # Remove private NLB
   terraform apply    # Create public NLB
   ```

### From Public to Private NLB

1. **Update subnet tier:**
   ```hcl
   subnet_tags = {
     tier = "private"  # Changed from "public"
   }
   ```

2. **Update DNS zone:**
   ```hcl
   dns_records = [{
     zone_name = "internal.example.com"  # Changed from "example.com"
   }]
   ```

3. **Update target security groups** to restrict to VPC traffic

4. **Apply changes:**
   ```bash
   terraform destroy  # Remove public NLB
   terraform apply    # Create private NLB
   ```

## Complete Test Structure

```
tests/
├── private/              # ALB in private subnets (HTTP/HTTPS)
│   └── terragrunt.hcl
├── public/               # ALB in public subnets (HTTP/HTTPS)
│   └── terragrunt.hcl
├── nlb-private/          # NLB in private subnets (TCP/TLS) ⭐ NEW
│   └── terragrunt.hcl
└── nlb-public/           # NLB in public subnets (TCP/TLS) ⭐ NEW
    └── terragrunt.hcl
```

## Quick Reference

### Use Public NLB When:
- ✅ Serving internet users
- ✅ Need public static IPs
- ✅ External API/service endpoints
- ✅ Public game servers
- ✅ IoT device endpoints

### Use Private NLB When:
- ✅ Internal microservices
- ✅ Private database access
- ✅ VPC-to-VPC communication
- ✅ Internal service mesh
- ✅ PrivateLink endpoints

## Summary

Both **private** and **public** Network Load Balancer configurations are now fully supported, matching the ALB functionality. Choose based on your accessibility requirements:

- **Public NLB** = Internet-facing services with static public IPs
- **Private NLB** = Internal VPC services with private IPs only

Both configurations use the same TCP/TLS protocols and have identical capabilities—the only difference is network placement and accessibility! 🚀

