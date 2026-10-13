# chancelage.me

Source and infrastructure for [chancelage.me](https://chancelage.me), my portfolio site.

The page itself is one static HTML file. The interesting part is how it is served: every AWS resource behind it is defined in a single CloudFormation template, so the whole environment can be torn down and rebuilt from one command.

## Architecture

```
Visitor --> Route 53 --> CloudFront --> S3 (private)
                            ^
                     ACM certificate
```

- **Route 53** hosts the zone. Alias A and AAAA records point the apex and `www` at the distribution. Alias records are used because a CNAME is not allowed at the zone apex.
- **CloudFront** terminates TLS (minimum TLS 1.2) with an ACM certificate, caches at the edge, compresses responses, and serves HTTP/2 and HTTP/3. A CloudFront Function redirects `www` to the apex so there is one canonical URL.
- **S3** holds the files in a fully private bucket: all four public access blocks on, ACLs disabled, encryption at rest, versioning enabled. Website hosting mode is off.
- **Origin Access Control** lets CloudFront, and only this specific distribution, read from the bucket. The bucket policy pins access with an `AWS:SourceArn` condition, so another account's distribution pointed at the bucket gets nothing.
- **ACM** issues the certificate with DNS validation written automatically into the hosted zone. It lives in `us-east-1` because that is the only region CloudFront reads certificates from.

## Security headers

A custom CloudFront response headers policy replaces the AWS managed one, which does not set a Content-Security-Policy. The page loads no scripts, fonts, or third-party resources, so the CSP can be as strict as a working page allows:

```
default-src 'none'; style-src 'self' 'unsafe-inline'; img-src 'self' data:;
base-uri 'none'; form-action 'none'; frame-ancestors 'none'; object-src 'none';
upgrade-insecure-requests
```

Also set: HSTS (two years, includeSubDomains), `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, a strict referrer policy, a Permissions-Policy denying camera, microphone, geolocation and similar APIs, and same-origin COOP and CORP.

## Files

| File | Purpose |
|---|---|
| `template.yaml` | Every AWS resource, commented |
| `index.html` | The site |
| `404.html` | Custom not-found page, served for missing objects |
| `robots.txt`, `sitemap.xml` | Crawler hints |

## Deploying

From AWS CloudShell in `us-east-1`, with the domain already registered in Route 53:

```bash
HZ_ID=$(aws route53 list-hosted-zones-by-name --dns-name chancelage.me \
  --query "HostedZones[?Name=='chancelage.me.'].Id" --output text | sed 's|/hostedzone/||')

aws cloudformation deploy --region us-east-1 --stack-name chancelage-me \
  --template-file template.yaml \
  --parameter-overrides DomainName=chancelage.me HostedZoneId="$HZ_ID"
```

Then copy the site files into the bucket named in the stack outputs and invalidate the CloudFront cache.

## Cost

Roughly $0.50 a month for the hosted zone plus the yearly domain fee. S3, CloudFront, ACM and CloudFormation round to nearly nothing at portfolio traffic. A $5 monthly AWS Budget with an email alert guards against surprises.
