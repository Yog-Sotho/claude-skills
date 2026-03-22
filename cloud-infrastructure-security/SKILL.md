---
name: cloud-infrastructure-security
description: Enforce cloud security best practices across AWS, Vercel, Railway, and Cloudflare deployments. Always activate this skill when the user is deploying to a cloud platform, writing or reviewing IaC (Terraform, CloudFormation, Pulumi), configuring IAM roles or policies, managing secrets, setting up CI/CD pipelines, configuring databases or storage, implementing logging/monitoring, or asking about cloud security — even if they don't explicitly mention security. When in doubt, activate. A missed security check is harder to fix than an unrequested one.
---

# Cloud & Infrastructure Security Skill

This skill ensures cloud infrastructure, CI/CD pipelines, and deployment configurations follow security best practices and comply with industry standards.

## Workflow

When this skill activates:

1. **Identify the scope** from user context — which cloud(s), which services, which stage (new build vs. review vs. pre-deploy).
2. **Navigate to the relevant section(s)** below. For broad requests, cover all applicable sections. For narrow ones (e.g., "set up CI/CD"), go straight to that section.
3. **Surface the applicable checklist and code patterns** for the user's specific stack. Adapt examples to the user's language/framework when possible.
4. **Run the pre-deployment checklist** any time a deployment is imminent — even if unsolicited. Cloud misconfigurations are the leading cause of data breaches; a quick checklist review is always worth it.
5. **Flag misconfigurations proactively.** If you spot an issue in code the user shares (an open security group, a public RDS instance, hardcoded credentials), call it out immediately — don't wait to be asked.

---

## 1. IAM & Access Control

The principle of least privilege is the single most impactful security control in cloud environments. Overly broad permissions are the root cause of most privilege escalation attacks.

### Least Privilege Policies

```yaml
# ✅ CORRECT: Minimal, resource-scoped permissions
iam_role:
  permissions:
    - s3:GetObject
    - s3:ListBucket
  resources:
    - arn:aws:s3:::my-bucket/*   # Specific bucket only

# ❌ WRONG: Wildcard permissions are a breach waiting to happen
iam_role:
  permissions:
    - s3:*
  resources:
    - "*"
```

### Service Accounts & OIDC

Prefer short-lived federated credentials over long-lived access keys. OIDC lets services like GitHub Actions assume roles without storing any secret.

```bash
# Create an OIDC identity provider for GitHub Actions
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

### MFA

```bash
# Enable MFA for privileged accounts — root and admin especially
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

### Checklist

- [ ] Root account not used in production; access keys deleted
- [ ] MFA enabled for all privileged accounts
- [ ] Service accounts use roles + OIDC, not long-lived credentials
- [ ] IAM policies follow least privilege (no `*` on resources or actions)
- [ ] Access reviews scheduled quarterly
- [ ] Unused credentials rotated or removed
- [ ] Permission boundaries set on delegated admin roles

---

## 2. Secrets Management

Secrets in code or environment variables are accidents waiting to happen — they end up in logs, error traces, and git history. Store them in a managed service with auditing and rotation.

### Cloud Secrets Managers

```typescript
// ✅ CORRECT: Fetch from secrets manager at runtime
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'prod/api-key' });
const apiKey = JSON.parse(secret.SecretString!).key;

// ❌ WRONG: env vars aren't rotated, audited, or access-controlled
const apiKey = process.env.API_KEY;
```

### Automatic Rotation

```bash
aws secretsmanager rotate-secret \
  --secret-id prod/db-password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

### Checklist

- [ ] All secrets in a managed store (AWS Secrets Manager, Vercel Secrets, GCP Secret Manager)
- [ ] Automatic rotation enabled for database credentials
- [ ] API keys rotated at least quarterly
- [ ] No secrets in source code, logs, or error messages
- [ ] Secret access audit logging enabled
- [ ] Secrets scoped to environments (prod secrets ≠ dev secrets)

---

## 3. Encryption

Encryption should be non-negotiable for any data touching production — at rest to protect against storage breaches, in transit to prevent interception.

### Encryption at Rest

```terraform
# ✅ CORRECT: Encrypted RDS instance
resource "aws_db_instance" "main" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.rds.arn
  # ... other config
}

# ✅ CORRECT: Encrypted S3 bucket (enforce via bucket policy)
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true
  }
}
```

### Encryption in Transit

```terraform
# ✅ CORRECT: Enforce HTTPS-only on S3 via bucket policy
resource "aws_s3_bucket_policy" "enforce_tls" {
  bucket = aws_s3_bucket.main.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:*"
      Resource  = ["${aws_s3_bucket.main.arn}/*", aws_s3_bucket.main.arn]
      Condition = { Bool = { "aws:SecureTransport" = "false" } }
    }]
  })
}

