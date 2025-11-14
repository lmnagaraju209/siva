# Quick Start: Network Load Balancer

## The Problem

When creating a Network Load Balancer (NLB), AWS **does not support HTTP or HTTPS** as listener or target group protocols. You must use **TCP, UDP, TCP_UDP, or TLS** instead.

The previous code configuration used HTTP/HTTPS by default, which only works with Application Load Balancers (ALB).

## The Solution

The module now automatically handles differences between ALB and NLB. Simply set `load_balancer_type = "network"` and use the correct protocols.

## Minimal NLB Configuration

```hcl
module "my_nlb" {
  source = "./aws-lb-main"

  project     = "my-project"
  environment = "production"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  # 1. Set load balancer type to network
  load_balancer_type = "network"

  subnet_tags = {
    tier        = "public"
    environment = "production"
  }

  # 2. Remove security group rules (not applicable to NLB)
  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [{
    name      = "my-app"
    zone_name = "example.com"
    type      = "A"
  }]

  # 3. Use TCP protocol (not HTTP/HTTPS)
  listeners = {
    main = {
      port     = 443
      protocol = "TCP"  # ← Changed from "HTTPS"
      forward = {
        target_group_key = "app"
      }
    }
  }

  # 4. Target group must also use TCP protocol
  target_groups = {
    app = {
      name              = "my-app-tg"
      protocol          = "TCP"  # ← Changed from "HTTPS"
      port              = 443
      health_check = {
        protocol = "TCP"  # ← Changed from "HTTPS"
      }
    }
  }
}
```

## What Changed in the Code

### 1. `locals.tf`
- Added `is_nlb` flag to detect Network Load Balancer
- WAF is now automatically disabled for NLB

### 2. `main.tf`
- Security groups are conditionally created (ALB only)
- ALB-specific settings (`idle_timeout`, `enable_xff_client_port`) are ignored for NLB

### 3. `variables.tf`
- Removed hardcoded HTTPS defaults
- Added comprehensive protocol documentation
- Made protocol fields required (must be explicitly set)

## Protocol Reference

### Listener Protocols

| Load Balancer Type | Supported Protocols |
|-------------------|---------------------|
| Application (ALB) | HTTP, HTTPS |
| Network (NLB) | **TCP, TLS, UDP, TCP_UDP** |

### Target Group Protocols

| Load Balancer Type | Supported Protocols |
|-------------------|---------------------|
| Application (ALB) | HTTP, HTTPS |
| Network (NLB) | **TCP, TLS, UDP, TCP_UDP** |

### Health Check Protocols

| Load Balancer Type | Supported Protocols |
|-------------------|---------------------|
| Application (ALB) | HTTP, HTTPS |
| Network (NLB) | **TCP, HTTP, HTTPS** |

## Common Scenarios

### Scenario 1: TCP with TCP Health Check (Simplest)

```hcl
load_balancer_type = "network"

listeners = {
  tcp = {
    port     = 443
    protocol = "TCP"
    forward  = { target_group_key = "app" }
  }
}

target_groups = {
  app = {
    name     = "app-tg"
    protocol = "TCP"
    port     = 443
    health_check = {
      protocol = "TCP"  # Simple connection check
    }
  }
}
```

### Scenario 2: TLS with HTTPS Health Check (SSL Termination)

```hcl
load_balancer_type = "network"

listeners = {
  tls = {
    port            = 443
    protocol        = "TLS"
    certificate_arn = "arn:aws:acm:us-east-1:xxxxx:certificate/xxxxx"
    ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
    forward         = { target_group_key = "app" }
  }
}

target_groups = {
  app = {
    name     = "app-tg"
    protocol = "TCP"
    port     = 443
    health_check = {
      protocol = "HTTPS"  # Application-level health check
      path     = "/health"
      matcher  = "200"
    }
  }
}
```

### Scenario 3: TCP with HTTP Health Check (Best of Both)

```hcl
load_balancer_type = "network"

listeners = {
  tcp = {
    port     = 80
    protocol = "TCP"
    forward  = { target_group_key = "app" }
  }
}

target_groups = {
  app = {
    name     = "app-tg"
    protocol = "TCP"
    port     = 80
    health_check = {
      protocol = "HTTP"   # Application-level health check
      path     = "/health"
      matcher  = "200"
    }
  }
}
```

## Migration Checklist

If you're migrating from ALB to NLB, update these fields:

- [ ] Set `load_balancer_type = "network"`
- [ ] Change listener `protocol` from HTTP/HTTPS to TCP/TLS/UDP
- [ ] Change target group `protocol` from HTTP/HTTPS to TCP/TLS/UDP
- [ ] Update health check `protocol` to TCP, HTTP, or HTTPS
- [ ] Remove or set `protocol_version = null` in target groups
- [ ] Set `security_group_ingress_rules = {}`
- [ ] Set `security_group_egress_rules = {}`
- [ ] Set `waf_enabled = false` (or remove, it's automatic)
- [ ] Remove `idle_timeout` (or leave it, will be ignored)
- [ ] Remove `enable_xff_client_port` (or leave it, will be ignored)

## Error Messages and Solutions

### ❌ Error: "Listener protocol 'HTTP' is not supported"
**Fix:** Change listener protocol to `"TCP"` or `"TLS"`

### ❌ Error: "Target group protocol 'HTTPS' is not supported"
**Fix:** Change target group protocol to `"TCP"` or `"TLS"`

### ❌ Error: "Health check protocol version is not supported"
**Fix:** Remove `protocol_version` from target group or set to `null`

### ❌ Error: "ValidationError: path is required when protocol is HTTP"
**Fix:** Add `path = "/health"` to health_check when using HTTP/HTTPS

## Testing

See working examples in:
- `tests/nlb/terragrunt.hcl` - Complete NLB configuration
- `NETWORK_LOAD_BALANCER_EXAMPLE.md` - Detailed examples and explanations

## Need More Help?

- See `NETWORK_LOAD_BALANCER_EXAMPLE.md` for comprehensive examples
- See `README.md` for full module documentation
- Check AWS docs: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/

## Key Takeaway

**For Network Load Balancers:**
- ✅ Use TCP, TLS, UDP protocols
- ❌ Don't use HTTP, HTTPS protocols
- ✅ Health checks can be TCP, HTTP, or HTTPS
- ❌ No security groups on the NLB itself
- ❌ No WAF support

