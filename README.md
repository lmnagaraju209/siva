# AWS Load Balancer Terraform Module

This Terraform module creates and manages AWS Elastic Load Balancers. It supports both **Application Load Balancers (ALB)** and **Network Load Balancers (NLB)**.

## Features

- ✅ Support for both Application Load Balancers (ALB) and Network Load Balancers (NLB)
- ✅ Automatic protocol handling based on load balancer type
- ✅ Conditional security group creation (ALB only)
- ✅ WAF integration (ALB only)
- ✅ Route53 DNS record creation
- ✅ S3 access logging
- ✅ Flexible listener and target group configuration

## Usage

### Application Load Balancer (Default)

```hcl
module "alb" {
  source = "path/to/module"

  project     = "my-app"
  environment = "production"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  # ALB is the default type
  load_balancer_type = "application"

  subnet_tags = {
    tier        = "public"
    environment = "production"
  }

  security_group_ingress_rules = {
    https = {
      from_port   = 443
      to_port     = 443
      ip_protocol = "tcp"
      description = "HTTPS from Internet"
      cidr_ipv4   = "0.0.0.0/0"
    }
  }

  dns_records = [
    {
      name      = "app"
      zone_name = "example.com"
      type      = "A"
    }
  ]

  listeners = {
    https = {
      port            = 443
      protocol        = "HTTPS"
      certificate_arn = "arn:aws:acm:us-east-1:xxxxx:certificate/xxxxx"
      forward = {
        target_group_key = "main"
      }
    }
  }

  target_groups = {
    main = {
      name              = "my-app-tg"
      protocol          = "HTTPS"
      protocol_version  = "HTTP1"
      port              = 443
      health_check = {
        protocol = "HTTPS"
        path     = "/health"
        matcher  = "200"
      }
    }
  }
}
```

### Network Load Balancer

```hcl
module "nlb" {
  source = "path/to/module"

  project     = "my-app"
  environment = "production"
  vpc_id      = "vpc-xxxxx"
  region      = "us-east-1"

  # Set to network for NLB
  load_balancer_type = "network"

  subnet_tags = {
    tier        = "public"
    environment = "production"
  }

  # Security groups not applicable to NLB
  security_group_ingress_rules = {}
  security_group_egress_rules  = {}

  dns_records = [
    {
      name      = "app-nlb"
      zone_name = "example.com"
      type      = "A"
    }
  ]

  # Use TCP or TLS protocols for NLB
  listeners = {
    tcp_443 = {
      port     = 443
      protocol = "TCP"  # Use TCP, TLS, UDP, or TCP_UDP
      forward = {
        target_group_key = "main"
      }
    }
  }

  # Target groups must use TCP/TLS protocols
  target_groups = {
    main = {
      name              = "my-app-nlb-tg"
      protocol          = "TCP"  # Use TCP, TLS, UDP, or TCP_UDP (NOT HTTP/HTTPS)
      port              = 443
      health_check = {
        protocol = "TCP"  # Can be TCP, HTTP, or HTTPS
      }
    }
  }
}
```

## Key Differences: ALB vs NLB

| Feature | Application Load Balancer | Network Load Balancer |
|---------|--------------------------|----------------------|
| **Listener Protocols** | HTTP, HTTPS | TCP, TLS, UDP, TCP_UDP |
| **Target Group Protocols** | HTTP, HTTPS | TCP, TLS, UDP, TCP_UDP |
| **Health Check Protocols** | HTTP, HTTPS | TCP, HTTP, HTTPS |
| **Security Groups** | ✅ Supported | ❌ Not supported on LB |
| **Protocol Version** | HTTP1, HTTP2, GRPC | ❌ Not applicable |
| **WAF Integration** | ✅ Supported | ❌ Not supported |
| **idle_timeout** | ✅ Supported | ❌ Not supported |
| **enable_xff_client_port** | ✅ Supported | ❌ Not supported |

## Important Notes

### Network Load Balancer Limitations

When using `load_balancer_type = "network"`, the following limitations apply:

1. **Protocols**: You MUST use TCP, TLS, UDP, or TCP_UDP for listeners and target groups. HTTP and HTTPS are NOT supported.

2. **Security Groups**: NLBs don't support security groups attached to the load balancer. Configure security groups on your target instances instead.

3. **Protocol Version**: The `protocol_version` parameter (HTTP1, HTTP2, GRPC) is not applicable to NLB and should be set to `null` or omitted.

4. **WAF**: Web Application Firewall (WAF) is not supported with NLB. It's automatically disabled when using NLB.

5. **ALB-specific Settings**: Parameters like `idle_timeout` and `enable_xff_client_port` are ignored for NLB.

### Health Check Configuration

For NLB target groups, you have three options for health checks:

- **TCP**: Simple connection-based health check (no path or matcher required)
- **HTTP**: Application-level health check with path and status code
- **HTTPS**: Secure application-level health check with path and status code

Example:

```hcl
# TCP Health Check
health_check = {
  protocol = "TCP"
  port     = "443"
}

# HTTP Health Check
health_check = {
  protocol = "HTTP"
  port     = "80"
  path     = "/health"
  matcher  = "200"
}
```

## Examples

See the following example configurations in `tests/`:

- [Public Subnet Configuration](tests/public/terragrunt.hcl) - Works for both ALB and NLB (default: ALB)
- [Private Subnet Configuration](tests/private/terragrunt.hcl) - Works for both ALB and NLB (default: NLB)