# Enforce TLS 1.2+ on RDS (parameter group)
resource "aws_db_parameter_group" "tls" {
  family = "postgres15"
  parameter {
    name  = "rds.force_ssl"
    value = "1"
  }
}
```

### Checklist

- [ ] All storage encrypted at rest (RDS, S3, EBS, EFS)
- [ ] Customer-managed KMS keys used for sensitive data
- [ ] TLS enforced in transit — HTTP rejected, minimum TLS 1.2
- [ ] Certificates managed via ACM (no self-signed certs in prod)
- [ ] KMS key rotation enabled

---

## 4. Network Security

Your network perimeter is the outermost layer of defense. Keep it tight — anything publicly exposed is an attack surface.

### VPC & Security Groups

```terraform
# ✅ CORRECT: Minimal-ingress security group
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # Internal VPC only
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]   # HTTPS outbound only
  }
}

# ❌ WRONG: Never open all ports to the internet
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Common Misconfigurations to Catch

```bash
# ❌ Public S3 bucket
aws s3api put-bucket-acl --bucket my-bucket --acl public-read

# ✅ Private bucket with explicit policy
aws s3api put-bucket-acl --bucket my-bucket --acl private
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

```terraform
# ❌ Publicly accessible RDS — extremely dangerous
resource "aws_db_instance" "bad" {
  publicly_accessible = true
}

# ✅ Private RDS in VPC
resource "aws_db_instance" "good" {
  publicly_accessible    = false
  vpc_security_group_ids = [aws_security_group.db.id]
  db_subnet_group_name   = aws_db_subnet_group.private.name
}
```

### Checklist

- [ ] No databases publicly accessible
- [ ] SSH/RDP restricted to VPN/bastion — never `0.0.0.0/0`
- [ ] Security groups scoped to minimum required ports and CIDRs
- [ ] Network ACLs as a second layer of defense
- [ ] VPC flow logs enabled for traffic visibility
- [ ] Private subnets for databases and internal services
- [ ] NAT gateway (not IGW) for private subnet egress

---

## 5. Logging & Monitoring

You can't defend what you can't see. Comprehensive logging is the foundation of incident detection, forensics, and compliance.

### CloudWatch / Structured Logging

```typescript
// ✅ CORRECT: Structured security event logging
import { CloudWatchLogsClient } from '@aws-sdk/client-cloudwatch-logs';

const logSecurityEvent = async (event: SecurityEvent) => {
  await cloudwatch.putLogEvents({
    logGroupName: '/aws/security/events',
    logStreamName: 'authentication',
    logEvents: [{
      timestamp: Date.now(),
      message: JSON.stringify({
        type: event.type,
        userId: event.userId,
        ip: event.ip,
        result: event.result,
        // Never log tokens, passwords, or PII
      })
    }]
  });
};
```

### Alerting

Set up alarms for high-signal security events. Don't wait to be breached before noticing.

```bash
# Alert on root account usage
aws cloudwatch put-metric-alarm \
  --alarm-name "RootAccountUsage" \
  --metric-name "RootAccountUsageCount" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789:SecurityAlerts
```

### Checklist

- [ ] CloudTrail enabled in all regions with log file validation
- [ ] CloudWatch Logs / equivalent enabled for all services
- [ ] Failed authentication attempts logged
- [ ] Admin and privileged actions audited
- [ ] Log retention ≥ 90 days (1 year for compliance workloads)
- [ ] Logs shipped to a separate, tamper-resistant account or bucket
- [ ] Alerts configured for: root usage, failed logins, config changes, IAM mutations

---

## 6. CI/CD Pipeline Security

Your pipeline has broad production access — it's a high-value target. Treat it with the same rigor as your production environment.

### Secure GitHub Actions Workflow

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read       # Minimal permissions — not write
      id-token: write      # Required for OIDC

    steps:
      - uses: actions/checkout@v4

      # Scan for accidentally committed secrets
      - name: Secret scanning
        uses: trufflesecurity/trufflehog@main

      # Catch vulnerable dependencies before they ship
      - name: Audit dependencies
        run: npm audit --audit-level=high

      # OIDC: no long-lived credentials stored anywhere
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1

      - name: Deploy
        run: ./scripts/deploy.sh
```

### Supply Chain Security

```json
{
  "scripts": {
    "install": "npm ci",
    "audit": "npm audit --audit-level=moderate",
    "check": "npm outdated"
  }
}
```

### Checklist

- [ ] OIDC used — no long-lived credentials in CI secrets
- [ ] Secret scanning in pipeline (TruffleHog, GitLeaks, or native)
- [ ] Dependency vulnerability scanning (npm audit, Dependabot, Snyk)
- [ ] Container image scanning if using Docker (Trivy, ECR scanning)
- [ ] Branch protection: require reviews + passing checks before merge
- [ ] Signed commits enforced on protected branches
- [ ] Pipeline IAM role scoped to minimum needed for deployment
- [ ] Artifact integrity verified before deploy (checksums / attestations)

---

## 7. CDN & Edge Security (Cloudflare)

Cloudflare is your outermost perimeter. Configure it correctly and you absorb DDoS, filter malicious traffic, and add headers before requests ever reach your origin.

### Security Headers via Workers

