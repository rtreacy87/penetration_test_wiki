---
tags: [recon, recon/osint]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# Cloud Resources

Finding cloud-hosted assets belonging to a target: S3 buckets, Azure blob storage, GCP storage, and associated techniques for passive discovery.

## Overview

Cloud storage is a frequent source of data exposure and an increasingly common part of company infrastructure. AWS S3 buckets, Azure blobs, and GCP Cloud Storage are often set to public by accident or by deliberate but poorly considered design.

Even when correctly secured, finding cloud storage reveals which providers a company uses, can expose publicly readable content, and often leads to sensitive data in less-traveled buckets.

## Key Concepts / Techniques

### DNS-Based Cloud Discovery

Cloud storage endpoints appear in DNS. When iterating through subdomains and resolving their IP addresses, S3 origins reveal themselves:

```bash
for i in $(cat subdomainlist); do host $i | grep "has address" | grep inlanefreight.com | cut -d" " -f1,4; done
```

Example output showing AWS S3:
```
s3-website-us-west-2.amazonaws.com 10.129.95.250
```

### Google Dorks

Google indexes cloud storage that has public listing enabled or that links to files directly.

| Dork | Target |
|------|--------|
| `inurl:amazonaws.com intext:companyname` | AWS S3 buckets |
| `inurl:blob.core.windows.net intext:companyname` | Azure Blob Storage |
| `inurl:storage.googleapis.com intext:companyname` | GCP Storage |

These often surface PDF files, documents, presentations, and internal resources that were placed in cloud storage for easy sharing.

### Source Code Analysis

Cloud storage URLs are often embedded in web page source code for serving assets (images, JS, CSS). Inspecting page source and network requests reveals:
- CDN and storage bucket hostnames
- Internal asset paths

Common pattern: `<link rel="dns-prefetch" href="//companyname.blob.core.windows.net">`

### GrayHatWarfare

GrayHatWarfare (buckets.grayhatwarfare.com) indexes publicly accessible AWS, Azure, and GCP storage. Search by company name, file type, or bucket name pattern. Can be used to:
- Discover buckets by company name abbreviation
- Filter by file type to find specific categories of exposed files
- Find SSH private keys (`id_rsa`), configuration files, and other sensitive material that was inadvertently made public

### domain.glass

Provides infrastructure intelligence including Cloudflare protection status, IP relationships, and SSL cert details. Useful for understanding the security posture of specific domains (relevant to Layer 2 — Gateway analysis).

### Naming Conventions

Companies frequently use abbreviations or brand names as bucket prefixes. When testing, try variations:
- Full company name
- Ticker symbol or abbreviation
- Product/brand names
- Department abbreviations (`dev`, `staging`, `prod`, `backup`)

## Commands / Syntax

```bash
# Resolve subdomain list to find cloud provider IPs
for i in $(cat subdomainlist); do host $i | grep "has address" | cut -d" " -f1,4; done

# Attempt to list an S3 bucket (if public)
aws s3 ls s3://bucketname --no-sign-request

# Download all files from a public S3 bucket
aws s3 cp s3://bucketname . --recursive --no-sign-request

# Check Azure blob container listing
curl https://storageaccount.blob.core.windows.net/container?restype=container&comp=list
```

## Flags & Options

| Discovery Method | Signals |
|-----------------|---------|
| DNS resolution | IP resolves to known cloud provider CIDR |
| HTTP response | HTTP 403 = bucket exists but access denied; HTTP 200 = public access |
| GrayHatWarfare | Direct file listing with download links |
| Page source | `blob.core.windows.net`, `amazonaws.com`, `storage.googleapis.com` URLs |

## Gotchas & Notes

- A 403 response from a cloud storage URL confirms the bucket exists even when access is denied. Document it; access controls may change or other vectors may expose data.
- Cloud storage is often added to DNS for employee convenience — this is exactly how it becomes discoverable.
- SSH private keys have been found in public buckets on multiple engagements. Always search for `id_rsa` on GrayHatWarfare.
- Company name abbreviations are frequently used as bucket names. Always try variants.
- Cloud storage discovery is an OSINT activity — do not directly interact with the bucket in ways that could trigger alerts if scope is limited to passive.
- Even if a bucket requires authentication, its existence and naming convention can inform other attack paths.

## Related Pages

- [[recon/_overview]]
- [[recon/osint/domain_information]]
- [[recon/osint/staff_enumeration]]
- [[enumeration/dns]]

## Sources

- raw/footprinting/infastructure_based_enumeration_cloud_resources.md
