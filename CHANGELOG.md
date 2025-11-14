# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).


## [1.2.0] - 2024-11-14

### Added
- Full support for Network Load Balancers (NLB) with proper protocol handling
- Automatic conditional logic for load balancer type-specific features
- Comprehensive documentation for NLB configuration (`NETWORK_LOAD_BALANCER_EXAMPLE.md`)
- Private vs Public deployment guide (`NLB_PRIVATE_PUBLIC_GUIDE.md`)
- Detailed README with ALB vs NLB comparison, usage examples, and troubleshooting
- Flexible test configurations that support both ALB and NLB in same file


### Changed
- **Simplified test structure**: Now just 2 directories (`tests/private/`, `tests/public/`) instead of 4
- Each test file includes both ALB and NLB configurations with easy switching via `load_balancer_type`
- `tests/private/` defaults to Network LB with commented ALB alternative
- `tests/public/` defaults to Application LB with commented NLB alternative
- Security groups are now conditionally created only for Application Load Balancers (not for NLB)
- WAF association is now automatically disabled for Network Load Balancers
- ALB-specific parameters (`idle_timeout`, `enable_xff_client_port`) are now ignored for NLB
- Updated `target_groups` variable defaults to be protocol-agnostic (removed hardcoded HTTPS defaults)
- Enhanced variable descriptions with protocol requirements for both ALB and NLB
- Simplified documentation structure, removed 5 transitional docs, kept 6 essential guides

### Fixed
- **BREAKING FIX**: Resolved issue where HTTP/HTTPS protocols were being used with Network Load Balancers, which don't support these protocols
- NLB now correctly expects TCP, TLS, UDP, or TCP_UDP protocols for listeners and target groups

### Documentation
- Added comprehensive migration guide from ALB to NLB
- Added troubleshooting section directly in README for common NLB configuration errors
- Added protocol comparison tables for ALB vs NLB
- Test configurations now show both ALB and NLB examples in same file with clear switching instructions
- Streamlined documentation: README, CHANGELOG, START_HERE, QUICK_START_NLB, NETWORK_LOAD_BALANCER_EXAMPLE, NLB_PRIVATE_PUBLIC_GUIDE

## [1.1.0] - 2023-10-31

### Changed
- Update module to support v9.x of upstream provider.  This migrated target groups and listeners to use maps instead of lists.  This prevents original behaviour of recreating resources when adding/removing configuration from anywhere besides the end of a list.
- Tests now use `terraform` config repo shared terragrunt.hcl.  This should move to us-east-2 eventually.

## [1.0.1] - 2023-10-23
### Fixed
- Issue with `var.target_groups`. `targets` mapping previously required all child objects, preventing association of a target group without having a standalone EC2 instance also associated.

## [1.0.0] - 2023-10-19
### Added
- Initial release.