Each configuration file includes:
- Active configuration for one load balancer type
- Commented examples showing how to switch to the other type
- Clear instructions on protocol requirements
- Security group handling for both ALB and NLB

For detailed NLB configuration guide, see [NETWORK_LOAD_BALANCER_EXAMPLE.md](NETWORK_LOAD_BALANCER_EXAMPLE.md)

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| load_balancer_type | Type of load balancer (application or network) | string | "application" | no |
| project | Project name for tagging | string | - | yes |
| environment | Environment name (dev, test, prod, etc.) | string | - | yes |
| vpc_id | VPC ID where load balancer will be created | string | - | yes |
| subnet_tags | Tags to identify subnets | object | - | yes |
| listeners | Map of listener configurations | any | {} | no |
| target_groups | Map of target group configurations | map(object) | {} | no |
| security_group_ingress_rules | Security group ingress rules (ALB only) | any | {} | no |
| security_group_egress_rules | Security group egress rules (ALB only) | any | {} | no |
| dns_records | List of DNS records to create | list(object) | - | yes |
| waf_enabled | Enable WAF (ALB only) | bool | true | no |
| enable_deletion_protection | Enable deletion protection | bool | true | no |
| idle_timeout | Connection idle timeout in seconds (ALB only) | number | 120 | no |
| enable_xff_client_port | Enable X-Forwarded-For client port (ALB only) | bool | false | no |
| access_logs | S3 access logs configuration | object | null | no |

## Outputs

| Name | Description |
|------|-------------|
| id | The ARN of the load balancer |
| arn | The ARN of the load balancer |
| dns_name | The DNS name of the load balancer |
| zone_id | The canonical hosted zone ID of the load balancer |

## Module Behavior

The module automatically adjusts its behavior based on the `load_balancer_type`:

### When `load_balancer_type = "application"`:
- Creates security groups and applies security rules
- Enables WAF (if configured)
- Applies ALB-specific settings (idle_timeout, enable_xff_client_port)
- Expects HTTP/HTTPS protocols

### When `load_balancer_type = "network"`:
- Skips security group creation (sets to null)
- Disables WAF
- Ignores ALB-specific settings
- Expects TCP/TLS/UDP protocols

## Troubleshooting

### Error: "Listener protocol 'HTTP' is not supported" or "Listener protocol 'HTTPS' must be one of 'TCP_UDP, UDP, TCP_QUIC, TLS, QUIC, TCP'"

**Problem**: You're trying to use HTTP/HTTPS protocols with a Network Load Balancer

**Identify the issue**: Check your ARN in the error message:
- `loadbalancer/net/...` = Network Load Balancer (requires TCP/TLS)
- `loadbalancer/app/...` = Application Load Balancer (requires HTTP/HTTPS)

**Solution 1 - Use Application Load Balancer (for HTTP/HTTPS traffic):**
```hcl
load_balancer_type = "application"

listeners = {
  https = {
    protocol = "HTTPS"  # ✅ Valid for ALB
    ...
  }
}

target_groups = {
  app = {
    protocol = "HTTPS"  # ✅ Valid for ALB
    protocol_version = "HTTP1"
    health_check = {
      protocol = "HTTPS"
      path = "/health"
    }
  }
}

# Security groups ARE used with ALB
security_group_ingress_rules = {
  https = {
    from_port = 443
    to_port = 443
    ip_protocol = "tcp"
    cidr_ipv4 = "0.0.0.0/0"
  }
}
```

**Solution 2 - Use Network Load Balancer with TCP/TLS:**
```hcl
load_balancer_type = "network"

listeners = {
  tcp = {
    protocol = "TCP"  # ✅ Valid for NLB
    ...
  }
}

target_groups = {
  app = {
    protocol = "TCP"  # ✅ Valid for NLB
    health_check = {
      protocol = "TCP"  # or "HTTP"/"HTTPS"
    }
  }
}

# Security groups NOT used with NLB
security_group_ingress_rules = {}
security_group_egress_rules = {}
```

### Error: "Target group protocol 'HTTPS' is not supported"

**Problem**: Using HTTP/HTTPS target group protocol with Network Load Balancer

**Solution**: Change protocols to TCP or TLS:
```hcl
target_groups = {
  main = {
    protocol = "TCP"  # Change from "HTTP" or "HTTPS"
    ...
  }
}
```

### Common Issues

1. **Forgot to set `load_balancer_type`**: Always explicitly set to "application" or "network"
2. **Mixed protocols**: Ensure listeners, target groups, and health checks all use compatible protocols
3. **Security groups on NLB**: Remove security group rules for NLB (configure on targets instead)
4. **Protocol version on NLB**: Remove `protocol_version` parameter for NLB target groups

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.0 |
| aws | >= 4.0 |

## License

This module is maintained by your organization.

## Contributing

When contributing to this module:

1. Ensure changes work for both ALB and NLB
2. Update examples for both load balancer types
3. Test with both load balancer types
4. Update documentation

## Support

For issues or questions:
- Check [NETWORK_LOAD_BALANCER_EXAMPLE.md](NETWORK_LOAD_BALANCER_EXAMPLE.md) for detailed NLB guidance
- Review example configurations in `tests/` directory
- Consult AWS documentation for specific protocol requirements

