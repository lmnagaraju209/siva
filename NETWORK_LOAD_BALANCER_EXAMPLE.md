# Network Load Balancer (NLB) Configuration Guide

## Overview

This document explains how to configure this module for Network Load Balancers (NLB) vs Application Load Balancers (ALB).

## Key Differences: ALB vs NLB

| Feature | Application Load Balancer (ALB) | Network Load Balancer (NLB) |
|---------|--------------------------------|----------------------------|
| **Listener Protocols** | HTTP, HTTPS | TCP, UDP, TCP_UDP, TLS |
| **Target Group Protocols** | HTTP, HTTPS | TCP, UDP, TCP_UDP, TLS |
| **Health Check Protocols** | HTTP, HTTPS | TCP, HTTP, HTTPS |
| **Security Groups** | Supported (attached to LB) | Not supported on LB (only on targets) |
| **Protocol Version** | HTTP1, HTTP2, GRPC | Not applicable |
| **idle_timeout** | Supported | Not supported |
| **enable_xff_client_port** | Supported | Not supported |

## Example Configuration: Network Load Balancer

### Basic NLB with TCP Listener

```hcl
module "nlb" {
  source = "path/to/module"

  project     = "my-app"
  environment = "prod"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  # Set load balancer type to network
  load_balancer_type = "network"

  subnet_tags = {
    tier        = "public"
    environment = "prod"
  }

  # Security group rules are ignored for NLB (NLB doesn't use security groups)
  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [
    {
      name      = "my-app-nlb"
      zone_name = "example.com"
      type      = "A"
    }
  ]

  # NLB Listeners - Use TCP or TLS protocols
  listeners = {
    tcp_443 = {
      port     = 443
      protocol = "TCP"
      forward = {
        target_group_key = "app_servers"
      }
    }
    tcp_80 = {
      port     = 80
      protocol = "TCP"
      forward = {
        target_group_key = "app_servers"
      }
    }
  }

  # NLB Target Groups - Use TCP or TLS protocols
  target_groups = {
    app_servers = {
      name              = "my-app-prod-tg"
      protocol          = "TCP"
      port              = 443
      target_type       = "instance"
      create_attachment = false
      
      health_check = {
        enabled             = true
        protocol            = "TCP"  # Can be TCP, HTTP, or HTTPS
        port                = "443"
        interval            = 30
        healthy_threshold   = 3
        unhealthy_threshold = 3
      }
      
      tags = {
        Application = "my-app"
      }
    }
  }

  enable_deletion_protection = true
}
```

### NLB with TLS Listener (SSL/TLS Termination)

```hcl
module "nlb_tls" {
  source = "path/to/module"

  project     = "my-app"
  environment = "prod"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  load_balancer_type = "network"

  subnet_tags = {
    tier        = "public"
    environment = "prod"
  }

  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [
    {
      name      = "my-app-nlb-tls"
      zone_name = "example.com"
      type      = "A"
    }
  ]

  # TLS Listener for NLB
  listeners = {
    tls_443 = {
      port            = 443
      protocol        = "TLS"
      certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/xxxxx"
      ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
      
      forward = {
        target_group_key = "app_servers"
      }
    }
  }

  # Target group with TCP protocol
  target_groups = {
    app_servers = {
      name              = "my-app-prod-tls-tg"
      protocol          = "TCP"  # Backend still uses TCP
      port              = 443
      target_type       = "instance"
      create_attachment = false
      
      health_check = {
        enabled             = true
        protocol            = "HTTPS"  # Can use HTTPS health check even with TCP target
        port                = "443"
        path                = "/health"
        interval            = 30
        healthy_threshold   = 3
        unhealthy_threshold = 3
      }
      
      tags = {
        Application = "my-app"
      }
    }
  }

  enable_deletion_protection = true
}
```

### NLB with HTTP Health Checks

