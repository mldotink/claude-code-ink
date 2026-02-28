---
name: ink-domains
description: Manage custom domains for an Ink (ml.ink) service. Use when the user wants to add, verify, or remove a custom domain.
argument-hint: "[domain]"
---

# Custom Domains on Ink

Help the user add or manage a custom domain for their Ink (ml.ink) service.

## Adding a custom domain

### 1. Identify the service and domain

Ask the user for:
- Which service to add the domain to (or use `mcp__ink__list_services` to list options)
- The domain name they want to use (or use `$ARGUMENTS` if provided)

### 2. DNS delegation (for full zone management)

If the user wants Ink to manage their DNS zone, use zone delegation:

```
mcp__ink__list_delegations  # Check existing delegations
```

To add a new zone:
- The user needs to update their domain's nameservers at their registrar to point to Ink's nameservers
- After updating, verify the delegation is working

### 3. Add the custom domain

```
mcp__ink__add_custom_domain(serviceName: "<service>", domain: "<domain>")
```

### 4. DNS configuration

If NOT using Ink DNS delegation, the user needs to add a CNAME record at their DNS provider:

```
CNAME  <domain>  →  <service-name>.ml.ink
```

For apex domains (e.g., `example.com` without `www`), they need an A record pointing to the Ink load balancer IP, or use a DNS provider that supports CNAME flattening (like Cloudflare).

### 5. Verify and confirm

- TLS certificates are provisioned automatically via Let's Encrypt
- It may take a few minutes for the certificate to be issued
- Test the domain by visiting `https://<domain>` in a browser

## Removing a custom domain

```
mcp__ink__remove_custom_domain(serviceName: "<service>", domain: "<domain>")
```

## Managing DNS records

If the user has delegated their zone to Ink:

```
mcp__ink__add_dns_record(zone: "<zone>", type: "A|AAAA|CNAME|MX|TXT|CAA", name: "<name>", content: "<value>", ttl: 3600)
mcp__ink__delete_dns_record(zone: "<zone>", recordId: "<id>")
mcp__ink__list_dns_records(zone: "<zone>")
```