```typescript
export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);
    const headers = new Headers(response.headers);

    headers.set('Strict-Transport-Security', 'max-age=63072000; includeSubDomains; preload');
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    headers.set('Permissions-Policy', 'geolocation=(), microphone=(), camera=()');
    headers.set('Content-Security-Policy', "default-src 'self'; script-src 'self'");

    return new Response(response.body, { status: response.status, headers });
  }
};
```

### WAF Configuration

Enable via Cloudflare dashboard or Terraform (`cloudflare_ruleset`):
- OWASP Core Ruleset + Cloudflare Managed Ruleset
- Rate limiting on sensitive endpoints (auth, password reset)
- Bot Fight Mode
- Geo-blocking if your user base doesn't span high-risk regions

### Checklist

- [ ] WAF enabled with OWASP Core Ruleset + Cloudflare Managed Ruleset
- [ ] Rate limiting on auth endpoints (login, register, password reset)
- [ ] Bot protection active
- [ ] DDoS protection enabled (automatic in Cloudflare)
- [ ] Security headers configured (HSTS, CSP, X-Frame-Options)
- [ ] SSL/TLS mode set to "Full (strict)" — not "Flexible"
- [ ] Origin IP kept private (never expose directly)

---

## 8. Backup & Disaster Recovery

Backups are only valuable if they can actually restore you. Automate everything and test restores on a schedule.

### Automated Backups

```terraform
resource "aws_db_instance" "main" {
  allocated_storage    = 20
  engine               = "postgres"

  backup_retention_period = 30           # 30 days
  backup_window           = "03:00-04:00"
  maintenance_window      = "mon:04:00-mon:05:00"
  deletion_protection     = true         # Prevent accidental deletion

  enabled_cloudwatch_logs_exports = ["postgresql"]
}
```

### Checklist

- [ ] Automated daily backups configured for all stateful services
- [ ] Backup retention meets compliance requirements (often 90 days+)
- [ ] Point-in-time recovery enabled for databases
- [ ] Backups stored in a separate region or account
- [ ] Restore tested quarterly — untested backups are not backups
- [ ] RPO and RTO defined, documented, and validated
- [ ] Disaster recovery runbook exists and is up to date

---

## 9. Compliance (GDPR / HIPAA / SOC 2)

Compliance requirements vary by regulation. Flag them early — they affect architecture decisions (data residency, encryption key control, logging) that are painful to change later.

**GDPR**: Document all PII data flows. Implement data subject rights (access, erasure, portability). Sign DPAs with all processors. 72-hour breach notification to supervisory authority. Minimize data collected and retained.

**HIPAA**: BAAs with all vendors touching PHI. Audit log all PHI access. Minimum necessary access standard. PHI encrypted at rest and in transit (§3).

**SOC 2**: Formalize security policies, access reviews, and IR procedures. Collect evidence continuously (logs, change records, reviews). Vendor risk program.

Checklist:
- [ ] Applicable regulations identified and documented
- [ ] Data flows mapped; PII/PHI inventory complete
- [ ] Vendor agreements (DPAs / BAAs) in place
- [ ] Breach notification process defined and tested
- [ ] Data retention / deletion policy enforced

---

## 10. Incident Response

Having a plan before an incident is the difference between a recoverable event and a catastrophe.

**Steps**: Detect → Contain → Investigate → Eradicate → Recover → Post-mortem

```bash
# Immediately revoke a compromised access key
aws iam update-access-key \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive \
  --user-name compromised-user

# Lock out user entirely by attaching deny-all
aws iam put-user-policy \
  --user-name compromised-user \
  --policy-name EmergencyDenyAll \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Action":"*","Resource":"*"}]}'
```

Key principles: preserve logs before remediating (evidence!), rotate all credentials after containment, write a blameless post-mortem within 5 business days.

Checklist:
- [ ] On-call rotation defined with escalation path
- [ ] Runbook accessible (not gated behind a system that may be down)
- [ ] Credential revocation procedure documented and tested
- [ ] Tabletop exercise run at least annually

---

## Pre-Deployment Security Checklist

Run this before every production deployment:

- [ ] **IAM**: Root unused, MFA on privileged accounts, least privilege enforced
- [ ] **Secrets**: All secrets in a managed store with rotation enabled
- [ ] **Encryption**: Storage encrypted at rest; TLS enforced in transit
- [ ] **Network**: Security groups minimal; no public databases or unnecessary ports
- [ ] **Logging**: CloudTrail + CloudWatch enabled; retention configured; alerts active
- [ ] **CI/CD**: OIDC auth; secrets scanning; dependency audit passing
- [ ] **CDN/WAF**: Cloudflare WAF active; security headers set; TLS strict
- [ ] **Backups**: Automated and retention verified; restore tested recently
- [ ] **Compliance**: GDPR/HIPAA obligations met if applicable
- [ ] **Incident Response**: On-call defined; runbook accessible; escalation path clear
- [ ] **Documentation**: Infrastructure documented; credentials in a password manager

---

## Resources

- [AWS Security Best Practices](https://aws.amazon.com/security/best-practices/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [Cloudflare Security Documentation](https://developers.cloudflare.com/security/)
- [OWASP Cloud Security](https://owasp.org/www-project-cloud-security/)
- [Terraform Security Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