```hcl
module "nlb_http_health" {
  source = "path/to/module"

  project     = "my-app"
  environment = "prod"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  load_balancer_type = "network"

  subnet_tags = {
    tier        = "private"
    environment = "prod"
  }

  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [
    {
      name      = "my-app-internal"
      zone_name = "internal.example.com"
      type      = "A"
    }
  ]

  listeners = {
    tcp_8080 = {
      port     = 8080
      protocol = "TCP"
      forward = {
        target_group_key = "web_servers"
      }
    }
  }

  target_groups = {
    web_servers = {
      name              = "my-app-web-tg"
      protocol          = "TCP"
      port              = 8080
      target_type       = "instance"
      create_attachment = false
      
      # HTTP health check for better application-level monitoring
      health_check = {
        enabled             = true
        protocol            = "HTTP"      # Use HTTP for health checks
        port                = "8080"
        path                = "/health"   # Required for HTTP health checks
        matcher             = "200"       # Expected HTTP status code
        interval            = 30
        healthy_threshold   = 3
        unhealthy_threshold = 3
        timeout             = 6
      }
      
      tags = {
        Application = "my-app"
        Component   = "web"
      }
    }
  }

  enable_deletion_protection = false
}
```

## Example Configuration: Application Load Balancer (ALB)

For comparison, here's how to configure an Application Load Balancer:

```hcl
module "alb" {
  source = "path/to/module"

  project     = "my-app"
  environment = "prod"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  # Set load balancer type to application (this is the default)
  load_balancer_type = "application"

  subnet_tags = {
    tier        = "public"
    environment = "prod"
  }

  # Security groups ARE supported for ALB
  security_group_ingress_rules = {
    http = {
      from_port   = 80
      to_port     = 80
      ip_protocol = "tcp"
      description = "HTTP Internet Traffic"
      cidr_ipv4   = "0.0.0.0/0"
    }
    https = {
      from_port   = 443
      to_port     = 443
      ip_protocol = "tcp"
      description = "HTTPS Internet Traffic"
      cidr_ipv4   = "0.0.0.0/0"
    }
  }

  security_group_egress_rules = {
    all = {
      ip_protocol = "-1"
      cidr_ipv4   = "0.0.0.0/0"
    }
  }

  dns_records = [
    {
      name      = "my-app"
      zone_name = "example.com"
      type      = "A"
    }
  ]

  # ALB Listeners - Use HTTP or HTTPS protocols
  listeners = {
    http_redirect = {
      port     = 80
      protocol = "HTTP"
      redirect = {
        port        = "443"
        protocol    = "HTTPS"
        status_code = "HTTP_301"
      }
    }
    https = {
      port            = 443
      protocol        = "HTTPS"
      certificate_arn = "arn:aws:acm:us-east-1:123456789012:certificate/xxxxx"
      ssl_policy      = "ELBSecurityPolicy-TLS13-1-2-2021-06"
      
      forward = {
        target_group_key = "app_servers"
      }
    }
  }

  # ALB Target Groups - Use HTTP or HTTPS protocols
  target_groups = {
    app_servers = {
      name              = "my-app-prod-alb-tg"
      protocol          = "HTTPS"
      protocol_version  = "HTTP1"
      port              = 443
      target_type       = "instance"
      create_attachment = false
      
      health_check = {
        enabled             = true
        protocol            = "HTTPS"
        port                = "443"
        path                = "/health"
        matcher             = "200"
        interval            = 30
        healthy_threshold   = 3
        unhealthy_threshold = 3
        timeout             = 6
      }
      
      tags = {
        Application = "my-app"
      }
    }
  }

  idle_timeout           = 60
  enable_xff_client_port = true
  enable_deletion_protection = true
}
```

## Migration Guide: ALB to NLB

If you're migrating from an Application Load Balancer to a Network Load Balancer, follow these steps:

### 1. Change Load Balancer Type
```hcl
load_balancer_type = "network"  # Change from "application"
```

### 2. Update Listener Protocols
**Before (ALB):**
```hcl
listeners = {
  https = {
    port     = 443
    protocol = "HTTPS"  # ❌ Not valid for NLB
    ...
  }
}
```

**After (NLB):**
```hcl
listeners = {
  tcp_443 = {
    port     = 443
    protocol = "TCP"    # ✅ Valid for NLB
    # Or use "TLS" for SSL/TLS termination
    ...
  }
}
```

### 3. Update Target Group Protocols
**Before (ALB):**
```hcl
target_groups = {
  app = {
    protocol          = "HTTPS"  # ❌ Not valid for NLB
    protocol_version  = "HTTP1"  # ❌ Not applicable for NLB
    ...
  }
}
```

**After (NLB):**
```hcl
target_groups = {
  app = {
    protocol          = "TCP"     # ✅ Valid for NLB
    protocol_version  = null      # ✅ Not used for NLB
    ...
  }
}
```

### 4. Update Health Check Configuration
**Before (ALB):**
```hcl
health_check = {
  protocol = "HTTPS"
  path     = "/health"
  matcher  = "200"
}
```

**After (NLB) - Option 1: TCP Health Check**
```hcl
health_check = {
  protocol = "TCP"    # Simple TCP connection check
  # path and matcher not needed for TCP
}
```

**After (NLB) - Option 2: HTTP/HTTPS Health Check**
```hcl
health_check = {
  protocol = "HTTPS"  # Can still use HTTP/HTTPS health checks with NLB
  path     = "/health"
  matcher  = "200"
}
```

### 5. Remove ALB-specific Parameters
Remove these parameters as they don't apply to NLB:
```hcl
# idle_timeout           = 60          # Not applicable to NLB
# enable_xff_client_port = true        # Not applicable to NLB
# protocol_version       = "HTTP1"     # Not applicable to NLB
```

### 6. Security Groups
NLBs don't support security groups attached to the load balancer itself. Security group rules apply to the target instances instead.

**Before (ALB):**
```hcl
security_group_ingress_rules = {
  https = {
    from_port   = 443
    to_port     = 443
    ip_protocol = "tcp"
    cidr_ipv4   = "0.0.0.0/0"
  }
}
```

**After (NLB):**
```hcl
# These are ignored for NLB but won't cause errors
security_group_ingress_rules = {}
security_group_egress_rules  = {}

# Instead, configure security groups on your target EC2 instances
# to allow traffic from the VPC CIDR or client IPs
```

## Common Use Cases

### Use ALB When:
- You need HTTP/HTTPS protocol features (headers, cookies, routing)
- You want path-based or host-based routing
- You need WAF integration
- You want to terminate SSL/TLS at the load balancer
- You need advanced request routing

### Use NLB When:
- You need ultra-low latency and high throughput
- You need static IP addresses
- You're load balancing non-HTTP protocols
- You need to preserve source IP addresses
- You want to handle millions of requests per second
- You're load balancing TCP/UDP traffic

## Troubleshooting

### Error: "InvalidConfigurationRequest: Listener protocol 'HTTP' is not supported for load balancer"

**Cause:** You're trying to use HTTP or HTTPS protocol with a Network Load Balancer.

**Solution:** Change the listener protocol to TCP, TLS, UDP, or TCP_UDP:
```hcl
listeners = {
  main = {
    protocol = "TCP"  # Change from "HTTP" or "HTTPS"
    ...
  }
}
```

### Error: "InvalidConfigurationRequest: Target group protocol 'HTTPS' is not supported for target group"

**Cause:** You're trying to use HTTP or HTTPS protocol for target groups with a Network Load Balancer.

**Solution:** Change the target group protocol to TCP or TLS:
```hcl
target_groups = {
  app = {
    protocol = "TCP"  # Change from "HTTP" or "HTTPS"
    ...
  }
}
```

### Error: "ValidationError: Health check protocol version is not supported"

**Cause:** You specified `protocol_version` for a Network Load Balancer target group.

**Solution:** Remove or set to null:
```hcl
target_groups = {
  app = {
    protocol_version = null  # Remove this line or set to null
    ...
  }
}
```

## Additional Resources

- [AWS NLB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/)
- [AWS ALB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [Comparison: ALB vs NLB](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)

