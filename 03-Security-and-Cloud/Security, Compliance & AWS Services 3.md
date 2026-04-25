--- START OF FILE Paste April 25, 2026 - 12:21PM ---

# Phase 6, Lesson 4: Network Security & AWS Security Services

---

## Why This Lesson Matters at NovaMart

```text
NovaMart processes $2B/year across 50M monthly active users.
That makes it a target for:

  - DDoS attacks (volumetric, protocol, application-layer)
  - Web application attacks (SQLi, XSS, SSRF, credential stuffing)
  - Data exfiltration (compromised service phoning home)
  - Lateral movement (attacker in one service reaching others)
  - Reconnaissance (port scanning, service enumeration)

The previous lesson secured WHAT runs in the cluster.
This lesson secures the NETWORK around, into, and within the cluster.

Three domains:
  1. PERIMETER DEFENSE — WAF, DDoS protection, CDN security
  2. NETWORK SEGMENTATION — VPC design, Security Groups, NACLs, NetworkPolicies
  3. DETECTION & RESPONSE — GuardDuty, Security Hub, Detective, CloudTrail analysis
```

```text
┌──────────────────────────────────────────────────────────────────────┐
│                   NOVAMART NETWORK SECURITY LAYERS                   │
│                                                                      │
│ INTERNET                                                             │
│    │                                                                 │
│    ▼                                                                 │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ LAYER 1: EDGE / PERIMETER                                        │ │
│ │ Cloudflare (CDN + DDoS + WAF L7)                                 │ │
│ │ Route53 (DNS, health checks, failover)                           │ │
│ └────────────────────────┬─────────────────────────────────────────┘ │
│                          │                                           │
│ ┌────────────────────────▼─────────────────────────────────────────┐ │
│ │ LAYER 2: AWS EDGE                                                │ │
│ │ AWS Shield Advanced (DDoS L3/L4)                                 │ │
│ │ AWS WAF (L7 rules on ALB)                                        │ │
│ │ ALB/NLB (TLS termination, access logs)                           │ │
│ └────────────────────────┬─────────────────────────────────────────┘ │
│                          │                                           │
│ ┌────────────────────────▼─────────────────────────────────────────┐ │
│ │ LAYER 3: VPC NETWORK                                             │ │
│ │ VPC (isolated network)                                           │ │
│ │ Subnets (public/private/data tier)                               │ │
│ │ NACLs (stateless subnet-level)                                   │ │
│ │ Security Groups (stateful instance-level)                        │ │
│ │ VPC Flow Logs (network telemetry)                                │ │
│ └────────────────────────┬─────────────────────────────────────────┘ │
│                          │                                           │
│ ┌────────────────────────▼─────────────────────────────────────────┐ │
│ │ LAYER 4: CLUSTER NETWORK                                         │ │
│ │ Kubernetes NetworkPolicies (pod-level L3/L4)                     │ │
│ │ Istio AuthorizationPolicy (L7, mTLS)                             │ │
│ │ Calico/Cilium policies (advanced, eBPF)                          │ │
│ └────────────────────────┬─────────────────────────────────────────┘ │
│                          │                                           │
│ ┌────────────────────────▼─────────────────────────────────────────┐ │
│ │ LAYER 5: DETECTION & RESPONSE                                    │ │
│ │ GuardDuty (threat detection)                                     │ │
│ │ Security Hub (centralized findings)                              │ │
│ │ CloudTrail (API audit)                                           │ │
│ │ VPC Flow Logs → Athena (network forensics)                       │ │
│ │ Detective (investigation)                                        │ │
│ └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Part 1: Perimeter Defense — DDoS Protection, WAF, CDN Security

### DDoS Attack Types and Defenses

```text
DDoS attacks target different layers of the stack:

LAYER 3/4 (VOLUMETRIC / PROTOCOL):
┌────────────────────┬──────────────────────┬──────────────────────────┐
│ Attack Type        │ How It Works         │ Defense                  │
├────────────────────┼──────────────────────┼──────────────────────────┤
│ UDP Flood          │ Massive UDP packets  │ Shield, Cloudflare       │
│                    │ overwhelm bandwidth  │ (absorb volume)          │
│ SYN Flood          │ Half-open TCP conns  │ SYN cookies,             │
│                    │ exhaust conn tables  │ Shield, NLB              │
│ DNS Amplification  │ Spoofed DNS queries  │ Shield, upstream         │
│                    │ reflect to target    │ filtering                │
│ NTP Amplification  │ Spoofed NTP monlist  │ Shield, upstream         │
│                    │ reflect to target    │ filtering                │
│ ICMP Flood         │ Ping flood           │ Rate limit, drop         │
└────────────────────┴──────────────────────┴──────────────────────────┘

Defense strategy:
  1. Absorb the volume (Cloudflare's 280+ Tbps network, AWS Shield)
  2. Filter at the edge (before traffic reaches your infrastructure)
  3. You CANNOT defend L3/L4 DDoS at the application level
     The pipe is full before your server sees a single packet

LAYER 7 (APPLICATION):
┌────────────────────┬──────────────────────┬──────────────────────────┐
│ Attack Type        │ How It Works         │ Defense                  │
├────────────────────┼──────────────────────┼──────────────────────────┤
│ HTTP Flood         │ Legitimate-looking   │ WAF rate                 │
│                    │ requests at high rate│ limiting                 │
│ Slowloris          │ Keep connections open│ Timeouts,                │
│                    │ with partial headers │ ALB handles              │
│ API Abuse          │ Expensive API calls  │ WAF rules,               │
│                    │ (search, checkout)   │ rate limits              │
│ Credential         │ Automated login      │ WAF + bot                │
│ Stuffing           │ attempts with leaked │ detection,               │
│                    │ credential databases │ CAPTCHA                  │
│ Web Scraping       │ Automated content    │ Bot mgmt,                │
│                    │ extraction at scale  │ fingerprinting           │
└────────────────────┴──────────────────────┴──────────────────────────┘

Defense strategy:
  1. Identify malicious traffic patterns (WAF rules)
  2. Rate limit per IP, per session, per API key
  3. Challenge suspicious clients (CAPTCHA, JavaScript challenge)
  4. L7 attacks look like legitimate traffic — harder to filter

KEY INSIGHT: L3/L4 defense = bandwidth. L7 defense = intelligence.
```

### AWS Shield

```text
AWS SHIELD STANDARD (Free, automatic):
  ✅ Protects ALL AWS resources automatically
  ✅ L3/L4 DDoS protection (SYN floods, UDP floods, reflection)
  ✅ Always-on detection and inline mitigation
  ❌ No L7 protection
  ❌ No DDoS Response Team access
  ❌ No cost protection (you pay for scaled-up resources)
  ❌ No visibility into attacks (no metrics/reports)

AWS SHIELD ADVANCED ($3,000/month + data transfer):
  ✅ Everything in Standard
  ✅ L7 DDoS protection (when combined with WAF)
  ✅ DDoS Response Team (DRT) — 24/7 AWS experts help during attack
  ✅ Cost protection — AWS credits you for DDoS-induced scaling
  ✅ Real-time metrics and attack visibility
  ✅ Automatic WAF rule creation during L7 attacks (SRT deploys rules)
  ✅ Health-based detection (uses Route53 health checks)
  ✅ Protects: CloudFront, Route53, ALB, NLB, EIP, Global Accelerator

NOVAMART USES SHIELD ADVANCED because:
  - $50K/minute outage cost makes $3K/month trivial
  - Cost protection alone justifies it (a large DDoS can generate
    $100K+ in bandwidth/scaling charges without Shield Advanced)
  - DRT access is invaluable during active attacks
  - Required for PCI compliance at NovaMart's scale
```

```hcl
# Shield Advanced + WAF + ALB integration (Terraform)

resource "aws_shield_protection" "alb" {
  name         = "novamart-alb-protection"
  resource_arn = aws_lb.main.arn

  tags = {
    Environment = "production"
  }
}

resource "aws_shield_protection" "cloudfront" {
  name         = "novamart-cloudfront-protection"
  resource_arn = aws_cloudfront_distribution.main.arn
}

resource "aws_shield_protection" "route53" {
  name         = "novamart-route53-protection"
  resource_arn = aws_route53_zone.main.arn
}

# Shield Advanced proactive engagement
# DRT automatically contacts you during detected events
resource "aws_shield_proactive_engagement" "main" {
  enabled = true

  emergency_contact {
    contact_notes = "Platform Engineering On-Call"
    email_address = "oncall-platform@novamart.com"
    phone_number  = "+1-555-0123"
  }

  emergency_contact {
    contact_notes = "VP of Engineering"
    email_address = "vp-eng@novamart.com"
    phone_number  = "+1-555-0456"
  }
}

# Shield Advanced DDoS automatic application-layer response
resource "aws_shield_application_layer_automatic_response" "alb" {
  resource_arn = aws_lb.main.arn
  action       = "COUNT"  # COUNT first, then switch to BLOCK after tuning
  # This automatically creates WAF rules during detected L7 DDoS
}
```

### AWS WAF — Web Application Firewall

```text
AWS WAF operates at Layer 7 (HTTP/HTTPS).
Attached to: ALB, CloudFront, API Gateway, AppSync, Cognito

WAF evaluates EVERY HTTP request against RULES.
Rules are organized in RULE GROUPS within a WEB ACL.

EVALUATION ORDER:
  Request arrives → WAF evaluates rules by priority (lowest number first)
  → First matching rule's action is taken (ALLOW, BLOCK, COUNT, CAPTCHA)
  → If no rule matches → Default action (ALLOW or BLOCK)

┌────────────────────────────────────────────────────────────────────┐
│                       WAF WEB ACL STRUCTURE                        │
│                                                                    │
│ Web ACL: novamart-production-waf                                   │
│ Default Action: ALLOW (block known-bad, allow everything else)     │
│                                                                    │
│ Priority 0: AWS Managed — AWSManagedRulesAmazonIpReputationList    │
│ Priority 1: AWS Managed — AWSManagedRulesCommonRuleSet (CRS)       │
│ Priority 2: AWS Managed — AWSManagedRulesSQLiRuleSet               │
│ Priority 3: AWS Managed — AWSManagedRulesKnownBadInputsRuleSet     │
│ Priority 4: AWS Managed — AWSManagedRulesBotControlRuleSet         │
│ Priority 5: Custom — Rate limiting (2000 req/5min per IP)          │
│ Priority 6: Custom — Geo-blocking (block specific countries)       │
│ Priority 7: Custom — URI path protection                           │
│ Priority 8: Custom — Request size constraints                      │
│                                                                    │
│ WCU (Web ACL Capacity Units): 1500 max per Web ACL                 │
│ Each rule costs WCUs. Managed rule groups cost 200-700 WCU each    │
└────────────────────────────────────────────────────────────────────┘
```

```hcl
# Full NovaMart WAF configuration (Terraform)

resource "aws_wafv2_web_acl" "main" {
  name        = "novamart-production-waf"
  description = "Production WAF for NovaMart ALB"
  scope       = "REGIONAL"  # REGIONAL for ALB, CLOUDFRONT for CloudFront
  # CloudFront WAF MUST be in us-east-1

  default_action {
    allow {}
  }

  # ─── MANAGED RULE GROUPS ───

  # IP Reputation — block known malicious IPs
  rule {
    name     = "aws-ip-reputation"
    priority = 0

    override_action {
      none {}  # Use rule group's actions as-is
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesAmazonIpReputationList"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-ip-reputation"
      sampled_requests_enabled   = true
    }
  }

  # Common Rule Set — OWASP Top 10 protections
  rule {
    name     = "aws-common-rules"
    priority = 1

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesCommonRuleSet"
        vendor_name = "AWS"

        # Exclude rules that cause false positives
        rule_action_override {
          name = "SizeRestrictions_BODY"
          action_to_use {
            count {}  # Count instead of block — file uploads are large
          }
        }
        rule_action_override {
          name = "CrossSiteScripting_BODY"
          action_to_use {
            count {}  # Count first — rich text editors trigger XSS rules
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-common-rules"
      sampled_requests_enabled   = true
    }
  }

  # SQL Injection protection
  rule {
    name     = "aws-sqli-rules"
    priority = 2

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesSQLiRuleSet"
        vendor_name = "AWS"
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-sqli-rules"
      sampled_requests_enabled   = true
    }
  }

  # Known Bad Inputs — Log4Shell, path traversal, etc.
  rule {
    name     = "aws-known-bad-inputs"
    priority = 3

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesKnownBadInputsRuleSet"
        vendor_name = "AWS"
        # This catches:
        # - Log4Shell (${jndi:ldap://...})
        # - Spring4Shell
        # - Path traversal (../../etc/passwd)
        # - Server-side includes
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-known-bad-inputs"
      sampled_requests_enabled   = true
    }
  }

  # Bot Control — detect and manage automated traffic
  rule {
    name     = "aws-bot-control"
    priority = 4

    override_action {
      none {}
    }

    statement {
      managed_rule_group_statement {
        name        = "AWSManagedRulesBotControlRuleSet"
        vendor_name = "AWS"

        managed_rule_group_configs {
          aws_managed_rules_bot_control_rule_set {
            inspection_level = "COMMON"
            # COMMON: Known bots (search engines, crawlers)
            # TARGETED: ML-based detection of sophisticated bots
            # TARGETED costs more and has higher WCU
          }
        }

        # Allow known good bots (Googlebot, etc.)
        rule_action_override {
          name = "CategoryVerifiedSearchEngine"
          action_to_use {
            allow {}
          }
        }
        rule_action_override {
          name = "CategoryVerifiedSocialMedia"
          action_to_use {
            allow {}
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "aws-bot-control"
      sampled_requests_enabled   = true
    }
  }

  # ─── CUSTOM RULES ───

  # Rate limiting — per IP
  rule {
    name     = "rate-limit-per-ip"
    priority = 5

    action {
      block {
        custom_response {
          response_code = 429
          custom_response_body_key = "rate-limited"
        }
      }
    }

    statement {
      rate_based_statement {
        limit              = 2000  # 2000 requests per 5-minute window per IP
        aggregate_key_type = "IP"

        scope_down_statement {
          not_statement {
            statement {
              # Don't rate-limit health check endpoints
              byte_match_statement {
                search_string         = "/health"
                positional_constraint = "STARTS_WITH"
                field_to_match {
                  uri_path {}
                }
                text_transformation {
                  priority = 0
                  type     = "LOWERCASE"
                }
              }
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "rate-limit-per-ip"
      sampled_requests_enabled   = true
    }
  }

  # Stricter rate limit for authentication endpoints
  rule {
    name     = "rate-limit-auth"
    priority = 6

    action {
      block {
        custom_response {
          response_code = 429
        }
      }
    }

    statement {
      rate_based_statement {
        limit              = 100  # 100 login attempts per 5 minutes per IP
        aggregate_key_type = "IP"

        scope_down_statement {
          byte_match_statement {
            search_string         = "/api/v1/auth"
            positional_constraint = "STARTS_WITH"
            field_to_match {
              uri_path {}
            }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
          }
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "rate-limit-auth"
      sampled_requests_enabled   = true
    }
  }

  # Geo-blocking — block countries with no NovaMart customers
  rule {
    name     = "geo-block"
    priority = 7

    action {
      block {}
    }

    statement {
      geo_match_statement {
        country_codes = ["KP", "IR", "SY", "CU"]
        # Sanctioned countries — also a compliance requirement
        # OFAC (Office of Foreign Assets Control) sanctions
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "geo-block"
      sampled_requests_enabled   = true
    }
  }

  # Request size limit — prevent oversized payloads
  rule {
    name     = "request-size-limit"
    priority = 8

    action {
      block {}
    }

    statement {
      size_constraint_statement {
        comparison_operator = "GT"
        size                = 10485760  # 10MB
        field_to_match {
          body {
            oversize_handling = "MATCH"  # Block if body > WAF inspection limit
          }
        }
        text_transformation {
          priority = 0
          type     = "NONE"
        }
      }
    }

    visibility_config {
      cloudwatch_metrics_enabled = true
      metric_name                = "request-size-limit"
      sampled_requests_enabled   = true
    }
  }

  # Custom response body for rate limiting
  custom_response_body {
    key          = "rate-limited"
    content      = "{\"error\": \"Rate limit exceeded. Please retry after 5 minutes.\"}"
    content_type = "APPLICATION_JSON"
  }

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "novamart-production-waf"
    sampled_requests_enabled   = true
  }

  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
  }
}

# Associate WAF with ALB
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}

# WAF logging to S3 (via Kinesis Firehose)
resource "aws_wafv2_web_acl_logging_configuration" "main" {
  log_destination_configs =[aws_kinesis_firehose_delivery_stream.waf_logs.arn]
  resource_arn            = aws_wafv2_web_acl.main.arn

  # Only log blocked requests and sampled allowed requests
  logging_filter {
    default_behavior = "DROP"  # Don't log everything (cost)

    filter {
      behavior    = "KEEP"
      requirement = "MEETS_ANY"

      condition {
        action_condition {
          action = "BLOCK"
        }
      }
      condition {
        action_condition {
          action = "COUNT"
        }
      }
    }
  }

  # Redact sensitive headers from logs
  redacted_fields {
    single_header {
      name = "authorization"
    }
    single_header {
      name = "cookie"
    }
  }
}
```

### WAF Deployment Strategy — The COUNT-First Pattern

```text
NEVER deploy WAF rules in BLOCK mode immediately.
This is the WAF equivalent of Gatekeeper's dryrun → warn → deny.

PHASE 1: COUNT MODE (Week 1-2)
  Set all rules to COUNT (observe, don't block)
  Analyze WAF logs:
    - Which rules are triggering?
    - Are any rules matching legitimate traffic? (false positives)
    - What's the baseline request rate?
  
  CRITICAL: Rich text editors, file uploads, API endpoints with
  complex payloads WILL trigger XSS and size rules as false positives.
  Identify and exclude these BEFORE enabling BLOCK.

PHASE 2: SELECTIVE BLOCK (Week 3-4)
  Block high-confidence rules first:
    - IP reputation (very few false positives)
    - Known bad inputs (Log4Shell patterns are never legitimate)
    - Geo-blocking (sanctioned countries)
    - Rate limiting (with generous thresholds initially)
  
  Keep in COUNT:
    - SQL injection (false positives with search queries)
    - XSS (false positives with rich text)
    - Bot control (some legitimate scrapers get caught)

PHASE 3: FULL BLOCK (Week 5+)
  After tuning false positives:
    - Enable BLOCK on remaining rules
    - Add rule_action_override for specific false-positive rules
    - Monitor WAF metrics daily for the first month

THE REASON: A WAF false positive that blocks legitimate
traffic is WORSE than no WAF at all. A blocked checkout =
lost revenue. A blocked API call = broken integration.
WAF is a tool that must be TUNED, not just deployed.
```

### Cloudflare + AWS WAF — Dual-Layer Architecture

```text
NovaMart uses BOTH Cloudflare and AWS WAF. This is common at scale.

WHY TWO WAFs?

┌──────────────────────────────────────────────────────────────────────┐
│ INTERNET                                                             │
│    │                                                                 │
│    ▼                                                                 │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ CLOUDFLARE                                                       │ │
│ │ • CDN (static assets cached at 300+ PoPs)                        │ │
│ │ • L3/L4 DDoS absorption (280+ Tbps capacity)                     │ │
│ │ • L7 WAF (OWASP rules, custom rules)                             │ │
│ │ • Bot Management (ML-based, JS challenge)                        │ │
│ │ • Rate Limiting (edge, before traffic hits AWS)                  │ │
│ │ • Geo-blocking                                                   │ │
│ │                                                                  │ │
│ │ PURPOSE: Absorb volume, block obvious attacks                    │ │
│ │ at the EDGE before traffic reaches AWS                           │ │
│ └────────────────────────┬─────────────────────────────────────────┘ │
│                          │ Only "clean" traffic passes               │
│                          ▼                                           │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ AWS WAF (on ALB)                                                 │ │
│ │ • Application-specific rules                                     │ │
│ │ • Rate limiting per endpoint (/auth, /checkout)                  │ │
│ │ • Request inspection (SQL injection, XSS)                        │ │
│ │ • Integration with Shield Advanced                               │ │
│ │ • Logging to S3/Firehose for analysis                            │ │
│ │                                                                  │ │
│ │ PURPOSE: Application-aware filtering,                            │ │
│ │ defense-in-depth if Cloudflare is bypassed                       │ │
│ └────────────────────────┬─────────────────────────────────────────┘ │
│                          │                                           │
│                          ▼                                           │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ EKS (Application)                                                │ │
│ └──────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘

CRITICAL CONFIGURATION: Restrict ALB to Cloudflare IPs only
  If attackers discover your ALB's direct IP, they bypass Cloudflare.
  
  Security Group for ALB:
    Ingress: Allow HTTPS (443) ONLY from Cloudflare IP ranges
    (Cloudflare publishes their IP ranges: https://www.cloudflare.com/ips/)
  
  ALB access logs: Verify X-Forwarded-For header matches Cloudflare
  Cloudflare: Enable "Authenticated Origin Pulls" (mTLS between
              Cloudflare and your ALB using Cloudflare's CA cert)
```

### WAF Failure Modes

```text
FAILURE 1: WAF false positive blocks legitimate traffic
  CAUSE: Rule too broad (e.g., SQLi rule triggers on product search
         "O'Reilly books" — the apostrophe looks like SQL injection)
  SYMPTOM: Customers get 403 Forbidden, support tickets spike
  IMPACT: Lost revenue, customer frustration
  DEBUG:
    # Check WAF logs for the blocked request
    # WAF logs include: terminatingRuleId, ruleGroupList, httpRequest
    aws wafv2 get-sampled-requests \
      --web-acl-arn <arn> \
      --rule-metric-name "aws-sqli-rules" \
      --scope REGIONAL \
      --time-window StartTime=2024-01-15T00:00:00Z,EndTime=2024-01-15T23:59:59Z \
      --max-items 100
    
    # Look at: terminatingRuleId tells you WHICH rule blocked
    # The httpRequest shows the actual request that was blocked
  FIX:
    - Add rule_action_override for the specific sub-rule to COUNT
    - OR add an IP/URI exclusion for the affected endpoint
    - NEVER disable the entire managed rule group
  PREVENTION:
    - COUNT mode first, always
    - Monitor WAF metrics: AllowedRequests, BlockedRequests
    - Alert on sudden spike in BlockedRequests:
      WAFBlockedRequestSpike > 200% baseline → investigate

FAILURE 2: WAF bypassed — attacker reaches ALB directly
  CAUSE: ALB IP discovered (DNS history, certificate transparency logs,
         Shodan, or educated guessing)
  SYMPTOM: Attack traffic appears in ALB access logs without
           Cloudflare headers (no CF-Connecting-IP header)
  FIX:
    - Security Group: restrict ALB ingress to Cloudflare IPs ONLY
    - Cloudflare Authenticated Origin Pulls (mTLS)
    - Validate CF-Connecting-IP header presence in application
  AUTOMATION:
    # Lambda function to update SG with Cloudflare IPs
    # Cloudflare publishes IPs at https://api.cloudflare.com/client/v4/ips
    # Run daily via EventBridge → Lambda → update Security Group

FAILURE 3: WAF capacity exhaustion (WCU limit)
  CAUSE: Too many rule groups exceed 1500 WCU limit per Web ACL
  SYMPTOM: Terraform apply fails with "WAF capacity exceeded"
  FIX:
    - Remove less critical rules
    - Consolidate custom rules
    - Use scope_down_statement to narrow rule applicability
    - Request WCU limit increase from AWS (possible for Shield Advanced)
  TYPICAL WCU USAGE:
    IP Reputation: 25 WCU
    Common Rules: 700 WCU
    SQLi: 200 WCU
    Known Bad: 200 WCU
    Bot Control: 50 WCU (Common), 750 WCU (Targeted)
    Custom rate limit: 2-5 WCU each
    Total: ~1175-1925 WCU → can hit limits with Bot Control Targeted

FAILURE 4: WAF logging costs explode
  CAUSE: Logging ALL requests (not just blocked/counted)
  SYMPTOM: $10K+ monthly Firehose/S3 charges for WAF logs
  FIX:
    - Use logging_filter to only log BLOCK and COUNT actions
    - Redact sensitive headers (Authorization, Cookie)
    - Set S3 lifecycle policy (Glacier after 90 days)
    - Sample allowed requests (1%) for baseline analysis

FAILURE 5: Shield Advanced DRT can't help during attack
  CAUSE: DRT needs access to your WAF to deploy rules,
         but you haven't granted the DRT IAM role access
  SYMPTOM: DRT contacts you during attack but can't make changes
  FIX: Proactively grant DRT access:
    resource "aws_shield_drt_access_role_arn_association" "main" {
      role_arn = aws_iam_role.shield_drt.arn
    }
    resource "aws_shield_drt_access_log_bucket_association" "main" {
      log_bucket              = aws_s3_bucket.waf_logs.id
      role_arn_association_id = aws_shield_drt_access_role_arn_association.main.id
    }
  PREVENTION: Set up DRT access as part of Shield Advanced provisioning,
              not during an active attack
```

---

## Part 2: Network Segmentation — VPC, Security Groups, NACLs, NetworkPolicies

### NovaMart VPC Architecture — Production Design

```text
NovaMart's VPC design follows the standard three-tier model
with specific security considerations:

┌─────────────────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16 (65,536 IPs) — us-east-1                           │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ PUBLIC SUBNETS (10.0.0.0/20, 10.0.16.0/20, 10.0.32.0/20)        │ │
│ │ 3 AZs × /20 = 4,096 IPs each                                    │ │
│ │                                                                 │ │
│ │ Contains:                                                       │ │
│ │   • ALB/NLB (internet-facing)                                   │ │
│ │   • NAT Gateways (one per AZ)                                   │ │
│ │   • Bastion host (if needed — prefer SSM Session Manager)       │ │
│ │                                                                 │ │
│ │ Route table: 0.0.0.0/0 → Internet Gateway                       │ │
│ │ NACL: Allow 443 inbound from 0.0.0.0/0, deny everything else    │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ PRIVATE SUBNETS (10.0.48.0/18, 10.0.112.0/18, 10.0.176.0/18)    │ │
│ │ 3 AZs × /18 = 16,384 IPs each (for EKS pod IPs — need many)     │ │
│ │                                                                 │ │
│ │ Contains:                                                       │ │
│ │   • EKS worker nodes                                            │ │
│ │   • All application pods (VPC CNI = real VPC IPs per pod)       │ │
│ │   • Internal ALB (service-to-service if needed)                 │ │
│ │                                                                 │ │
│ │ Route table: 0.0.0.0/0 → NAT Gateway (for outbound only)        │ │
│ │ NO Internet Gateway route — no inbound from internet            │ │
│ │ NACL: Allow from VPC CIDR, allow responses on ephemeral ports   │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ DATA SUBNETS (10.0.240.0/22, 10.0.244.0/22, 10.0.248.0/22)      │ │
│ │ 3 AZs × /22 = 1,024 IPs each (databases don't need many)        │ │
│ │                                                                 │ │
│ │ Contains:                                                       │ │
│ │   • RDS PostgreSQL (Multi-AZ)                                   │ │
│ │   • ElastiCache Redis                                           │ │
│ │   • MongoDB Atlas (via PrivateLink)                             │ │
│ │                                                                 │ │
│ │ Route table: NO default route (no internet, no NAT)             │ │
│ │ Only VPC CIDR routes + VPC endpoint routes                      │ │
│ │ NACL: Allow 5432, 6379, 27017 ONLY from private subnets         │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ VPC ENDPOINTS (no internet traversal for AWS services):             │
│   • Gateway: S3, DynamoDB (free)                                    │
│   • Interface: ECR (api + dkr), STS, KMS, Secrets Manager,          │
│                CloudWatch Logs, SSM, EC2                            │
│   • Each interface endpoint: ~$7.50/AZ/month + data transfer        │
│   • ECR endpoints are CRITICAL — without them, image pulls go       │
│     through NAT Gateway ($0.045/GB — extremely expensive at scale)  │
└─────────────────────────────────────────────────────────────────────┘
```

```text
WHY /18 FOR PRIVATE SUBNETS?

  EKS with VPC CNI assigns REAL VPC IPs to every pod.
  NovaMart: 200+ microservices × 3-20 replicas = 1000-4000 pods per AZ
  Plus: system pods (kube-proxy, CoreDNS, monitoring, Istio sidecars)
  
  Each pod = 1 IP. Each node has secondary IPs pre-allocated.
  A /20 (4,096 IPs) sounds like enough but:
    - Node ENIs consume IPs (warm pool for fast scheduling)
    - IP warm pool: each node pre-allocates 10-50 IPs
    - During deployments: old + new pods coexist (double IP usage)
    - Headroom for scaling events (Black Friday = 3-5x)
  
  /18 per AZ (16,384 IPs) provides comfortable headroom.
  /19 is the minimum viable for EKS at NovaMart's scale.
  
  IF YOU RUN OUT OF IPs (and many companies do):
    1. Enable VPC CNI prefix delegation (/28 prefix per ENI slot = 16x more IPs)
    2. Use secondary CIDR block (add 100.64.0.0/16 — RFC 6598 shared space)
    3. Use custom networking (pods get IPs from different subnet than nodes)
```

### Security Groups — NovaMart's Design

```text
SECURITY GROUP PRINCIPLES:
  1. Stateful (return traffic auto-allowed)
  2. Allow-only (no deny rules — use NACLs for deny)
  3. Evaluated ALL rules (not first-match like NACLs)
  4. Can reference OTHER security groups (SG-to-SG)
  5. Applied at ENI level (instance, pod, Lambda, RDS, etc.)

NovaMart Security Group Architecture:
  SG-to-SG references create an implicit trust chain:
  
  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
  │ sg-alb       │────→│ sg-eks-node  │────→│ sg-rds       │
  │              │     │              │     │              │
  │ Inbound:     │     │ Inbound:     │     │ Inbound:     │
  │  443 from    │     │  Any from    │     │  5432 from   │
  │  Cloudflare  │     │  sg-alb      │     │  sg-eks-node │
  │  IPs         │     │  All from    │     │              │
  │              │     │  sg-eks-node │     │ NO public    │
  │              │     │  (pod-to-pod)│     │ access       │
  └──────────────┘     └──────────────┘     └──────────────┘
  
  WHY SG-TO-SG (not CIDR)?
    - SGs are dynamic — instances can be added/removed
    - CIDR rules are static — must update when IPs change
    - SG-to-SG automatically includes any ENI in the referenced SG
    - Cleaner, more maintainable, less error-prone
```

```hcl
# NovaMart Security Groups (Terraform)

# ALB Security Group
resource "aws_security_group" "alb" {
  name_prefix = "novamart-alb-"
  vpc_id      = aws_vpc.main.id
  description = "ALB - Internet-facing"

  # HTTPS from Cloudflare only (not 0.0.0.0/0)
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = var.cloudflare_ip_ranges  # Updated by automation
    description = "HTTPS from Cloudflare"
  }

  # No direct internet access
  # If someone discovers the ALB IP, they can't reach it
  # without going through Cloudflare

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    security_groups = [aws_security_group.eks_node.id]
    description     = "To EKS nodes only"
  }

  tags = { Name = "novamart-alb" }

  lifecycle {
    create_before_destroy = true
  }
}

# EKS Node Security Group
resource "aws_security_group" "eks_node" {
  name_prefix = "novamart-eks-node-"
  vpc_id      = aws_vpc.main.id
  description = "EKS worker nodes"

  # From ALB
  ingress {
    from_port       = 0
    to_port         = 65535
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "From ALB"
  }

  # Node-to-node (pod communication via VPC CNI)
  ingress {
    from_port = 0
    to_port   = 65535
    protocol  = "tcp"
    self      = true
    description = "Node-to-node (pod communication)"
  }

  ingress {
    from_port = 0
    to_port   = 65535
    protocol  = "udp"
    self      = true
    description = "Node-to-node UDP (DNS, VXLAN)"
  }

  # EKS control plane → nodes (kubelet API, webhook)
  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups =[aws_security_group.eks_control_plane.id]
    description     = "EKS control plane to kubelet"
  }

  ingress {
    from_port       = 1025
    to_port         = 65535
    protocol        = "tcp"
    security_groups =[aws_security_group.eks_control_plane.id]
    description     = "EKS control plane to pods (webhooks)"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks =["0.0.0.0/0"]  # Outbound to NAT GW for external deps
    description = "Outbound (via NAT Gateway)"
  }

  tags = { Name = "novamart-eks-node" }
}

# RDS Security Group — MOST RESTRICTIVE
resource "aws_security_group" "rds" {
  name_prefix = "novamart-rds-"
  vpc_id      = aws_vpc.main.id
  description = "RDS PostgreSQL - database tier"

  # ONLY from EKS nodes (pods)
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups =[aws_security_group.eks_node.id]
    description     = "PostgreSQL from EKS pods"
  }

  # NO other ingress. No SSH. No public access.
  # DBA access: use RDS IAM auth + SSM Session Manager port forwarding

  # No broad egress — RDS doesn't need outbound
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [aws_vpc.main.cidr_block]
    description = "VPC internal only"
  }

  tags = { Name = "novamart-rds" }
}

# ElastiCache Security Group
resource "aws_security_group" "elasticache" {
  name_prefix = "novamart-redis-"
  vpc_id      = aws_vpc.main.id
  description = "ElastiCache Redis"

  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [aws_security_group.eks_node.id]
    description     = "Redis from EKS pods"
  }

  tags = { Name = "novamart-redis" }
}

# VPC Endpoint Security Group
resource "aws_security_group" "vpc_endpoints" {
  name_prefix = "novamart-vpce-"
  vpc_id      = aws_vpc.main.id
  description = "VPC Endpoints (ECR, KMS, STS, etc.)"

  ingress {
    from_port       = 443
    to_port         = 443
    protocol        = "tcp"
    security_groups =[aws_security_group.eks_node.id]
    description     = "HTTPS from EKS nodes to VPC endpoints"
  }

  tags = { Name = "novamart-vpc-endpoints" }
}
```

### Kubernetes NetworkPolicies — Pod-Level Segmentation

```text
Security Groups protect at the VPC/ENI level.
NetworkPolicies protect at the POD level WITHIN the cluster.

Without NetworkPolicies:
  Every pod can talk to every other pod. A compromised pod in
  the "search" namespace can reach the payment database.

NovaMart's NetworkPolicy strategy:
  1. DEFAULT DENY ALL — every namespace starts locked down
  2. EXPLICIT ALLOW — only documented communication paths
  3. DNS EGRESS — always allow (or nothing works)
```

```yaml
# STEP 1: Default deny all ingress and egress in every namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: payments  # Applied per namespace
spec:
  podSelector: {}  # Matches ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules = deny everything
  # THIS BREAKS EVERYTHING until you add allow rules
  # That's the point — nothing works unless explicitly permitted

---
# STEP 2: Allow DNS (CRITICAL — without this, no service discovery)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# STEP 3: Allow specific service-to-service communication

# payment-svc can receive from api-gateway and order-svc
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-svc-ingress
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-svc
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: api-gateway
          podSelector:
            matchLabels:
              app: api-gateway
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: orders
          podSelector:
            matchLabels:
              app: order-svc
      ports:
        - protocol: TCP
          port: 8080

---
# payment-svc can reach RDS (data subnet) and Vault
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-svc-egress
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-svc
  policyTypes:
    - Egress
  egress:
    # RDS PostgreSQL
    - to:
        - ipBlock:
            cidr: 10.0.240.0/22  # Data subnet CIDR
      ports:
        - protocol: TCP
          port: 5432
    # Redis
    - to:
        - ipBlock:
            cidr: 10.0.244.0/22
      ports:
        - protocol: TCP
          port: 6379
    # Vault
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: vault
          podSelector:
            matchLabels:
              app.kubernetes.io/name: vault
      ports:
        - protocol: TCP
          port: 8200
    # Stripe API (external payment processor)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0  # Must allow external for Stripe
            except:
              - 10.0.0.0/8       # But not internal ranges
              - 172.16.0.0/12    # (force Stripe through specific egress)
              - 192.168.0.0/16
      ports:
        - protocol: TCP
          port: 443

---
# Allow Prometheus to scrape all pods (monitoring exception)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: payments
spec:
  podSelector: {}  # All pods in namespace
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
      ports:
        - protocol: TCP
          port: 9090  # Metrics port
        - protocol: TCP
          port: 15090  # Istio metrics port
```

```text
NETWORKPOLICY GOTCHAS:

1. CNI MUST SUPPORT NetworkPolicy
   - VPC CNI alone does NOT enforce NetworkPolicies
   - Need: Calico CNI, Cilium, or VPC CNI + Calico policy engine
   - EKS: Install Calico for NetworkPolicy enforcement:
     kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-operator.yaml
     kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/master/calico-crs.yaml

2. NetworkPolicies are ADDITIVE
   - Multiple policies on same pod: UNION of all rules
   - You can't have one policy allow and another deny the same traffic
   - Only the default-deny (empty rules) creates actual deny
   - To block specific traffic: don't include it in any policy

3. DNS EGRESS — most common mistake
   - Default deny egress blocks DNS
   - Without DNS: all service names fail to resolve
   - Symptom: everything times out (not "connection refused")
   - ALWAYS add DNS egress rule to every namespace

4. Istio and NetworkPolicies — BOTH needed
   - NetworkPolicy: L3/L4 (IP + port)
   - Istio AuthorizationPolicy: L7 (HTTP method, path, headers)
   - NetworkPolicy blocks the network connection
   - Istio blocks the HTTP request after connection is established
   - Defense in depth: NetworkPolicy for coarse control,
     Istio for fine-grained L7 control

5. NAMESPACE LABELS are critical
   - namespaceSelector matches on namespace LABELS
   - Kubernetes adds kubernetes.io/metadata.name automatically (1.22+)
   - If you're on older K8s: manually label namespaces
   - Missing label = policy doesn't match = traffic denied/allowed unexpectedly
```

### Istio Security Policies — L7 Enforcement

```yaml
# Istio mTLS — encrypt all pod-to-pod traffic
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system  # Mesh-wide
spec:
  mtls:
    mode: STRICT
    # STRICT: Only mTLS connections accepted
    # PERMISSIVE: Accept both plain and mTLS (migration mode)
    # DISABLE: No mTLS
    
    # Start with PERMISSIVE during Istio rollout
    # Switch to STRICT once all services have sidecars
    # STRICT prevents non-mesh services from communicating
    # (which is a security feature AND a migration headache)

---
# Istio AuthorizationPolicy — L7 access control for payment-svc
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-svc-authz
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payment-svc
  action: ALLOW
  rules:
    # API Gateway can call payment endpoints
    - from:
        - source:
            principals: ["cluster.local/ns/api-gateway/sa/api-gateway"]
      to:
        - operation:
            methods: ["POST"]
            paths:["/api/v1/payments/*"]
    # Order service can check payment status
    - from:
        - source:
            principals: ["cluster.local/ns/orders/sa/order-svc"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/api/v1/payments/*/status"]
    # Prometheus can scrape metrics
    - from:
        - source:
            principals:["cluster.local/ns/monitoring/sa/prometheus"]
      to:
        - operation:
            methods: ["GET"]
            paths: ["/metrics", "/actuator/prometheus"]

---
# Default deny — all traffic to payments namespace denied unless explicitly allowed
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: payments
spec:
  {}
  # Empty spec = DENY ALL
  # This is the Istio equivalent of the NetworkPolicy default-deny
  # Combined with specific ALLOW policies above = allowlist model
```

```text
NETWORKPOLICY vs ISTIO AUTHORIZATIONPOLICY:

┌───────────────────┬──────────────────┬──────────────────────┐
│ Feature           │ NetworkPolicy    │ Istio AuthzPolicy    │
├───────────────────┼──────────────────┼──────────────────────┤
│ Layer             │ L3/L4 (IP/port)  │ L7 (HTTP/gRPC)       │
│ Identity          │ Pod labels, CIDR │ Service identity     │
│                   │                  │ (SPIFFE, mTLS cert)  │
│ Granularity       │ IP + port        │ Method + path +      │
│                   │                  │ headers + JWT claims │
│ Encryption        │ No               │ Yes (mTLS)           │
│ Non-mesh services │ ✅ Works         │ ❌ Need sidecar      │
│ Performance       │ Kernel (fast)    │ Userspace (Envoy)    │
│ Enforcement point │ Linux kernel/eBPF│ Envoy proxy          │
│ Failure mode      │ Packets dropped  │ HTTP 403 returned    │
└───────────────────┴──────────────────┴──────────────────────┘

USE BOTH:
  NetworkPolicy = coarse network segmentation (namespaces, CIDRs)
  Istio = fine-grained application access control (methods, paths)
  
  An attacker who compromises a sidecar can bypass Istio policies
  but still hits NetworkPolicy. An attacker who spoofs pod labels
  might bypass NetworkPolicy but can't forge mTLS certificates.
```

---

## Part 3: AWS Detection & Response Services

### GuardDuty — Threat Detection

```text
GuardDuty is AWS's managed threat detection service.
It analyzes multiple data sources and produces FINDINGS
(security alerts) without requiring any infrastructure from you.

DATA SOURCES GuardDuty analyzes:
┌──────────────────────┬─────────────────────────────────────────┐
│ Source               │ What It Detects                         │
├──────────────────────┼─────────────────────────────────────────┤
│ CloudTrail Events    │ Unusual API calls, credential abuse,    │
│                      │ unauthorized regions, privilege esc.    │
│ CloudTrail Mgmt      │ Console logins, IAM changes,            │
│                      │ resource policy modifications           │
│ VPC Flow Logs        │ Port scanning, C2 communication,        │
│                      │ cryptocurrency mining traffic,          │
│                      │ data exfiltration patterns              │
│ DNS Logs             │ DNS queries to malicious domains,       │
│                      │ DGA (domain generation algorithm),      │
│                      │ DNS tunneling                           │
│ EKS Audit Logs       │ Suspicious K8s API calls, anonymous     │
│                      │ auth, privileged containers, exec into  │
│                      │ pods, unusual RBAC changes              │
│ S3 Data Events       │ Unusual S3 access patterns, public      │
│                      │ bucket access, data exfiltration        │
│ RDS Login Activity   │ Brute force login attempts, unusual     │
│                      │ login locations, successful login from  │
│                      │ known-bad IPs                           │
│ Lambda Network       │ Lambda functions contacting malicious   │
│                      │ IPs, unusual network behavior           │
│ EBS Malware          │ Scans EBS volumes for malware when      │
│ Protection           │ GuardDuty detects suspicious behavior   │
└──────────────────────┴─────────────────────────────────────────┘

GUARDDUTY DOES NOT:
  ❌ Prevent attacks (detection only, not prevention)
  ❌ Require you to manage infrastructure (fully managed)
  ❌ Inspect packet contents (analyzes metadata/flow, not payload)
  ❌ Replace WAF, Shield, or SecurityGroups (different layers)

GUARDDUTY FINDING TYPES (examples):
  Recon:EC2/PortProbeUnprotectedPort — port scan detected
  UnauthorizedAccess:EC2/MaliciousIPCaller — EC2 contacted C2 server
  CryptoCurrency:EC2/BitcoinTool.B!DNS — crypto mining DNS query
  Trojan:EC2/DropPoint — EC2 is a data exfiltration drop point
  Policy:Kubernetes/ExposedDashboard — K8s dashboard exposed
  CredentialAccess:Kubernetes/SuccessfulAnonymousAccess — anonymous K8s API
  Execution:Kubernetes/ExecInKubePod — exec into pod (could be legitimate)
  Impact:Kubernetes/SuccessfulAnonymousAccess — unauthorized K8s access
  Persistence:IAMUser/AnomalousBehavior — unusual IAM activity pattern
```

```hcl
# GuardDuty — enable all protection plans

resource "aws_guardduty_detector" "main" {
  enable = true

  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }

  finding_publishing_frequency = "FIFTEEN_MINUTES"
  # Options: FIFTEEN_MINUTES, ONE_HOUR, SIX_HOURS
  # FIFTEEN_MINUTES for production — fastest notification

  tags = {
    Environment = "production"
  }
}

# Enable EKS Runtime Monitoring (agent-based, deeper than audit logs)
resource "aws_guardduty_detector_feature" "eks_runtime" {
  detector_id = aws_guardduty_detector.main.id
  name        = "EKS_RUNTIME_MONITORING"
  status      = "ENABLED"

  additional_configuration {
    name   = "EKS_ADDON_MANAGEMENT"
    status = "ENABLED"
    # AWS manages the GuardDuty agent DaemonSet in EKS
    # Similar to Falco but AWS-managed
  }
}

# Enable RDS Login Activity monitoring
resource "aws_guardduty_detector_feature" "rds" {
  detector_id = aws_guardduty_detector.main.id
  name        = "RDS_LOGIN_EVENTS"
  status      = "ENABLED"
}

# Enable Lambda Network Activity monitoring
resource "aws_guardduty_detector_feature" "lambda" {
  detector_id = aws_guardduty_detector.main.id
  name        = "LAMBDA_NETWORK_LOGS"
  status      = "ENABLED"
}
```

### GuardDuty → Automated Response

```text
GuardDuty DETECTS. You need to BUILD the response pipeline.

┌──────────────┐     ┌──────────────┐     ┌────────────────┐
│ GuardDuty    │────→│ EventBridge  │────→│ Step Functions │
│ Finding      │     │ Rule         │     │ Workflow       │
└──────────────┘     └──────────────┘     └────────┬───────┘
                                                   │
                                         ┌─────────┼─────────┐
                                         │         │         │
                                         ▼         ▼         ▼
                                    ┌────────┐ ┌──────┐ ┌──────────┐
                                    │ Lambda │ │Slack │ │PagerDuty │
                                    │(isolate│ │alert │ │page      │
                                    │ EC2,   │ │      │ │on-call   │
                                    │ revoke │ │      │ │          │
                                    │ creds) │ │      │ │          │
                                    └────────┘ └──────┘ └──────────┘
```

```hcl
# EventBridge rule for GuardDuty findings
resource "aws_cloudwatch_event_rule" "guardduty_high" {
  name        = "guardduty-high-severity"
  description = "GuardDuty HIGH and CRITICAL findings"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [
        { numeric =[">=", 7] }
        # GuardDuty severity: 0-3.9 LOW, 4-6.9 MEDIUM, 7-8.9 HIGH, 9-10 CRITICAL
      ]
    }
  })
}

resource "aws_cloudwatch_event_target" "guardduty_sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_high.name
  target_id = "guardduty-to-sns"
  arn       = aws_sns_topic.security_alerts.arn
}

resource "aws_cloudwatch_event_target" "guardduty_lambda" {
  rule      = aws_cloudwatch_event_rule.guardduty_high.name
  target_id = "guardduty-auto-response"
  arn       = aws_lambda_function.guardduty_response.arn
}

# EventBridge rule for specific critical finding types
resource "aws_cloudwatch_event_rule" "guardduty_crypto" {
  name        = "guardduty-cryptocurrency"
  description = "Cryptocurrency mining detected"

  event_pattern = jsonencode({
    source      =["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      type =[
        "CryptoCurrency:EC2/BitcoinTool.B!DNS",
        "CryptoCurrency:EC2/BitcoinTool.B",
        "CryptoCurrency:Runtime/BitcoinTool.B",
        "CryptoCurrency:Runtime/BitcoinTool.B!DNS"
      ]
    }
  })
}

resource "aws_cloudwatch_event_target" "crypto_isolate" {
  rule      = aws_cloudwatch_event_rule.guardduty_crypto.name
  target_id = "auto-isolate-instance"
  arn       = aws_lambda_function.isolate_instance.arn
}
```

```python
# Lambda: Auto-isolate compromised EC2 instance
# Triggered by GuardDuty finding via EventBridge

import boto3
import json
import os

ec2 = boto3.client('ec2')
sns = boto3.client('sns')

# Pre-created "quarantine" security group — denies all traffic
QUARANTINE_SG = os.environ['QUARANTINE_SG_ID']
SNS_TOPIC = os.environ['SNS_TOPIC_ARN']

def handler(event, context):
    finding = event['detail']
    finding_type = finding['type']
    severity = finding['severity']
    
    # Extract instance ID from finding
    instance_id = None
    if 'resource' in finding:
        resource = finding['resource']
        if 'instanceDetails' in resource:
            instance_id = resource['instanceDetails']['instanceId']
    
    if not instance_id:
        print(f"No instance ID in finding: {finding_type}")
        return
    
    print(f"ALERT: {finding_type} (severity: {severity}) on {instance_id}")
    
    # Step 1: Snapshot the instance's current security groups (forensics)
    instance = ec2.describe_instances(InstanceIds=[instance_id])
    current_sgs = [
        sg['GroupId'] 
        for sg in instance['Reservations'][0]['Instances'][0]['SecurityGroups']
    ]
    
    # Step 2: Replace all security groups with quarantine SG
    # This immediately cuts ALL network access (except what quarantine SG allows)
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[QUARANTINE_SG]
    )
    
    # Step 3: Tag the instance for tracking
    ec2.create_tags(
        Resources=[instance_id],
        Tags=[
            {'Key': 'SecurityIncident', 'Value': 'true'},
            {'Key': 'IncidentType', 'Value': finding_type},
            {'Key': 'OriginalSecurityGroups', 'Value': json.dumps(current_sgs)},
            {'Key': 'IsolatedAt', 'Value': context.function_name},
            {'Key': 'QuarantinedBy', 'Value': 'guardduty-auto-response'}
        ]
    )
    
    # Step 4: Create EBS snapshot for forensic analysis
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]}]
    )
    for vol in volumes['Volumes']:
        ec2.create_snapshot(
            VolumeId=vol['VolumeId'],
            Description=f"Forensic snapshot - {finding_type} - {instance_id}",
            TagSpecifications=[{
                'ResourceType': 'snapshot',
                'Tags':[
                    {'Key': 'SecurityIncident', 'Value': 'true'},
                    {'Key': 'SourceInstance', 'Value': instance_id}
                ]
            }]
        )
    
    # Step 5: Notify security team
    sns.publish(
        TopicArn=SNS_TOPIC,
        Subject=f"🚨 GuardDuty: {finding_type} - Instance Isolated",
        Message=json.dumps({
            'finding_type': finding_type,
            'severity': severity,
            'instance_id': instance_id,
            'action_taken': 'Instance isolated with quarantine SG',
            'original_security_groups': current_sgs,
            'forensic_snapshots': 'Created for all attached volumes',
            'next_steps':[
                'Review GuardDuty finding details',
                'Analyze forensic snapshots',
                'Check CloudTrail for lateral movement',
                'Determine blast radius',
                'Begin incident response procedure'
            ]
        }, indent=2)
    )
    
    # DO NOT terminate the instance — preserve for forensics
    # DO NOT reboot — may trigger cleanup scripts
    
    return {
        'statusCode': 200,
        'body': f"Instance {instance_id} quarantined for {finding_type}"
    }
```

```hcl
# Quarantine Security Group — used by auto-response Lambda
resource "aws_security_group" "quarantine" {
  name        = "novamart-quarantine"
  vpc_id      = aws_vpc.main.id
  description = "Quarantine SG - denies all traffic. Applied to compromised instances."

  # NO ingress rules = deny all inbound
  # NO egress rules = deny all outbound

  # Exception: Allow SSM Session Manager for forensic investigation
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks =[aws_vpc.main.cidr_block]
    description = "SSM endpoint for forensic access"
  }

  tags = {
    Name    = "novamart-quarantine"
    Purpose = "Incident response - auto-applied by GuardDuty automation"
  }
}
```

### GuardDuty Failure Modes

```text
FAILURE 1: GuardDuty disabled or not enabled in all regions
  CAUSE: Terraform only enables GuardDuty in primary region,
         attacker operates in unused region (ap-southeast-1)
  SYMPTOM: No findings for activity in secondary regions
  FIX: Enable GuardDuty in ALL regions (even unused ones)
    # Use aws_guardduty_detector in every region
    # Or use AWS Organizations delegated admin for auto-enable
  THIS IS A COMMON AUDIT FINDING.

FAILURE 2: GuardDuty finding suppression too broad
  CAUSE: Engineer suppressed "Recon:EC2/PortProbeUnprotectedPort"
         globally to stop noise from internal scanning tools
  SYMPTOM: Real port scanning from attackers also suppressed
  FIX: Use targeted suppression filters:
    resource "aws_guardduty_filter" "suppress_internal_scan" {
      detector_id = aws_guardduty_detector.main.id
      name        = "suppress-internal-scanner"
      action      = "ARCHIVE"
      
      finding_criteria {
        criterion {
          field  = "type"
          equals = ["Recon:EC2/PortProbeUnprotectedPort"]
        }
        criterion {
          field  = "service.action.networkConnectionAction.remoteIpDetails.ipAddressV4"
          equals =["10.0.50.100"]  # Only suppress from the scanner IP
        }
      }
    }
  RULE: NEVER suppress by finding type alone. Always add source/target filter.

FAILURE 3: GuardDuty EKS findings too noisy
  CAUSE: Legitimate kubectl exec operations trigger
         "Execution:Kubernetes/ExecInKubePod" findings
  SYMPTOM: Security team ignores K8s findings due to noise
  FIX: Suppress findings from specific service accounts (CI/CD) and
       legitimate admin users, while alerting on unexpected sources
  ALSO: Configure trusted IP lists:
    resource "aws_guardduty_ipset" "trusted" {
      detector_id = aws_guardduty_detector.main.id
      format      = "TXT"
      location    = "s3://${aws_s3_bucket.guardduty.id}/trusted-ips.txt"
      name        = "NovaMartTrustedIPs"
      activate    = true
    }

FAILURE 4: Auto-response Lambda isolates wrong instance
  CAUSE: GuardDuty finding references an instance that's already terminated
         (ASG replaced it), Lambda tries to modify non-existent instance
  SYMPTOM: Lambda error, no isolation, security gap
  FIX: Add existence check before modification:
    try:
        ec2.describe_instances(InstanceIds=[instance_id])
    except ec2.exceptions.InvalidInstanceID.NotFound:
        # Instance already gone — check if ASG launched replacement
        # Verify replacement isn't also compromised
  ALSO: For EKS, the response should target the POD (kubectl delete)
        not the EC2 instance (which hosts many pods)

FAILURE 5: GuardDuty findings not reaching the team
  CAUSE: EventBridge rule doesn't match the finding format,
         SNS topic has no subscriptions, Lambda has no permissions
  SYMPTOM: Findings accumulate in GuardDuty console, nobody knows
  FIX: 
    - Test the pipeline: generate a test finding:
      aws guardduty create-sample-findings \
        --detector-id <id> \
        --finding-types "Recon:EC2/PortProbeUnprotectedPort"
    - Verify: EventBridge → Lambda → Slack message arrives
    - Monitor: alert on Lambda errors for the response function
  PREVENTION: Include GuardDuty pipeline testing in quarterly DR exercises
```

### Security Hub — Centralized Security Posture

```text
Security Hub AGGREGATES findings from multiple sources into
a single pane of glass:

┌─────────────────────────────────────────────────────────────────┐
│                        AWS SECURITY HUB                         │
│                                                                 │
│ FINDING SOURCES:                                                │
│ ┌─────────────┐ ┌──────────┐ ┌──────────┐ ┌───────────────────┐ │
│ │ GuardDuty   │ │Inspector │ │ Firewall │ │ IAM Access        │ │
│ │ (threats)   │ │(vulns)   │ │ Manager  │ │ Analyzer          │ │
│ └──────┬──────┘ └────┬─────┘ └────┬─────┘ └─────────┬─────────┘ │
│        │             │            │                 │           │
│        └──────┬──────┴─────┬──────┘─────────────────┘           │
│               │            │                                    │
│ ┌─────────────▼────────────▼──────────────────────────────────┐ │
│ │                      SECURITY HUB                           │ │
│ │                                                             │ │
│ │ ASFF (AWS Security Finding Format)                          │ │
│ │ Normalizes all findings into a common format                │ │
│ │                                                             │ │
│ │ COMPLIANCE STANDARDS:                                       │ │
│ │ ├── AWS Foundational Security Best Practices                │ │
│ │ ├── CIS AWS Foundations Benchmark                           │ │
│ │ ├── PCI DSS v3.2.1                                          │ │
│ │ ├── NIST 800-53                                             │ │
│ │ └── SOC 2                                                   │ │
│ │                                                             │ │
│ │ Each standard = set of automated checks (controls)          │ │
│ │ Security Hub runs these checks continuously                 │ │
│ │ Results: PASSED, FAILED, NOT_AVAILABLE                      │ │
│ │ Overall compliance score per standard                       │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                 │
│ OUTPUTS:                                                        │
│ ├── Dashboard (compliance scores, finding trends)               │
│ ├── EventBridge (automated response)                            │
│ ├── Custom actions (manual workflows)                           │
│ └── Cross-account aggregation (org-wide view)                   │
└─────────────────────────────────────────────────────────────────┘

SECURITY HUB IS NOT:
  ❌ A SIEM (no log aggregation, no correlation, no custom queries)
  ❌ A replacement for GuardDuty (Hub aggregates, GuardDuty detects)
  ❌ Free ($0.0010 per check, adds up at scale)

SECURITY HUB IS:
  ✅ Compliance dashboard (are we meeting PCI/CIS/NIST?)
  ✅ Finding aggregator (one place for all security findings)
  ✅ Automated remediation trigger (via EventBridge)
  ✅ Cross-account/cross-region view (via Organizations)
```

```hcl
# Security Hub configuration

resource "aws_securityhub_account" "main" {}

# Enable compliance standards
resource "aws_securityhub_standards_subscription" "aws_best_practices" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/aws-foundational-security-best-practices/v/1.0.0"
  depends_on    = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
  depends_on    = [aws_securityhub_account.main]
}

resource "aws_securityhub_standards_subscription" "pci" {
  standards_arn = "arn:aws:securityhub:us-east-1::standards/pci-dss/v/3.2.1"
  depends_on    = [aws_securityhub_account.main]
}

# Aggregate findings from all regions and member accounts
resource "aws_securityhub_finding_aggregator" "main" {
  linking_mode = "ALL_REGIONS"
  depends_on   =[aws_securityhub_account.main]
}

# Enable product integrations
resource "aws_securityhub_product_subscription" "guardduty" {
  product_arn = "arn:aws:securityhub:us-east-1::product/aws/guardduty"
  depends_on  =[aws_securityhub_account.main]
}

resource "aws_securityhub_product_subscription" "inspector" {
  product_arn = "arn:aws:securityhub:us-east-1::product/aws/inspector"
  depends_on  = [aws_securityhub_account.main]
}

# Auto-remediation for specific findings
resource "aws_cloudwatch_event_rule" "securityhub_s3_public" {
  name = "securityhub-s3-public-bucket"

  event_pattern = jsonencode({
    source      = ["aws.securityhub"]
    detail-type = ["Security Hub Findings - Imported"]
    detail = {
      findings = {
        Compliance = { Status = ["FAILED"] }
        GeneratorId =["aws-foundational-security-best-practices/v/1.0.0/S3.2"]
        # S3.2: S3 buckets should prohibit public read access
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "fix_s3_public" {
  rule      = aws_cloudwatch_event_rule.securityhub_s3_public.name
  target_id = "auto-fix-public-s3"
  arn       = aws_lambda_function.fix_s3_public_access.arn
}
```

### CloudTrail — API Audit Trail

```text
CloudTrail records EVERY API call made in your AWS account.
Every. Single. One. This is your forensic goldmine.

WHO did WHAT to WHICH resource, WHEN, and from WHERE.

┌───────────────────────────────────────────────────────────┐
│ CloudTrail Event:                                         │
│                                                           │
│ eventTime:      2024-01-15T14:23:45Z                      │
│ eventSource:    iam.amazonaws.com                         │
│ eventName:      CreateAccessKey                           │
│ userIdentity:                                             │
│   type:         AssumedRole                               │
│   arn:          arn:aws:sts::888:assumed-role/dev/user    │
│   principalId:  AROA...:user@novamart.com                 │
│ sourceIPAddress: 203.0.113.42                             │
│ userAgent:      aws-cli/2.15.0                            │
│ requestParameters:                                        │
│   userName:     payment-svc-deployer                      │
│ responseElements:                                         │
│   accessKey:                                              │
│     accessKeyId: AKIA...                                  │
│     status:     Active                                    │
└───────────────────────────────────────────────────────────┘

This tells you:
- A developer assumed the dev role
- From IP 203.0.113.42 using AWS CLI
- Created an access key for payment-svc-deployer
- The new key ID is AKIA...
- This happened at 14:23:45 UTC on Jan 15

If this was unauthorized → you know exactly who, what, when, where
```

```hcl
# CloudTrail — production configuration

resource "aws_cloudtrail" "main" {
  name                          = "novamart-production"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  s3_key_prefix                 = "cloudtrail"
  include_global_service_events = true
  is_multi_region_trail         = true  # ALL regions, not just primary
  enable_log_file_validation    = true  # Tamper-proof digest files

  # Send to CloudWatch for real-time alerting
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cloudwatch.arn

  # Data events — API calls to specific resources
  event_selector {
    read_write_type           = "All"
    include_management_events = true

    # S3 data events (object-level operations)
    data_resource {
      type   = "AWS::S3::Object"
      values =[
        "${aws_s3_bucket.payment_data.arn}/",
        "${aws_s3_bucket.customer_data.arn}/",
        # Only log data events for sensitive buckets
        # Logging ALL S3 operations is extremely expensive
      ]
    }

    # Lambda invocation events
    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda"]  # All Lambda functions
    }
  }

  # Advanced event selectors for EKS API
  advanced_event_selector {
    name = "EKS-API-calls"
    field_selector {
      field  = "eventCategory"
      equals = ["Management"]
    }
    field_selector {
      field  = "eventSource"
      equals = ["eks.amazonaws.com"]
    }
  }

  # KMS encryption for CloudTrail logs
  kms_key_id = aws_kms_key.cloudtrail.arn

  tags = {
    Environment = "production"
    Compliance  = "pci-soc2"
  }
}

# CloudTrail S3 bucket — tamper-proof
resource "aws_s3_bucket" "cloudtrail" {
  bucket = "novamart-cloudtrail-${data.aws_caller_identity.current.account_id}"

  # Prevent accidental deletion
  force_destroy = false
}

resource "aws_s3_bucket_versioning" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id
  versioning_configuration {
    status = "Enabled"  # Prevent overwrite of logs
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    id     = "archive-and-expire"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "STANDARD_IA"
    }
    transition {
      days          = 365
      storage_class = "GLACIER"
    }
    # PCI requires 1 year retention. SOC2 may require more.
    # NovaMart: 2 years before deletion
    expiration {
      days = 730
    }
  }
}

# Object Lock for compliance (write-once-read-many)
resource "aws_s3_bucket_object_lock_configuration" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  rule {
    default_retention {
      mode = "COMPLIANCE"  # Even root can't delete during retention
      days = 365
    }
  }
}
```

### CloudTrail → Athena — Forensic Queries

```sql
-- Create Athena table over CloudTrail logs
-- (AWS provides this automatically via CloudTrail Lake, but Athena is cheaper)

CREATE EXTERNAL TABLE cloudtrail_logs (
  eventVersion STRING,
  userIdentity STRUCT<
    type: STRING,
    principalId: STRING,
    arn: STRING,
    accountId: STRING,
    invokedBy: STRING,
    accessKeyId: STRING,
    userName: STRING,
    sessionContext: STRUCT<
      attributes: STRUCT<mfaAuthenticated: STRING, creationDate: STRING>,
      sessionIssuer: STRUCT<type: STRING, principalId: STRING, arn: STRING, accountId: STRING, userName: STRING>
    >
  >,
  eventTime STRING,
  eventSource STRING,
  eventName STRING,
  awsRegion STRING,
  sourceIPAddress STRING,
  userAgent STRING,
  requestParameters STRING,
  responseElements STRING,
  errorCode STRING,
  errorMessage STRING,
  resources ARRAY<STRUCT<arn: STRING, accountId: STRING, type: STRING>>
)
ROW FORMAT SERDE 'org.apache.hive.hcljson.serde.JsonSerDe'
LOCATION 's3://novamart-cloudtrail-888888888888/cloudtrail/AWSLogs/888888888888/CloudTrail/';

-- FORENSIC QUERY 1: Who created IAM credentials in the last 7 days?
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName,
  requestParameters,
  sourceIPAddress,
  userAgent
FROM cloudtrail_logs
WHERE eventName IN ('CreateAccessKey', 'CreateLoginProfile', 'CreateUser')
  AND from_iso8601_timestamp(eventTime) > current_timestamp - interval '7' day
ORDER BY eventTime DESC;

-- FORENSIC QUERY 2: Console logins without MFA
SELECT
  eventTime,
  userIdentity.arn AS who,
  sourceIPAddress,
  userAgent,
  responseElements
FROM cloudtrail_logs
WHERE eventName = 'ConsoleLogin'
  AND userIdentity.sessionContext.attributes.mfaAuthenticated = 'false'
  AND from_iso8601_timestamp(eventTime) > current_timestamp - interval '30' day
ORDER BY eventTime DESC;

-- FORENSIC QUERY 3: Unauthorized API calls (AccessDenied)
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName,
  eventSource,
  errorCode,
  errorMessage,
  sourceIPAddress
FROM cloudtrail_logs
WHERE errorCode IN ('AccessDenied', 'UnauthorizedAccess', 'Client.UnauthorizedAccess')
  AND from_iso8601_timestamp(eventTime) > current_timestamp - interval '24' hour
ORDER BY eventTime DESC;

-- FORENSIC QUERY 4: Security group modifications
SELECT
  eventTime,
  userIdentity.arn AS who,
  eventName,
  requestParameters,
  sourceIPAddress
FROM cloudtrail_logs
WHERE eventName IN (
  'AuthorizeSecurityGroupIngress',
  'AuthorizeSecurityGroupEgress',
  'RevokeSecurityGroupIngress',
  'RevokeSecurityGroupEgress',
  'CreateSecurityGroup',
  'DeleteSecurityGroup'
)
  AND from_iso8601_timestamp(eventTime) > current_timestamp - interval '7' day
ORDER BY eventTime DESC;

-- FORENSIC QUERY 5: All actions by a specific compromised credential
SELECT
  eventTime,
  eventName,
  eventSource,
  sourceIPAddress,
  userAgent,
  requestParameters,
  errorCode
FROM cloudtrail_logs
WHERE userIdentity.accessKeyId = 'AKIA_COMPROMISED_KEY_ID'
ORDER BY eventTime ASC;  -- Chronological to trace the attack path
```

### VPC Flow Logs — Network Forensics

```text
VPC Flow Logs capture network traffic metadata (NOT payload):
  Source IP, Destination IP, Source Port, Destination Port,
  Protocol, Packets, Bytes, Start/End time, Action (ACCEPT/REJECT)

Flow Logs can be attached to:
  - VPC (all ENIs in the VPC)
  - Subnet (all ENIs in the subnet)
  - ENI (specific network interface)

NovaMart: VPC-level flow logs → S3 → Athena for queries
```

```hcl
# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"  # ACCEPT, REJECT, or ALL
  log_destination_type = "s3"
  log_destination      = aws_s3_bucket.flow_logs.arn
  
  max_aggregation_interval = 60  # 60 seconds (vs default 600)
  # 60s gives better granularity for incident investigation
  # More expensive (more log entries) but worth it for production
  
  log_format = "$${version} $${account-id} $${interface-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${start} $${end} $${action} $${log-status} $${vpc-id} $${subnet-id} $${az-id} $${sublocation-type} $${sublocation-id} $${pkt-srcaddr} $${pkt-dstaddr} $${region} $${pkt-src-aws-service} $${pkt-dst-aws-service} $${flow-direction} $${traffic-path}"
  # Custom format includes:
  # - pkt-srcaddr/pkt-dstaddr: Original IPs (before NAT)
  # - pkt-src-aws-service: Whether traffic is to/from AWS service
  # - flow-direction: ingress or egress
  # - traffic-path: Through IGW, NAT GW, VPC Peering, etc.

  tags = {
    Environment = "production"
  }
}
```

```sql
-- Athena queries over VPC Flow Logs

-- QUERY 1: Top talkers by bytes (data exfiltration detection)
SELECT
  srcaddr,
  dstaddr,
  SUM(bytes) as total_bytes,
  COUNT(*) as flow_count
FROM vpc_flow_logs
WHERE start >= to_unixtime(current_timestamp - interval '1' hour)
  AND action = 'ACCEPT'
  AND dstaddr NOT LIKE '10.%'  -- External destinations only
GROUP BY srcaddr, dstaddr
ORDER BY total_bytes DESC
LIMIT 20;

-- QUERY 2: Rejected connections (port scanning, misconfig)
SELECT
  srcaddr,
  dstaddr,
  dstport,
  protocol,
  COUNT(*) as reject_count
FROM vpc_flow_logs
WHERE action = 'REJECT'
  AND start >= to_unixtime(current_timestamp - interval '1' hour)
GROUP BY srcaddr, dstaddr, dstport, protocol
ORDER BY reject_count DESC
LIMIT 50;

-- QUERY 3: Traffic to known bad IPs (feed from threat intel)
SELECT
  srcaddr,
  dstaddr,
  dstport,
  bytes,
  start,
  "end",
  "interface-id"
FROM vpc_flow_logs
WHERE dstaddr IN ('45.33.32.156', '185.143.223.0/24')  -- Known C2 IPs
  AND action = 'ACCEPT'
ORDER BY start DESC;

-- QUERY 4: Unusual outbound ports from EKS subnets
SELECT
  srcaddr,
  dstaddr,
  dstport,
  SUM(bytes) as total_bytes,
  COUNT(*) as connection_count
FROM vpc_flow_logs
WHERE srcaddr LIKE '10.0.48.%' OR srcaddr LIKE '10.0.112.%' OR srcaddr LIKE '10.0.176.%'
  -- EKS private subnets
  AND dstport NOT IN (443, 80, 53, 5432, 6379, 8200)
  -- Unexpected ports (not HTTPS, HTTP, DNS, Postgres, Redis, Vault)
  AND action = 'ACCEPT'
  AND dstaddr NOT LIKE '10.%'  -- External only
  AND start >= to_unixtime(current_timestamp - interval '24' hour)
GROUP BY srcaddr, dstaddr, dstport
ORDER BY total_bytes DESC
LIMIT 50;
```

### AWS Detective — Investigation Graphs

```text
Detective builds RELATIONSHIP GRAPHS from:
  - CloudTrail events
  - VPC Flow Logs
  - GuardDuty findings
  - EKS audit logs

When GuardDuty fires a finding, Detective helps you answer:
  "What else did this entity do?"

┌──────────────────────────────────────────────────────────┐
│ GuardDuty Finding: UnauthorizedAccess from IP 1.2.3.4    │
│                                                          │
│ Detective investigation graph:                           │
│                                                          │
│ IP: 1.2.3.4                                              │
│   ├── Assumed Role: dev-role (14:23 UTC)                 │
│   │   ├── Created AccessKey for deploy-user              │
│   │   ├── Listed S3 buckets                              │
│   │   ├── Downloaded s3://customer-data/export.csv       │
│   │   └── Modified Security Group sg-abc123              │
│   │                                                      │
│   ├── Also seen from same IP:                            │
│   │   ├── Failed ConsoleLogin (3 attempts)               │
│   │   └── Successful ConsoleLogin (after brute force)    │
│   │                                                      │
│   └── Network activity:                                  │
│       ├── Connected to EC2 instance i-xyz (SSH port 22)  │
│       └── Exfiltrated 2.3GB to external IP 5.6.7.8       │
└──────────────────────────────────────────────────────────┘

Detective aggregates 12+ months of data to build these graphs.
You don't query it — you NAVIGATE the graph from a finding.

Cost: Based on data volume ingested. Can be significant at scale.
NovaMart: Enabled for production accounts. Security team uses it
during incident investigations.
```

```hcl
# Enable Detective
resource "aws_detective_graph" "main" {
  tags = {
    Environment = "production"
  }
}

# Auto-enable for new member accounts (Organizations)
resource "aws_detective_organization_admin_account" "main" {
  account_id = data.aws_caller_identity.current.account_id
}
```

### NovaMart Complete Detection & Response Architecture

```text
┌─────────────────────────────────────────────────────────────────────┐
│          NOVAMART DETECTION & RESPONSE — COMPLETE PIPELINE          │
│                                                                     │
│ DATA SOURCES:                                                       │
│ ┌─────────────┐ ┌───────────┐ ┌──────────┐ ┌───────────────────┐    │
│ │ CloudTrail  │ │ VPC Flow  │ │ DNS Logs │ │ EKS Audit Logs    │    │
│ │ (API calls) │ │ Logs      │ │          │ │ (K8s API calls)   │    │
│ └──────┬──────┘ └─────┬─────┘ └────┬─────┘ └────────┬──────────┘    │
│        │              │            │                │               │
│        └──────────────┼────────────┼────────────────┘               │
│                       │            │                                │
│                       ▼            ▼                                │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ DETECTION:                                                      │ │
│ │                                                                 │ │
│ │ GuardDuty (threats) ──┐                                         │ │
│ │ Inspector (vulns) ────┤                                         │ │
│ │ IAM Access Analyzer ──┼──→ Security Hub (aggregate)             │ │
│ │ Config Rules (config)─┤    ├── Compliance scoring               │ │
│ │ Firewall Manager ─────┘    ├── Finding prioritization           │ │
│ │                            └── Cross-account/region view        │ │
│ └───────────────────────────────────┬─────────────────────────────┘ │
│                                     │                               │
│ ┌───────────────────────────────────▼─────────────────────────────┐ │
│ │ RESPONSE:                                                       │ │
│ │                                                                 │ │
│ │ EventBridge ──→ Step Functions ──→ Lambda (auto-remediate)      │ │
│ │             ──→ SNS ──→ PagerDuty (page on-call)                │ │
│ │             ──→ SNS ──→ Slack #security-alerts                  │ │
│ │             ──→ Lambda ──→ Jira (create incident ticket)        │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ INVESTIGATION:                                                      │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Detective (relationship graphs — who did what else)             │ │
│ │ Athena + CloudTrail (forensic API queries)                      │ │
│ │ Athena + VPC Flow Logs (network forensics)                      │ │
│ │ Loki (application + Falco logs)                                 │ │
│ │ Jaeger/Tempo (distributed trace analysis)                       │ │
│ └─────────────────────────────────────────────────────────────────┘ │
│                                                                     │
│ AUTO-REMEDIATION RULES:                                             │
│ ┌─────────────────────────────────────────────────────────────────┐ │
│ │ Finding                          │ Action                       │ │
│ │ ─────────────────────────────────┼──────────────────────────────│ │
│ │ Crypto mining (EC2)              │ Quarantine SG + page         │ │
│ │ S3 bucket public                 │ Block public access          │ │
│ │ IAM access key exposed           │ Disable key + notify         │ │
│ │ SG allows 0.0.0.0/0 SSH          │ Remove rule + notify         │ │
│ │ Unauthorized region activity     │ Disable credentials          │ │
│ │ RDS publicly accessible          │ Disable public access        │ │
│ └──────────────────────────────────┴──────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Part 4: AWS Config — Continuous Compliance Monitoring

```text
AWS Config continuously monitors your AWS resources for
configuration compliance. Unlike Security Hub (which checks
periodically), Config tracks EVERY configuration change in real time.

Config answers: "What is the current configuration of every
resource, and does it comply with our rules?"

┌───────────────────────────────────────────────────────────┐
│ AWS Config                                                │
│                                                           │
│ RECORDS:                                                  │
│ - Configuration items (every resource, every change)      │
│ - Configuration history (how resources changed over time) │
│ - Relationships (SG → EC2, Subnet → VPC, etc.)            │
│                                                           │
│ EVALUATES:                                                │
│ - Managed rules (150+ pre-built by AWS)                   │
│ - Custom rules (Lambda-based or Guard policy language)    │
│                                                           │
│ REMEDIATES:                                               │
│ - SSM Automation documents (auto-fix non-compliant)       │
│ - Manual remediation (notify, create ticket)              │
└───────────────────────────────────────────────────────────┘
```

```hcl
# AWS Config — recorder and key rules

resource "aws_config_configuration_recorder" "main" {
  name     = "novamart-config"
  role_arn = aws_iam_role.config.arn

  recording_group {
    all_supported                 = true
    include_global_resource_types = true
    # Records ALL resource types in ALL regions
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "novamart-config"
  s3_bucket_name = aws_s3_bucket.config.id
  s3_key_prefix  = "config"
  sns_topic_arn  = aws_sns_topic.config_changes.arn

  snapshot_delivery_properties {
    delivery_frequency = "Six_Hours"
  }

  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
  depends_on =[aws_config_delivery_channel.main]
}

# ─── CONFIG RULES ───

# Rule: EBS volumes must be encrypted
resource "aws_config_config_rule" "ebs_encrypted" {
  name = "ebs-volumes-encrypted"

  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }

  depends_on =[aws_config_configuration_recorder.main]
}

# Rule: S3 buckets must have versioning
resource "aws_config_config_rule" "s3_versioning" {
  name = "s3-bucket-versioning"

  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_VERSIONING_ENABLED"
  }
}

# Rule: RDS must not be publicly accessible
resource "aws_config_config_rule" "rds_not_public" {
  name = "rds-not-publicly-accessible"

  source {
    owner             = "AWS"
    source_identifier = "RDS_INSTANCE_PUBLIC_ACCESS_CHECK"
  }
}

# Rule: Security Groups must not allow 0.0.0.0/0 SSH
resource "aws_config_config_rule" "no_unrestricted_ssh" {
  name = "no-unrestricted-ssh"

  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"
  }
}

# Rule: IAM users must have MFA enabled
resource "aws_config_config_rule" "iam_mfa" {
  name = "iam-user-mfa-enabled"

  source {
    owner             = "AWS"
    source_identifier = "IAM_USER_MFA_ENABLED"
  }
}

# Rule: CloudTrail must be enabled and encrypted
resource "aws_config_config_rule" "cloudtrail_enabled" {
  name = "cloudtrail-enabled"

  source {
    owner             = "AWS"
    source_identifier = "CLOUD_TRAIL_ENABLED"
  }
}

# Auto-remediation: If SG allows SSH from 0.0.0.0/0, auto-remove the rule
resource "aws_config_remediation_configuration" "remove_ssh_open" {
  config_rule_name = aws_config_config_rule.no_unrestricted_ssh.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-DisablePublicAccessForSecurityGroup"

  parameter {
    name           = "GroupId"
    resource_value = "RESOURCE_ID"
  }

  automatic                  = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60
}
```

---

## Self-Evaluation

| Sub-Topic | Covered? | Depth | Notes |
|-----------|----------|-------|-------|
| DDoS attack types (L3/L4/L7) | ✅ | 10/10 | All types with defense strategies |
| AWS Shield (Standard + Advanced) | ✅ | 10/10 | Terraform, DRT, cost protection, proactive engagement |
| AWS WAF architecture | ✅ | 10/10 | Web ACL structure, WCU, evaluation order |
| WAF managed rules (5 groups) | ✅ | 10/10 | IP reputation, CRS, SQLi, known bad, bot control |
| WAF custom rules | ✅ | 10/10 | Rate limiting (per-IP, per-endpoint), geo-blocking, size constraints |
| WAF rule_action_override | ✅ | 10/10 | False positive handling pattern |
| WAF COUNT-first strategy | ✅ | 10/10 | Phase-by-phase rollout |
| Cloudflare + AWS WAF dual layer | ✅ | 10/10 | Why both, ALB restriction to CF IPs |
| WAF failure modes | ✅ | 10/10 | 5 failure modes including bypass and WCU |
| WAF logging (cost-aware) | ✅ | 10/10 | Filter, redact, lifecycle |
| VPC architecture (3-tier) | ✅ | 10/10 | CIDR design, /18 for EKS justification |
| Security Groups design | ✅ | 10/10 | SG-to-SG references, full NovaMart SGs |
| K8s NetworkPolicies | ✅ | 10/10 | Default deny, DNS egress, per-service rules |
| NetworkPolicy gotchas (5) | ✅ | 10/10 | CNI support, additive, DNS, Istio, namespace labels |
| Istio security (mTLS, AuthzPolicy) | ✅ | 10/10 | PeerAuthentication + AuthorizationPolicy |
| NetworkPolicy vs Istio comparison | ✅ | 10/10 | L3/L4 vs L7, why both needed |
| GuardDuty architecture | ✅ | 10/10 | All data sources, finding types |
| GuardDuty EKS integration | ✅ | 10/10 | Audit logs + runtime monitoring |
| GuardDuty → auto-response pipeline | ✅ | 10/10 | EventBridge → Lambda with full Python code |
| Quarantine SG pattern | ✅ | 10/10 | Auto-isolate with forensic preservation |
| GuardDuty failure modes | ✅ | 10/10 | 5 failure modes |
| Security Hub | ✅ | 9/10 | Compliance standards, finding aggregation |
| CloudTrail configuration | ✅ | 10/10 | Multi-region, data events, tamper-proof S3 |
| CloudTrail forensic queries (Athena) | ✅ | 10/10 | 5 production queries |
| VPC Flow Logs | ✅ | 10/10 | Custom format, Athena queries, 60s aggregation |
| VPC Flow Log forensic queries | ✅ | 10/10 | 4 production queries |
| AWS Detective | ✅ | 9/10 | Graph investigation concept, integration |
| AWS Config | ✅ | 10/10 | Rules, auto-remediation, Terraform |
| NovaMart detection architecture | ✅ | 10/10 | Complete pipeline diagram |

**Known gaps:**
- Didn't cover AWS Firewall Manager (centralized SG/WAF/Shield management for Organizations) — mentioned but not detailed
- Didn't cover AWS Network Firewall (managed IDS/IPS for VPC) — relevant for egress inspection but NovaMart uses Istio + NetworkPolicies
- Didn't cover AWS Macie (S3 sensitive data discovery) — covered in concept but no Terraform
- Didn't cover CloudTrail Lake (managed query service) — used Athena instead (cheaper, more flexible)
- Didn't cover WAF Bot Control Targeted mode details (ML-based, higher WCU, more expensive)
- Didn't cover Cloudflare-specific configuration in depth (focused on AWS side)
- Didn't cover DNSSEC for Route53

**Honest self-score: 9.4/10** — Comprehensive coverage of all five network security layers from edge (DDoS/WAF) through VPC to pod-level, plus the full AWS detection/response stack with production Terraform, forensic queries, and automated response code. Gaps are in adjacent AWS services (Firewall Manager, Macie, Network Firewall) that complement but don't replace the core architecture covered.

---

## Quick Reference Card

```text
PERIMETER DEFENSE
─────────────────
Shield Standard: Free, L3/L4, automatic
Shield Advanced: $3K/mo, L7 (with WAF), DRT, cost protection, metrics
WAF: L7 rules on ALB/CloudFront. Web ACL → Rule Groups → Rules
  Managed rules: IP reputation, CRS (OWASP), SQLi, Known Bad, Bot Control
  Custom rules: Rate limiting, geo-blocking, URI/header matching
  WCU limit: 1500 per Web ACL
  Deploy strategy: COUNT → selective BLOCK → full BLOCK
  NEVER deploy rules in BLOCK without COUNT period first
Cloudflare + AWS WAF: Dual layer. Restrict ALB to CF IPs only.

NETWORK SEGMENTATION
────────────────────
VPC: 3-tier (public/private/data). /18 for EKS private subnets (IP exhaustion).
Security Groups: Stateful, allow-only, SG-to-SG references (not CIDR).
NACLs: Stateless, ordered rules, ephemeral port trap.
VPC Endpoints: Gateway (S3/DynamoDB free), Interface ($7.50/AZ/mo).
  ECR endpoints CRITICAL — saves massive NAT GW costs.
NetworkPolicies: Default deny all → explicit allow. ALWAYS allow DNS egress.
  Requires Calico/Cilium — VPC CNI alone doesn't enforce.
Istio AuthzPolicy: L7 (method+path), mTLS identity, complement NetworkPolicies.

DETECTION & RESPONSE
────────────────────
GuardDuty: Threat detection. CloudTrail + Flow Logs + DNS + EKS + RDS + S3.
  Enable ALL regions. Auto-response via EventBridge → Lambda.
  Quarantine pattern: Replace SGs, tag, snapshot, notify. DON'T terminate.
Security Hub: Finding aggregator + compliance dashboard (CIS/PCI/NIST).
CloudTrail: Every API call logged. S3 + Object Lock for tamper-proof.
  Athena queries for forensics. Multi-region. Enable data events for sensitive S3.
VPC Flow Logs: Network metadata. 60s aggregation for production.
  Custom format for NAT visibility. Athena for network forensics.
Detective: Relationship graphs. Navigate from finding to full attack chain.
Config: Continuous compliance. 150+ managed rules. Auto-remediation via SSM.

RESPONSE PRIORITY ORDER
───────────────────────
1. Contain (quarantine SG, NetworkPolicy, disable credentials)
2. Preserve (snapshot, tag, DON'T terminate)
3. Investigate (Detective graphs, CloudTrail + Athena, Flow Logs)
4. Remediate (patch, rotate, fix config)
5. Recover (restore from known-good, verify)
6. Learn (postmortem, improve detection, add automation)
```

---

## Progress Tracker

```text
PART 1: FOUNDATIONS (Phases 0-6)
  Phase 0: Linux Deep Dive                     ████████████████████ 100% ✅
  Phase 1: Networking                          ████████████████████ 100% ✅
  Phase 2: Git, Docker, K8s                    ████████████████████ 100% ✅
  Phase 3: CI/CD Pipelines                     ████████████████████ 100% ✅
  Phase 4: IaC (Terraform/Ansible)             ████████████████████ 100% ✅
  Phase 5: Observability & SRE                 ████████████████████ 100% ✅
  Phase 6: Security, Compliance, AWS           ████████████████░░░░  80%
    → Lesson 1: AWS IAM Deep Dive ✅
    → Lesson 2: Secrets Management ✅
    → Lesson 3: K8s Security (OPA, PSA, Falco, Supply Chain) ✅
    → Lesson 4: Network Security & AWS Security Services ✅ (Qs PENDING)
    → Lesson 5: Compliance & Governance

Overall: ~68%
```

---

## Retention Questions

### Q1: DDoS Attack During Black Friday 🔥

**Scenario:** Black Friday, 8 AM. Traffic is 3x normal. At 8:15 AM, traffic spikes to 15x normal. CloudWatch shows ALB 5xx errors climbing from 0.1% to 12%. Latency p99 goes from 200ms to 8 seconds. Your monitoring shows:

- Cloudflare dashboard: 40 Gbps of traffic (vs normal 5 Gbps)
- ALB request count: 50K/sec (vs normal 8K/sec)
- EKS pod CPU: most pods at 85-95%
- WAF BlockedRequests metric: spiking with `aws-ip-reputation` rule
- Some blocked requests are from legitimate customer IPs (identified by matching against known customer accounts)

1. Is this a DDoS attack, legitimate Black Friday traffic, or both? How do you determine the difference? What specific metrics and log fields distinguish malicious from legitimate traffic?
2. The WAF IP reputation rule is blocking 2,000 requests/min, but 15% of those are confirmed legitimate customers whose IPs are on a shared threat intelligence list (they're behind corporate proxies flagged by previous abuse). What do you do RIGHT NOW? Give the exact WAF configuration change.
3. You confirm there's a genuine L7 DDoS component — HTTP flood targeting `/api/v1/search` with randomized query strings to bypass caching. Design the WAF rule that stops this specific attack pattern without blocking legitimate search traffic. Include the exact Terraform/WAF rule.
4. After containing the attack, what Shield Advanced features should have made this easier? What was missing from NovaMart's DDoS preparation?

### Q2: Lateral Movement Investigation 🔥

**Scenario:** Wednesday 2 PM. GuardDuty finding: `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.InsideAWS` — severity 8 (HIGH). The finding indicates that EC2 instance credentials from an EKS worker node (`i-0abc123def456`) are being used from a DIFFERENT EC2 instance (`i-0xyz789`) in a different VPC.

1. Explain what this finding means technically. How does an attacker get EC2 instance credentials from an EKS worker node, and why does GuardDuty flag usage from a different instance?
2. Your first 15 minutes — walk through every investigation step with exact commands/queries. You need to determine: what credentials were stolen, what the attacker did with them, and whether they're still active.
3. The CloudTrail investigation reveals: the stolen credentials were used to call `sts:AssumeRole` to assume a role in the production account, then `secretsmanager:GetSecretValue` to read database credentials. The VPC Flow Logs show outbound traffic from `i-0xyz789` to an external IP on port 443. Trace the full attack chain and identify every control that failed.
4. Design the architecture changes that prevent this attack vector. Cover: IMDS, IRSA, VPC segmentation, and GuardDuty response automation.

### Q3: NetworkPolicy Debugging 🔥

**Scenario:** A new microservice `recommendation-svc` was deployed in the `search` namespace on Tuesday. By Thursday, the on-call engineer gets a report: `recommendation-svc` can reach the payment database (RDS PostgreSQL on port 5432 in the data subnet). This SHOULD NOT be possible — `recommendation-svc` should only communicate with `search-svc` and Redis.

The existing NetworkPolicies in the `search` namespace:
```yaml
# Default deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: search
spec:
  podSelector: {}
  policyTypes: ["Ingress", "Egress"]

# Allow DNS
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: search
spec:
  podSelector: {}
  policyTypes:["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

# search-svc egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: search-svc-egress
  namespace: search
spec:
  podSelector:
    matchLabels:
      app: search-svc
  policyTypes: ["Egress"]
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.240.0/22
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - ipBlock:
            cidr: 10.0.244.0/22
      ports:
        - protocol: TCP
          port: 6379
```

1. Explain exactly WHY `recommendation-svc` can reach the payment database, despite the `default-deny-all` NetworkPolicy. (This is a trap — think carefully about how NetworkPolicies work.)
2. The developer says "I see default-deny-all, so how can anything reach anything?" Explain the precise mechanism by which the `search-svc-egress` policy affects `recommendation-svc`'s network access.
3. Write the corrected NetworkPolicies that properly isolate `recommendation-svc` to only `search-svc` (port 8080) and Redis (port 6379). Ensure `search-svc` retains its existing database access.
4. How would you have detected this misconfiguration BEFORE it reached production? Design the validation pipeline (tools, tests, CI integration).

### Q4: Security Incident — Full Response 🔥

**Scenario:** Sunday 6 AM. Multiple alerts fire simultaneously:

- GuardDuty: `Persistence:IAMUser/AnomalousBehavior` — a service account `ci-deployer` is calling `iam:AttachRolePolicy` at 6 AM Sunday (it normally only runs weekdays during deploys)
- CloudTrail shows: `ci-deployer` attached `AdministratorAccess` to itself, then called `ec2:DescribeInstances`, `secretsmanager:ListSecrets`, `secretsmanager:GetSecretValue` for 8 different secrets
- Security Hub: New CRITICAL finding — `ci-deployer` access key is flagged as potentially compromised
- VPC Flow Logs: The API calls originate from an IP not in NovaMart's known CIDR ranges

1. Write your first 10 actions in order with exact commands. The attacker has `AdministratorAccess` on a service account — this is a worst-case scenario. Time is critical.
2. After containing the immediate threat, how do you determine the FULL blast radius? The attacker had AdministratorAccess for approximately 45 minutes. What specific evidence do you search for, and where?
3. The 8 secrets accessed include RDS master credentials for 3 databases (payments, orders, users) and API keys for Stripe, SendGrid, and Twilio. Design the rotation plan — what order, what dependencies, what can be parallel, what needs coordination?
4. The root cause is that `ci-deployer` had overly permissive IAM policies (`iam:Attach*`, `iam:Put*`) and its access key was stored in a Jenkins credential store that was backed up to an S3 bucket with public read access. Design the systemic fixes that address every link in this chain.

---

# NovaMart AWS & Kubernetes Security — Advanced Incident Response

---

## Q1: DDoS Attack During Black Friday 🔥

### 1. Distinguishing DDoS from Legitimate Traffic — Or Both

**Verdict: It's BOTH.** Black Friday legitimately drives 3x traffic, but 15x with the specific characteristics described indicates a DDoS layered on top of real traffic. Here's how to determine this definitively:

**Metric 1: Request Distribution by URI Path**

```sql
# Query ALB access logs in Athena
SELECT 
  target_group_arn,
  request_url,
  COUNT(*) as request_count,
  COUNT(DISTINCT client_ip) as unique_ips,
  COUNT(*) / COUNT(DISTINCT client_ip) as requests_per_ip
FROM alb_logs
WHERE time > now() - interval '30' minute
GROUP BY target_group_arn, request_url
ORDER BY request_count DESC
LIMIT 50;
```

- **Legitimate Black Friday:** Traffic distributed across `/product/*`, `/cart`, `/checkout`, `/api/v1/search` — following normal user journey patterns with a high conversion funnel ratio
- **DDoS:** One or two endpoints hammered disproportionately (like `/api/v1/search`), with requests that don't follow any user journey

**Metric 2: Session Behavior Analysis**

```sql
# Check if "users" have realistic session patterns
SELECT 
  client_ip,
  COUNT(*) as total_requests,
  COUNT(DISTINCT request_url) as unique_urls,
  MIN(time) as first_seen,
  MAX(time) as last_seen,
  APPROX_DISTINCT(request_url) as path_diversity
FROM alb_logs
WHERE time > now() - interval '15' minute
GROUP BY client_ip
HAVING COUNT(*) > 100
ORDER BY total_requests DESC;
```

- **Legitimate users:** 5-50 requests over minutes, diverse paths (homepage → search → product → cart → checkout), carry cookies/session tokens
- **Bot/DDoS:** Hundreds of requests per second per IP, low path diversity OR highly randomized paths, no cookies, no progressive session

**Metric 3: HTTP Headers & Fingerprinting**

```sql
# In WAF Sampled Requests or ALB logs — check User-Agent distribution
SELECT 
  "user_agent",
  COUNT(*) as count,
  COUNT(DISTINCT client_ip) as unique_ips
FROM alb_logs
WHERE time > now() - interval '15' minute
GROUP BY "user_agent"
ORDER BY count DESC
LIMIT 20;
```

- **Legitimate:** Diverse user agents (Chrome, Safari, iOS, Android — matches NovaMart's customer demographics)
- **DDoS:** Clustered user agents (same bot UA, or suspiciously uniform versions), or randomized UAs that don't match real browser TLS fingerprints

**Metric 4: Geographic Distribution**

```sql
# Cloudflare analytics or ALB + GeoIP
SELECT 
  geo_country,
  COUNT(*) as requests,
  COUNT(*) * 100.0 / SUM(COUNT(*)) OVER () as percentage
FROM alb_logs
WHERE time > now() - interval '15' minute
GROUP BY geo_country
ORDER BY requests DESC;
```

- **Legitimate:** NovaMart's market footprint (US-heavy if US company, maybe 70% US, 10% UK, etc.)
- **DDoS:** Sudden traffic from countries NovaMart doesn't serve, or impossibly uniform geographic distribution

**Metric 5: TLS Fingerprint (JA3/JA4)**

```bash
# If Cloudflare Bot Management is enabled:
# Check JA3 hash distribution
# Legitimate browsers have well-known JA3 fingerprints
# Botnets often have uniform or unusual TLS fingerprints
```

**Metric 6: Request Timing Patterns**

```sql
# Check inter-arrival times
# Legitimate humans: variable intervals (thinking time between clicks)
# Bots: metronomic regularity or perfectly randomized (paradoxically detectable)
SELECT 
  client_ip,
  time,
  LAG(time) OVER (PARTITION BY client_ip ORDER BY time) as prev_time,
  EXTRACT(MILLISECOND FROM time - LAG(time) OVER (PARTITION BY client_ip ORDER BY time)) as interval_ms,
  STDDEV(EXTRACT(MILLISECOND FROM time - LAG(time) OVER (PARTITION BY client_ip ORDER BY time))) 
    OVER (PARTITION BY client_ip) as interval_stddev
FROM alb_logs
WHERE time > now() - interval '5' minute
  AND client_ip IN (SELECT client_ip FROM alb_logs GROUP BY client_ip HAVING COUNT(*) > 50);
```

---

### 2. Unblocking Legitimate Customers Blocked by IP Reputation — RIGHT NOW

The problem: 15% of WAF blocks are false positives — real customers behind corporate proxies whose IPs appear on shared threat intel lists. We can't disable the IP reputation rule entirely during a DDoS attack, but we can create an exception path.

**Immediate Action — WAF IP Set Override:**

```bash
# Step 1: Create an IP set of confirmed legitimate customer IPs
# Pull the IPs from your customer database/order history correlation

aws wafv2 create-ip-set \
  --name "verified-customer-ips-emergency" \
  --scope REGIONAL \
  --ip-address-version IPV4 \
  --addresses \
    "203.0.113.0/24" \
    "198.51.100.0/24" \
    "192.0.2.0/24" \
  --region us-east-1

# Capture the IP set ARN
IP_SET_ARN=$(aws wafv2 list-ip-sets --scope REGIONAL --region us-east-1 \
  --query "IPSets[?Name=='verified-customer-ips-emergency'].ARN" --output text)
```

```bash
# Step 2: Get current Web ACL config
aws wafv2 get-web-acl \
  --name novamart-web-acl \
  --scope REGIONAL \
  --id <WEB_ACL_ID> \
  --region us-east-1 > /tmp/current-webacl.json

LOCK_TOKEN=$(jq -r '.LockToken' /tmp/current-webacl.json)
```

```bash
# Step 3: Insert a PRIORITY-1 rule that ALLOWS verified customer IPs
# BEFORE the IP reputation rule evaluates them
# This is done by adding a rule with lower numeric priority (evaluated first)

aws wafv2 update-web-acl \
  --name novamart-web-acl \
  --scope REGIONAL \
  --id <WEB_ACL_ID> \
  --lock-token "$LOCK_TOKEN" \
  --default-action '{"Allow":{}}' \
  --rules '[
    {
      "Name": "AllowVerifiedCustomers",
      "Priority": 0,
      "Statement": {
        "IPSetReferenceStatement": {
          "ARN": "'"$IP_SET_ARN"'"
        }
      },
      "Action": {
        "Allow": {
          "CustomRequestHandling": {
            "InsertHeaders":[
              {
                "Name": "x-waf-bypass",
                "Value": "verified-customer-emergency"
              }
            ]
          }
        }
      },
      "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "VerifiedCustomerBypass"
      }
    },
    ... (existing rules with their original priorities)
  ]' \
  --region us-east-1
```

**Alternatively, if modifying the Web ACL JSON is too risky under pressure, use a scoped-down override on the IP reputation rule itself:**

```json
# Create a rule that combines: 
# IF (IP reputation match) AND (NOT in verified customer list) → BLOCK
# This effectively whitelists verified customers from the IP reputation rule

# This is done with a nested AND/NOT statement:
{
  "Name": "IPReputationWithCustomerException",
  "Priority": 5,
  "Statement": {
    "AndStatement": {
      "Statements":[
        {
          "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesAmazonIpReputationList"
          }
        },
        {
          "NotStatement": {
            "Statement": {
              "IPSetReferenceStatement": {
                "ARN": "$IP_SET_ARN"
              }
            }
          }
        }
      ]
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "IPRepWithException"
  }
}
```

> ⚠️ **Note:** AWS Managed Rule Groups can't be directly nested in AND statements. The actual implementation requires setting the managed rule group to COUNT mode and then evaluating the label:

```json
# The CORRECT approach for AWS Managed Rules:
# 1. Set IP Reputation rule group to COUNT (not BLOCK)
# 2. Add a custom rule that checks the label AND excludes verified IPs

{
  "Name": "AWSIPReputation",
  "Priority": 5,
  "Statement": {
    "ManagedRuleGroupStatement": {
      "VendorName": "AWS",
      "Name": "AWSManagedRulesAmazonIpReputationList"
    }
  },
  "OverrideAction": { "Count": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "IPReputationCount"
  }
},
{
  "Name": "BlockBadIPsExceptVerifiedCustomers",
  "Priority": 6,
  "Statement": {
    "AndStatement": {
      "Statements":[
        {
          "LabelMatchStatement": {
            "Scope": "LABEL",
            "Key": "awswaf:managed:aws:amazon-ip-list:AWSManagedIPReputationList"
          }
        },
        {
          "NotStatement": {
            "Statement": {
              "IPSetReferenceStatement": {
                "ARN": "$IP_SET_ARN"
              }
            }
          }
        }
      ]
    }
  },
  "Action": { "Block": {} },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BlockBadIPExceptVerified"
  }
}
```

This pattern: managed rule → COUNT → label match + exception → BLOCK is the canonical AWS WAF pattern for adding exceptions to managed rules.

---

### 3. L7 HTTP Flood Mitigation — `/api/v1/search` with Randomized Query Strings

The attack pattern: `GET /api/v1/search?q=<random_string>` — each request has a unique query string to defeat caching.

**Analysis of attack characteristics vs. legitimate traffic:**

| Property | Legitimate Search | DDoS Search |
|----------|-------------------|-------------|
| Rate per IP | 1-5 searches/min | 100+ searches/sec |
| Query format | Real words, brands, products | Random strings, gibberish |
| Session context | Has cookies, referrer from NovaMart | No cookies, no referrer |
| Follow-up | Clicks on search results | No follow-up requests |
| Query length | 2-50 chars, natural language | Uniform length or very long |

**Terraform WAF Rule:**

```hcl
# Rate-based rule: limit search requests per IP
resource "aws_wafv2_rule_group" "anti_ddos_search" {
  name        = "anti-ddos-search-flood"
  description = "Mitigates L7 HTTP flood targeting /api/v1/search"
  scope       = "REGIONAL"
  capacity    = 250

  # Rule 1: Rate limit search requests per IP
  rule {
    name     = "SearchRateLimit"
    priority = 1

    action {
      block {
        custom_response {
          response_code = 429
          custom_response_body_key = "rate-limited"
        }
      }
    }

    statement {
      rate_based_statement {
        limit              = 300  # 300 requests per 5-minute window = 1/sec avg
        aggregate_key_type = "IP"

        scope_down_statement {
          byte_match_statement {
            search_string         = "/api/v1/search"
            field_to_match {
              uri_path {}
            }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
            positional_constraint = "STARTS_WITH"
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SearchRateLimit"
    }
  }

  # Rule 2: Block search requests without valid session cookie
  # Legitimate users will have a session cookie from browsing the site
  rule {
    name     = "SearchRequiresSession"
    priority = 2

    action {
      block {}
    }

    statement {
      and_statement {
        statement {
          byte_match_statement {
            search_string = "/api/v1/search"
            field_to_match {
              uri_path {}
            }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
            positional_constraint = "STARTS_WITH"
          }
        }

        statement {
          not_statement {
            statement {
              byte_match_statement {
                search_string = "session_id="
                field_to_match {
                  single_header {
                    name = "cookie"
                  }
                }
                text_transformation {
                  priority = 0
                  type     = "LOWERCASE"
                }
                positional_constraint = "CONTAINS"
              }
            }
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SearchNoSession"
    }
  }

  # Rule 3: Block search queries that look like random strings
  # Legitimate searches: words, spaces, dashes, product codes
  # Attack queries: random alphanumeric, no spaces, unusual character patterns
  rule {
    name     = "SearchQueryValidation"
    priority = 3

    action {
      block {}
    }

    statement {
      and_statement {
        statement {
          byte_match_statement {
            search_string = "/api/v1/search"
            field_to_match {
              uri_path {}
            }
            text_transformation {
              priority = 0
              type     = "LOWERCASE"
            }
            positional_constraint = "STARTS_WITH"
          }
        }

        # Block if query string exceeds 200 chars
        # (legitimate searches are rarely > 100 chars)
        statement {
          size_constraint_statement {
            comparison_operator = "GT"
            size                = 200
            field_to_match {
              query_string {}
            }
            text_transformation {
              priority = 0
              type     = "NONE"
            }
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SearchQueryTooLong"
    }
  }

  # Rule 4: Challenge suspected bots with CAPTCHA
  # If an IP hits search more than 50 times in 5 min without
  # typical browser headers, issue a CAPTCHA challenge
  rule {
    name     = "SearchBotChallenge"
    priority = 4

    action {
      captcha {}
    }

    statement {
      rate_based_statement {
        limit              = 50
        aggregate_key_type = "IP"

        scope_down_statement {
          and_statement {
            statement {
              byte_match_statement {
                search_string = "/api/v1/search"
                field_to_match {
                  uri_path {}
                }
                text_transformation {
                  priority = 0
                  type     = "LOWERCASE"
                }
                positional_constraint = "STARTS_WITH"
              }
            }

            statement {
              not_statement {
                statement {
                  byte_match_statement {
                    search_string = "text/html"
                    field_to_match {
                      single_header {
                        name = "accept"
                      }
                    }
                    text_transformation {
                      priority = 0
                      type     = "LOWERCASE"
                    }
                    positional_constraint = "CONTAINS"
                  }
                }
              }
            }
          }
        }
      }
    }

    visibility_config {
      sampled_requests_enabled   = true
      cloudwatch_metrics_enabled = true
      metric_name                = "SearchBotCaptcha"
    }
  }

  visibility_config {
    sampled_requests_enabled   = true
    cloudwatch_metrics_enabled = true
    metric_name                = "AntiDDOSSearchFlood"
  }

  custom_response_body {
    key          = "rate-limited"
    content      = "{\"error\": \"Too many requests. Please try again shortly.\", \"retry_after\": 30}"
    content_type = "APPLICATION_JSON"
  }
}

# Associate with Web ACL
resource "aws_wafv2_web_acl_association" "anti_ddos" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

**Additionally, at the application level:**

```bash
# Deploy a Cloudflare rate limiting rule (if using Cloudflare in front)
# This catches traffic before it even reaches AWS

# Cloudflare rule (via API):
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/rulesets" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Search DDoS Mitigation",
    "kind": "zone",
    "phase": "http_ratelimit",
    "rules":[
      {
        "action": "block",
        "expression": "http.request.uri.path contains \"/api/v1/search\" and rate.requests > 10",
        "ratelimit": {
          "characteristics": ["ip.src"],
          "period": 60,
          "requests_per_period": 10,
          "mitigation_timeout": 600
        }
      }
    ]
  }'
```

---

### 4. Shield Advanced — What Was Missing

**Shield Advanced features that should have been in place:**

**1. Proactive Engagement (DRT — DDoS Response Team)**

Shield Advanced customers get access to the AWS DDoS Response Team who proactively reach out during detected events. NovaMart should have:

```hcl
resource "aws_shield_proactive_engagement" "main" {
  enabled = true

  emergency_contact {
    email_address = "security-oncall@novamart.com"
    phone_number  = "+1-555-0123"
    contact_notes = "NOC / Security On-Call — 24/7"
  }

  emergency_contact {
    email_address = "platform-lead@novamart.com"
    phone_number  = "+1-555-0456"
    contact_notes = "Platform Engineering Lead — escalation"
  }
}
```

**2. Shield Advanced Protections on All Critical Resources**

```hcl
# Protect ALB
resource "aws_shield_protection" "alb" {
  name         = "novamart-alb-protection"
  resource_arn = aws_lb.main.arn
}

# Protect CloudFront distribution
resource "aws_shield_protection" "cloudfront" {
  name         = "novamart-cloudfront-protection"
  resource_arn = aws_cloudfront_distribution.main.arn
}

# Protect Route53 hosted zone
resource "aws_shield_protection" "route53" {
  name         = "novamart-route53-protection"
  resource_arn = "arn:aws:route53:::hostedzone/${aws_route53_zone.main.zone_id}"
}

# Protect Elastic IPs (for NAT Gateways)
resource "aws_shield_protection" "nat_eip" {
  for_each     = aws_eip.nat
  name         = "novamart-nat-eip-${each.key}"
  resource_arn = each.value.arn
}
```

**3. Shield Advanced Automatic Application Layer DDoS Mitigation**

```hcl
resource "aws_shield_application_layer_automatic_response" "alb" {
  resource_arn = aws_lb.main.arn
  action       = "COUNT"  # Start with COUNT, move to BLOCK after tuning

  # Shield Advanced will automatically create WAF rules in response
  # to detected L7 attacks, based on traffic patterns
}
```

**4. Health-Based Detection**

```hcl
# Create Route53 health check that Shield uses for DDoS detection
resource "aws_route53_health_check" "alb" {
  fqdn              = "api.novamart.com"
  port               = 443
  type               = "HTTPS"
  resource_path      = "/health"
  failure_threshold  = 3
  request_interval   = 10

  tags = {
    Name = "novamart-api-health"
  }
}

# Associate health check with Shield protection
resource "aws_shield_protection_health_check_association" "alb" {
  health_check_arn     = aws_route53_health_check.alb.arn
  shield_protection_id = aws_shield_protection.alb.id
}
```

With health-based detection, Shield Advanced can detect attacks based on the health of your application, not just volumetric thresholds. It would have detected the spike in 5xx errors and latency increase and triggered automated mitigation much sooner.

**5. What Was Missing From NovaMart's DDoS Preparation — Complete Gap Analysis:**

| Preparation Gap | Impact During Incident | Fix |
|---|---|---|
| No Shield Advanced auto-mitigation for L7 | Manual WAF rule creation under pressure | Enable application-layer auto-response |
| No pre-built "DDoS runbook" with pre-approved WAF rules | Scrambling to write rules at 8:15 AM on Black Friday | Prepare rate-limit rules in `COUNT` mode, flip to `BLOCK` |
| No verified-customer IP allowlist | 15% false positive rate on legitimate customers | Maintain dynamic allowlist from customer session data |
| No health-based detection | Shield detected volume but not application impact | Associate Route53 health checks |
| No DRT proactive engagement | Had to handle alone at 8 AM | Register emergency contacts for proactive DRT engagement |
| No pre-scaled infrastructure | Pods at 85-95% CPU from legitimate 3x traffic alone | HPA pre-scaling before Black Friday, min replicas 3x |
| No separate search endpoint scaling | Search DDoS impacts all services equally | Isolate search-svc behind its own ALB/target group |
| No Cloudflare Bot Management | No JA3/browser fingerprinting capability | Enable Bot Management for automated bot detection |
| No DDoS simulation/game day | Team unfamiliar with response procedures | Run quarterly DDoS simulations with Shield Advanced |

**Pre-Black Friday Checklist (should have been executed 2 weeks prior):**

```bash
# 1. Pre-scale all HPA minimums
kubectl patch hpa -n orders order-svc --type merge \
  -p '{"spec":{"minReplicas": 15}}'  # 3x normal minimum

# 2. Pre-warm ALB (request via AWS support for Shield Advanced customers)
# AWS can pre-provision ALB capacity if you give them advance notice

# 3. Pre-deploy WAF rules in COUNT mode for rapid activation
# 4. Verify Shield Advanced protections on all resources
# 5. Test DRT emergency contact information
# 6. Brief on-call team with DDoS runbook
```

---

## Q2: Lateral Movement Investigation 🔥

### 1. Technical Explanation of the Finding

**How the credentials were obtained:**

Every EC2 instance (including EKS worker nodes) has access to the **Instance Metadata Service (IMDS)** at `http://169.254.169.254`. When an EC2 instance has an IAM instance profile attached, any process on that instance can request temporary credentials:

```bash
# From inside ANY pod on the worker node (if IMDS is accessible):
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/<role-name>
# Returns:
# {
#   "AccessKeyId": "ASIA...",
#   "SecretAccessKey": "...",
#   "Token": "...",
#   "Expiration": "2024-11-27T14:30:00Z"
# }
```

The attacker likely:
1. Compromised a pod running on the EKS worker node (via RCE, SSRF, or a compromised container)
2. Queried IMDS from inside the pod — if **IMDSv2 was not enforced** or the **pod had network access to 169.254.169.254** (no NetworkPolicy blocking it), the pod could retrieve the node's IAM credentials
3. Exfiltrated the temporary credentials (AccessKeyId, SecretAccessKey, Session Token)
4. Used those credentials from a **different** EC2 instance (`i-0xyz789`) — possibly a pre-compromised machine or an attacker-controlled instance in a different AWS account that they pivoted into

**Why GuardDuty flags this:**

GuardDuty monitors CloudTrail and VPC Flow Logs. It knows that credentials issued to instance `i-0abc123def456` (via IMDS) should only be used from that instance's IP address. When the same temporary credentials appear in API calls from `i-0xyz789` (a different source IP), GuardDuty flags `InstanceCredentialExfiltration.InsideAWS` — the credentials are being used from inside AWS but not from the expected instance. This is a definitive indicator of credential theft.

---

### 2. First 15 Minutes — Investigation Steps

**Minute 0-1: Acknowledge and verify the finding**

```bash
# Get the full GuardDuty finding
aws guardduty get-findings \
  --detector-id $(aws guardduty list-detectors --query 'DetectorIds[0]' --output text) \
  --finding-ids <FINDING_ID> \
  --query 'Findings[0]' \
  --output json > /tmp/guardduty-finding.json

# Extract key details
cat /tmp/guardduty-finding.json | jq '{
  type: .Type,
  severity: .Severity,
  instanceId: .Resource.InstanceDetails.InstanceId,
  role: .Resource.AccessKeyDetails.PrincipalId,
  accessKeyId: .Resource.AccessKeyDetails.AccessKeyId,
  remoteIp: .Service.Action.AwsApiCallAction.RemoteIpDetails.IpAddressV4,
  remoteInstance: .Service.Action.AwsApiCallAction.RemoteAccountDetails,
  firstSeen: .Service.EventFirstSeen,
  lastSeen: .Service.EventLastSeen
}'
```

**Minute 1-3: IMMEDIATELY revoke the compromised credentials**

The node's IAM role has temporary credentials. You can't "delete" them, but you can revoke all active sessions:

```bash
# Step 1: Identify the IAM role attached to the worker node
ROLE_NAME=$(aws ec2 describe-instances \
  --instance-ids i-0abc123def456 \
  --query 'Reservations[0].Instances[0].IamInstanceProfile.Arn' \
  --output text | awk -F'/' '{print $NF}')

# The instance profile name maps to a role
ROLE_ARN=$(aws iam get-instance-profile \
  --instance-profile-name $ROLE_NAME \
  --query 'InstanceProfile.Roles[0].Arn' --output text)

echo "Compromised role: $ROLE_ARN"

# Step 2: Attach an inline policy that revokes ALL sessions issued before NOW
# This invalidates ALL existing temporary credentials for this role
REVOKE_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

aws iam put-role-policy \
  --role-name $ROLE_NAME \
  --policy-name "RevokeOldSessions-Emergency" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement":[
      {
        "Effect": "Deny",
        "Action": "*",
        "Resource": "*",
        "Condition": {
          "DateLessThan": {
            "aws:TokenIssueTime": "'"$REVOKE_TIME"'"
          }
        }
      }
    ]
  }'

echo "All sessions issued before $REVOKE_TIME are now denied"
```

> ⚠️ **Impact:** This will also revoke credentials for ALL legitimate pods on ALL nodes using this node role. Pods using IRSA (IAM Roles for Service Accounts) are NOT affected since they use separate role assumptions. Pods using the node role directly will temporarily lose AWS access until they refresh credentials from IMDS.

**Minute 3-5: Isolate the compromised node**

```bash
# Cordon the node — no new pods scheduled
kubectl cordon ip-10-0-1-xxx.ec2.internal  # node name for i-0abc123def456

# Isolate the node at network level via Security Group
# Get the node's security group
NODE_SG=$(aws ec2 describe-instances \
  --instance-ids i-0abc123def456 \
  --query 'Reservations[0].Instances[0].SecurityGroups[0].GroupId' \
  --output text)

# Create quarantine security group — deny all inbound and outbound
QUARANTINE_SG=$(aws ec2 create-security-group \
  --group-name "quarantine-$(date +%s)" \
  --description "Emergency quarantine for i-0abc123def456" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Revoke the default outbound rule
aws ec2 revoke-security-group-egress \
  --group-id $QUARANTINE_SG \
  --protocol all --cidr 0.0.0.0/0

# Replace node's security group with quarantine SG
aws ec2 modify-instance-attribute \
  --instance-id i-0abc123def456 \
  --groups $QUARANTINE_SG
```

**Minute 5-7: Investigate the attacker's instance**

```bash
# Who owns i-0xyz789? Is it in our account?
aws ec2 describe-instances \
  --instance-ids i-0xyz789 \
  --query 'Reservations[0].Instances[0].{
    VpcId: VpcId,
    SubnetId: SubnetId,
    PrivateIp: PrivateIpAddress,
    PublicIp: PublicIpAddress,
    IAMRole: IamInstanceProfile.Arn,
    State: State.Name,
    LaunchTime: LaunchTime,
    Tags: Tags
  }' 2>/dev/null

# If "not found" — the instance is in a DIFFERENT AWS account
# This is very serious — the attacker has their own AWS infrastructure
```

**Minute 7-10: Query CloudTrail for ALL actions taken with the stolen credentials**

```bash
# Get the access key ID from the GuardDuty finding
STOLEN_KEY=$(cat /tmp/guardduty-finding.json | jq -r '.Resource.AccessKeyDetails.AccessKeyId')

# Search CloudTrail for all API calls using this access key
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=$STOLEN_KEY \
  --start-time $(date -d '24 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ") \
  --end-time $(date -u +"%Y-%m-%dT%H:%M:%SZ") \
  --max-results 50 \
  --query 'Events[].{
    Time: EventTime,
    Name: EventName,
    Source: EventSource,
    SourceIP: CloudTrailEvent
  }' --output json > /tmp/stolen-creds-activity.json

# Parse the activity
cat /tmp/stolen-creds-activity.json | jq '.[].Name' | sort | uniq -c | sort -rn
```

```bash
# More detailed CloudTrail analysis with CloudWatch Logs Insights
# (if CloudTrail is shipping to CloudWatch)
aws logs start-query \
  --log-group-name "CloudTrail/ManagementEvents" \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, eventName, eventSource, sourceIPAddress, 
           requestParameters, responseElements, errorCode
    | filter userIdentity.accessKeyId = "'"$STOLEN_KEY"'"
    | sort @timestamp asc
  '
```

**Minute 10-12: Check for persistence mechanisms**

```bash
# Did the attacker create new IAM users, roles, or access keys?
cat /tmp/stolen-creds-activity.json | jq '
  [.[] | select(.Name | test("Create|Attach|Put|Add|Update"))] |
  sort_by(.Time)'

# Specifically check for:
# - CreateUser, CreateAccessKey (new backdoor accounts)
# - CreateRole, AttachRolePolicy (privilege escalation)
# - PutRolePolicy (inline policy changes)
# - CreateLoginProfile (console access)
# - AuthorizeSecurityGroupIngress (opening firewall)
# - RunInstances (new EC2 for persistence)
# - CreateFunction (Lambda backdoor)
```

**Minute 12-13: Check if the attacker pivoted to Kubernetes**

```bash
# The node role typically has EKS permissions
# Check if the attacker called EKS APIs
cat /tmp/stolen-creds-activity.json | jq '
  [.[] | select(.Name | test("eks|kubernetes|k8s|describe-cluster|get-token"))]'

# Check Kubernetes audit logs for API calls from the node's identity
# that seem unusual (listing secrets, creating pods, etc.)
kubectl logs -n kube-system -l component=kube-apiserver --since=24h 2>/dev/null | \
  grep "i-0abc123def456"
```

**Minute 13-15: Assess the second instance and VPC Flow Logs**

```bash
# Check VPC Flow Logs for traffic between the two instances
aws logs start-query \
  --log-group-name "VPCFlowLogs" \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, srcAddr, dstAddr, srcPort, dstPort, action, protocol, bytes
    | filter srcAddr = "10.x.x.x" or dstAddr = "10.x.x.x"
    | filter srcAddr != dstAddr
    | sort @timestamp asc
  '
# (replace 10.x.x.x with the IP of i-0xyz789)

# Check for data exfiltration — large outbound transfers
aws logs start-query \
  --log-group-name "VPCFlowLogs" \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, srcAddr, dstAddr, dstPort, bytes
    | filter srcAddr like "10.0."
    | filter not (dstAddr like "10.0.")
    | filter not (dstAddr like "172.16.")
    | stats sum(bytes) as total_bytes by srcAddr, dstAddr, dstPort
    | sort total_bytes desc
    | limit 20
  '
```

**Minute 15: Escalate and begin formal incident response**

```bash
# Post to #security-incidents:
# "🔴 ACTIVE COMPROMISE — Stolen EKS node credentials used from external instance.
# Credentials revoked. Node isolated. CloudTrail analysis shows [summary].
# Attacker performed: [list of API calls]. 
# Formal IR engaged. Bridge: [URL]"
```

---

### 3. Full Attack Chain and Failed Controls

```text
┌─────────────────────────────────────────────────────────────────────────┐
│                            FULL ATTACK CHAIN                            │
├────┬────────────────────────────────┬───────────────────────────────────┤
│ #  │ ATTACKER ACTION                │ CONTROL THAT FAILED               │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 1  │ Compromised a pod on the EKS   │ Application vulnerability (RCE,   │
│    │ worker node i-0abc123def456    │ SSRF, or supply chain) — no WAF   │
│    │                                │ rule caught the initial exploit   │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 2  │ Queried IMDS from pod:         │ IMDSv2 NOT ENFORCED on the node   │
│    │ curl http://169.254.169.254/   │ group. If IMDSv2 with hop limit   │
│    │ latest/meta-data/iam/...       │ =1 was set, pod containers        │
│    │                                │ (2+ hops via veth) couldn't       │
│    │                                │ reach IMDS.                       │
│    │                                │                                   │
│    │                                │ No NetworkPolicy blocking         │
│    │                                │ 169.254.169.254 from pods.        │
│    │                                │                                   │
│    │                                │ IRSA not used — pods were using   │
│    │                                │ node role instead of pod-level    │
│    │                                │ IAM roles.                        │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 3  │ Exfiltrated temporary creds    │ No egress NetworkPolicy — pod     │
│    │ to attacker-controlled infra   │ could connect to any external     │
│    │                                │ IP. No DLP monitoring on          │
│    │                                │ outbound traffic.                 │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 4  │ Used creds from i-0xyz789      │ Node IAM role was OVERLY          │
│    │ to call sts:AssumeRole for     │ PERMISSIVE — a worker node        │
│    │ production account role        │ should NEVER have                 │
│    │                                │ sts:AssumeRole for prod.          │
│    │                                │                                   │
│    │                                │ The cross-account role's trust    │
│    │                                │ policy was too broad — it         │
│    │                                │ allowed the node role without     │
│    │                                │ condition keys restricting to     │
│    │                                │ specific source IPs or VPC.       │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 5  │ Called secretsmanager:         │ Secrets Manager resource policy   │
│    │ GetSecretValue for database    │ did not restrict access to        │
│    │ credentials                    │ specific roles/VPCs. The          │
│    │                                │ assumed prod role had overly      │
│    │                                │ broad secretsmanager:Get*         │
│    │                                │ permissions.                      │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 6  │ Outbound data exfil from       │ No VPC endpoint policies.         │
│    │ i-0xyz789 to external IP       │ No VPC-level egress filtering.    │
│    │ on port 443                    │ Security groups allowed 443       │
│    │                                │ outbound to 0.0.0.0/0.            │
├────┼────────────────────────────────┼───────────────────────────────────┤
│ 7  │ Attack continued for           │ GuardDuty DID detect it, but      │
│    │ duration until GuardDuty       │ no automated response was         │
│    │ alert was investigated         │ configured. No Lambda trigger     │
│    │                                │ to auto-revoke credentials        │
│    │                                │ on HIGH severity findings.        │
└────┴────────────────────────────────┴───────────────────────────────────┘
```

---

### 4. Architecture Changes to Prevent This Attack Vector

**Layer 1: IMDS Hardening**

```hcl
# ENFORCE IMDSv2 with hop limit = 1 on ALL EKS node groups
resource "aws_launch_template" "eks_nodes" {
  name_prefix = "eks-nodes-"

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"      # FORCES IMDSv2
    http_put_response_hop_limit = 1               # Container network = 2 hops
                                                   # So pods CANNOT reach IMDS
    instance_metadata_tags      = "disabled"
  }

  # ... rest of launch template
}

# Additionally, block IMDS at the network level as defense-in-depth
# Calico or Cilium NetworkPolicy:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-imds
  namespace: default  # Apply to all namespaces via Gatekeeper/Kyverno
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 169.254.169.254/32  # Block IMDS
```

**Layer 2: IRSA (IAM Roles for Service Accounts) — Eliminate Node Role Dependency**

```hcl
# Create IRSA for each service that needs AWS access
module "order_svc_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"

  role_name = "order-svc-irsa"

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["orders:order-svc"]
    }
  }

  role_policy_arns = {
    secrets = aws_iam_policy.order_svc_secrets.arn
    # ONLY the specific permissions this service needs
  }
}

# Minimal policy — only the secrets THIS service needs
resource "aws_iam_policy" "order_svc_secrets" {
  name = "order-svc-secrets-readonly"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action =["secretsmanager:GetSecretValue"]
        Resource =[
          "arn:aws:secretsmanager:us-east-1:*:secret:orders/db-creds-*"
        ]
        Condition = {
          StringEquals = {
            "aws:RequestedRegion" = "us-east-1"
          }
        }
      }
    ]
  })
}

# Strip the node role down to BARE MINIMUM
resource "aws_iam_role_policy" "node_role_minimal" {
  name = "eks-node-minimal"
  role = aws_iam_role.eks_node.name
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action =[
          "ec2:DescribeInstances",
          "ec2:DescribeNetworkInterfaces",
          "ecr:GetAuthorizationToken",
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "*"
      }
      # NO sts:AssumeRole
      # NO secretsmanager:*
      # NO s3:*
      # NO iam:*
    ]
  })
}
```

```yaml
# Kubernetes ServiceAccount annotation for IRSA
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-svc
  namespace: orders
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/order-svc-irsa
```

**Layer 3: VPC Segmentation**

```hcl
# Separate VPCs or strict subnet segmentation

# 1. EKS worker nodes should NOT have routes to the management VPC
# 2. Cross-account role assumptions should require VPC endpoint + conditions

# Cross-account role trust policy — restrict to VPC endpoint
resource "aws_iam_role" "prod_role" {
  name = "prod-data-access"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::ACCOUNT:role/order-svc-irsa"  # Only IRSA role, not node role
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "aws:SourceVpc"    = var.eks_vpc_id    # Must come from EKS VPC
            "aws:PrincipalTag/Environment" = "production"
          }
          IpAddress = {
            "aws:SourceIp" = var.eks_nat_gateway_ips  # Must come from known NAT IPs
          }
        }
      }
    ]
  })
}

# VPC Endpoint for STS — keep STS calls within the VPC
resource "aws_vpc_endpoint" "sts" {
  vpc_id              = module.eks_vpc.vpc_id
  service_name        = "com.amazonaws.us-east-1.sts"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = module.eks_vpc.private_subnets
  security_group_ids  =[aws_security_group.vpc_endpoints.id]

  policy = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "sts:AssumeRole"
        Resource  =["arn:aws:iam::PROD_ACCOUNT:role/order-svc-*"]
        # Only allow specific role assumptions through this endpoint
      }
    ]
  })
}

# VPC Endpoint for Secrets Manager
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = module.eks_vpc.vpc_id
  service_name        = "com.amazonaws.us-east-1.secretsmanager"
  vpc_endpoint_type   = "Interface"
  private_dns_enabled = true
  subnet_ids          = module.eks_vpc.private_subnets
  security_group_ids  =[aws_security_group.vpc_endpoints.id]
}
```

**Layer 4: GuardDuty Automated Response**

```hcl
# EventBridge rule to auto-respond to credential exfiltration
resource "aws_cloudwatch_event_rule" "guardduty_cred_exfil" {
  name        = "guardduty-credential-exfiltration-response"
  description = "Auto-respond to credential exfiltration findings"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type =["GuardDuty Finding"]
    detail = {
      type =[
        "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.InsideAWS",
        "UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS"
      ]
      severity = [{ numeric = [">=", 7] }]
    }
  })
}

resource "aws_cloudwatch_event_target" "auto_revoke" {
  rule = aws_cloudwatch_event_rule.guardduty_cred_exfil.name
  arn  = aws_lambda_function.auto_revoke_credentials.arn
}
```

```python
# Lambda function for auto-remediation
import boto3
import json
from datetime import datetime

def handler(event, context):
    finding = event['detail']
    instance_id = finding['resource']['instanceDetails']['instanceId']
    role_name = finding['resource']['accessKeyDetails']['userName']
    
    iam = boto3.client('iam')
    ec2 = boto3.client('ec2')
    sns = boto3.client('sns')
    
    # 1. Revoke all active sessions for the role
    revoke_time = datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%SZ')
    iam.put_role_policy(
        RoleName=role_name,
        PolicyName=f'RevokeOldSessions-{instance_id}',
        PolicyDocument=json.dumps({
            "Version": "2012-10-17",
            "Statement":[{
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*",
                "Condition": {
                    "DateLessThan": {
                        "aws:TokenIssueTime": revoke_time
                    }
                }
            }]
        })
    )
    
    # 2. Isolate the instance - replace SG with quarantine SG
    # (quarantine SG pre-created with no ingress/egress rules)
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[QUARANTINE_SG_ID]
    )
    
    # 3. Create snapshot for forensics
    volumes = ec2.describe_instances(
        InstanceIds=[instance_id]
    )['Reservations'][0]['Instances'][0]['BlockDeviceMappings']
    
    for vol in volumes:
        vol_id = vol['Ebs']['VolumeId']
        ec2.create_snapshot(
            VolumeId=vol_id,
            Description=f'Forensic snapshot - GuardDuty {finding["id"]}',
            TagSpecifications=[{
                'ResourceType': 'snapshot',
                'Tags': [{'Key': 'Incident', 'Value': finding['id']}]
            }]
        )
    
    # 4. Alert security team
    sns.publish(
        TopicArn=SECURITY_SNS_TOPIC,
        Subject=f'🔴 AUTO-RESPONSE: Credential exfiltration from {instance_id}',
        Message=json.dumps({
            'action_taken':[
                f'Revoked all sessions for role {role_name}',
                f'Isolated instance {instance_id} with quarantine SG',
                f'Created forensic snapshots of all volumes'
            ],
            'finding': finding,
            'manual_steps_required':[
                'Investigate CloudTrail for all actions taken with stolen creds',
                'Determine initial compromise vector on the pod/node',
                'Verify no persistence mechanisms were created',
                'Drain and terminate the node after forensics'
            ]
        }, indent=2, default=str)
    )
    
    return {'statusCode': 200, 'action': 'credentials_revoked_instance_isolated'}
```

**Complete Defense Architecture:**

```text
                         BEFORE (Vulnerable)
                         ═══════════════════
Pod → IMDS (v1, no hop limit) → Node IAM Role (overpermissioned)
  → sts:AssumeRole (no conditions) → Prod Account (broad access)
  → secretsmanager:Get* (all secrets) → Exfil (no egress control)

                         AFTER (Hardened)
                         ════════════════
Pod → IMDS BLOCKED (IMDSv2 + hop limit 1 + NetworkPolicy)
Pod → IRSA (pod-specific IAM role via OIDC, minimal permissions)
  → sts:AssumeRole (VPC condition + source IP condition + specific principal)
  → VPC Endpoint (policy-restricted)
  → secretsmanager:Get* (only specific secrets, via VPC endpoint)
  → Egress: NetworkPolicy allowlist only
  → GuardDuty: auto-revoke on detection within SECONDS
```

---

## Q3: NetworkPolicy Debugging 🔥

### 1. WHY `recommendation-svc` Can Reach the Payment Database

**This is the critical Kubernetes NetworkPolicy trap that catches almost everyone.**

The `default-deny-all` policy says:

```yaml
spec:
  podSelector: {}     # Matches ALL pods in namespace
  policyTypes: ["Ingress", "Egress"]
  # NO ingress or egress rules defined
```

This means: "Select all pods. Apply Ingress and Egress policy types. With no rules, deny all traffic."

**BUT THEN** the `search-svc-egress` policy says:

```yaml
spec:
  podSelector:
    matchLabels:
      app: search-svc    # Matches ONLY search-svc pods
  policyTypes: ["Egress"]
  egress:
    - to:
        - ipBlock:
            cidr: 10.0.240.0/22    # Database subnet
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - ipBlock:
            cidr: 10.0.244.0/22    # Redis subnet
      ports:
        - protocol: TCP
          port: 6379
```

So far this looks correct — only `search-svc` should get database and Redis access.

**Here's the trap:** `recommendation-svc` was deployed **without** the label `app: search-svc`, so the `search-svc-egress` policy doesn't select it. The only policy selecting `recommendation-svc` is `default-deny-all` (which selects all pods via `podSelector: {}`).

So `recommendation-svc` should be fully denied, right?

**WRONG.** The answer lies in a subtle but critical detail. Let me look more carefully...

The `default-deny-all` policy selects all pods and declares both Ingress and Egress policy types but specifies no rules — this means all ingress AND egress is denied for ALL pods... **UNLESS another policy provides additional rules for that pod.**

And here's the key: `allow-dns` has:

```yaml
spec:
  podSelector: {}     # Matches ALL pods
  policyTypes:["Egress"]
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

This is correct — all pods get DNS egress. But wait — this doesn't explain how `recommendation-svc` reaches the database.

Let me re-examine the problem. The trap must be more subtle.

**THE ACTUAL ANSWER:**

NetworkPolicies are **additive** (union), not subtractive. When multiple NetworkPolicies select the same pod, the **effective policy is the UNION of all rules from all policies that select that pod.**

Here's what's happening. Consider what policies select `recommendation-svc`:

1. `default-deny-all` — `podSelector: {}` → selects `recommendation-svc` → declares Egress type with no rules → denies all egress
2. `allow-dns` — `podSelector: {}` → selects `recommendation-svc` → adds DNS egress rule

So `recommendation-svc` gets: deny-all + DNS. No database access. **But the scenario says it CAN reach the database.**

Let me reconsider. The most common trap is that `recommendation-svc` happens to have the label `app: search-svc`. But the scenario says it's a new microservice...

**AH — I see it now.** The actual trap:

Check `recommendation-svc`'s labels. If the development team deployed `recommendation-svc` using a template/Helm chart that includes `app: search-svc` as a label (e.g., they copied the search-svc deployment manifest and only changed the container image but forgot to change the label), then `search-svc-egress` would ALSO select `recommendation-svc`.

But that seems like a stretch. Let me think about this differently...

**ACTUALLY — here's the real answer, and it IS a trap:**

Look at the `search-svc-egress` policy again:

```yaml
podSelector:
  matchLabels:
    app: search-svc
```

If `recommendation-svc` was deployed with the label `app: search-svc` (perhaps because it's in the search namespace and the Helm chart defaults the app label to the namespace name, or because the developer copied the manifest), then `search-svc-egress` selects it and grants it database access.

**But there's a more fundamental NetworkPolicy misconception that's the REAL trap:**

The `default-deny-all` says `policyTypes: ["Ingress", "Egress"]` with `podSelector: {}`. This SELECTS all pods for BOTH policy types but provides NO rules. This means: for all pods, ingress is restricted (no ingress rules = no ingress allowed) and egress is restricted (no egress rules = no egress allowed).

The `search-svc-egress` policy ADDS egress rules for pods matching `app: search-svc`. NetworkPolicies are additive — the effective egress for `search-svc` pods is the UNION of: (nothing from default-deny) + (database+redis from search-svc-egress) + (DNS from allow-dns).

For `recommendation-svc` (assuming it does NOT have `app: search-svc`), the effective egress is: (nothing from default-deny) + (DNS from allow-dns) = **DNS only**.

So... if the NetworkPolicies are as stated, `recommendation-svc` SHOULD be blocked.

**The answer must be that `recommendation-svc` has the label `app: search-svc`.** This is the most common real-world scenario — a developer copies a manifest, changes the name but forgets to change all labels.

**OR — the REAL trap might be about the CNI plugin.** If the EKS cluster is using the default AWS VPC CNI without a NetworkPolicy enforcement engine (like Calico or Cilium), then **NetworkPolicy objects exist in the API but are not enforced.** The default AWS VPC CNI does NOT enforce NetworkPolicies. You need the AWS VPC CNI NetworkPolicy controller (available since VPC CNI v1.14+) or a separate Calico/Cilium installation.

**THAT'S the trap.** Let me go with both angles:

**The answer has TWO parts:**

**Part A — The CNI might not enforce NetworkPolicies:**

If the EKS cluster uses the default AWS VPC CNI **without** the NetworkPolicy controller enabled, then ALL NetworkPolicy objects are **decorative** — they exist in etcd but have zero enforcement effect. Every pod can reach every other pod and every external IP. This is the most dangerous misconfiguration because teams believe they have network segmentation but actually have none.

```bash
# Check if NetworkPolicy enforcement is actually active:
kubectl get daemonset -n kube-system aws-node -o json | \
  jq '.spec.template.spec.containers[0].env[] | select(.name=="ENABLE_NETWORK_POLICY")'

# If this returns nothing or "false", NetworkPolicies are NOT ENFORCED

# Also check for Calico/Cilium:
kubectl get pods -n calico-system 2>/dev/null
kubectl get pods -n kube-system -l k8s-app=calico-node 2>/dev/null
kubectl get pods -n kube-system -l app.kubernetes.io/name=cilium 2>/dev/null
```

**Part B — If NetworkPolicies ARE enforced, the label issue:**

If the CNI does enforce NetworkPolicies, then `recommendation-svc` MUST have the label `app: search-svc` (or whatever label makes `search-svc-egress` select it). The developer likely copied the search-svc deployment manifest as a starting template.

```bash
# Verify:
kubectl get pod -n search -l app=search-svc --show-labels
# If recommendation-svc pods appear here, that's the problem

kubectl get deployment recommendation-svc -n search -o jsonpath='{.spec.template.metadata.labels}'
# Check if app: search-svc is present
```

---

### 2. Explaining the Mechanism to the Developer

**"I see default-deny-all, so how can anything reach anything?"**

> "Great question — and this is one of the most counterintuitive things about Kubernetes NetworkPolicies. Let me explain exactly what's happening.
> 
> **First, the possible showstopper:** NetworkPolicy objects only work if your CNI plugin actively enforces them. In EKS with the default VPC CNI, enforcement is OFF by default. Let me check..."

```bash
kubectl get daemonset -n kube-system aws-node -o jsonpath='{.spec.template.spec.initContainers[*].env}' | jq '.[] | select(.name=="ENABLE_NETWORK_POLICY")'
```

> **If enforcement is off:** "There it is. Your NetworkPolicies exist in the Kubernetes API but they're being completely ignored by the network layer. Every pod can talk to everything. It's like having a firewall config file but the firewall service isn't running."
> 
> **If enforcement is on:** "Enforcement is active, so let's look at the actual policy evaluation. Here's how Kubernetes NetworkPolicy works:
> 
> 1. **NetworkPolicies are ADDITIVE.** If three policies select the same pod, the effective rules are the UNION of all three. There is no 'deny' rule — you can't say 'take away access that another policy grants.'
> 
> 2. **The deny effect comes from SELECTION, not from rules.** When at least one NetworkPolicy selects a pod for a given direction (egress/ingress), that direction becomes 'default deny' for that pod. Then, only the explicitly listed rules from ALL selecting policies are allowed.
> 
> 3. **Your `default-deny-all` selects all pods for both ingress and egress with no rules — this establishes the baseline deny.** 
> 
> 4. **But `search-svc-egress` ALSO selects `recommendation-svc` pods** — because recommendation-svc has the label `app: search-svc`. Check:"

```bash
kubectl get pods -n search -l app=search-svc
# Shows BOTH search-svc AND recommendation-svc pods
```

> "So the effective egress for recommendation-svc is: (nothing from default-deny) ∪ (database+redis from search-svc-egress) ∪ (DNS from allow-dns) = database + redis + DNS. The default-deny provides the baseline restriction, but search-svc-egress ADDS the database and Redis access back because it selects recommendation-svc too.
> 
> Think of it like a building security system: `default-deny-all` locks every door. But `search-svc-egress` hands out keycards. The problem is that the keycard was issued based on the label `app: search-svc`, and recommendation-svc is wearing that badge too.
> 
> **The fix is twofold:**
> 1. Fix recommendation-svc's labels so it has its own identity
> 2. Create a dedicated NetworkPolicy for recommendation-svc that grants ONLY the access it needs
> 
> **The deeper fix:** Never rely on a single label for NetworkPolicy selection. Use specific, purpose-built labels like `network-policy-group: search-svc-data-access` that are explicitly managed, not inherited from generic Helm templates."

---

### 3. Corrected NetworkPolicies

**Step 1: Fix the labels on recommendation-svc**

```yaml
# FIRST: Fix the recommendation-svc deployment labels
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-svc
  namespace: search
spec:
  selector:
    matchLabels:
      app: recommendation-svc          # ← FIXED: was incorrectly "search-svc"
  template:
    metadata:
      labels:
        app: recommendation-svc        # ← FIXED
        team: search
    spec:
      containers:
        - name: recommendation-svc
          image: 888888888888.dkr.ecr.us-east-1.amazonaws.com/recommendation-svc:v1.0.0
          ports:
            - containerPort: 8080
```

**Step 2: Keep the existing default-deny and DNS policies unchanged**

```yaml
# These remain as-is — they're correct
# default-deny-all: baseline deny for all pods
# allow-dns: DNS egress for all pods
```

**Step 3: Tighten search-svc-egress to be more explicit**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: search-svc-egress
  namespace: search
spec:
  podSelector:
    matchLabels:
      app: search-svc                # Now only matches actual search-svc pods
  policyTypes:
    - Egress
  egress:
    # PostgreSQL database access
    - to:
        - ipBlock:
            cidr: 10.0.240.0/22
      ports:
        - protocol: TCP
          port: 5432
    # Redis access
    - to:
        - ipBlock:
            cidr: 10.0.244.0/22
      ports:
        - protocol: TCP
          port: 6379
    # Allow communication with recommendation-svc
    - to:
        - podSelector:
            matchLabels:
              app: recommendation-svc
      ports:
        - protocol: TCP
          port: 8080
```

**Step 4: search-svc ingress policy**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: search-svc-ingress
  namespace: search
spec:
  podSelector:
    matchLabels:
      app: search-svc
  policyTypes:
    - Ingress
  ingress:
    # Allow ingress from API gateway / ingress controller
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
          podSelector:
            matchLabels:
              app.kubernetes.io/name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
    # Allow ingress from recommendation-svc (if it needs to call back)
    - from:
        - podSelector:
            matchLabels:
              app: recommendation-svc
      ports:
        - protocol: TCP
          port: 8080
```

**Step 5: recommendation-svc egress policy — ONLY search-svc and Redis**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: recommendation-svc-egress
  namespace: search
spec:
  podSelector:
    matchLabels:
      app: recommendation-svc
  policyTypes:
    - Egress
  egress:
    # ONLY to search-svc on port 8080
    - to:
        - podSelector:
            matchLabels:
              app: search-svc
      ports:
        - protocol: TCP
          port: 8080
    # ONLY to Redis on port 6379
    - to:
        - ipBlock:
            cidr: 10.0.244.0/22
      ports:
        - protocol: TCP
          port: 6379
    # EXPLICITLY: No database access. No ipBlock for 10.0.240.0/22.
```

**Step 6: recommendation-svc ingress policy**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: recommendation-svc-ingress
  namespace: search
spec:
  podSelector:
    matchLabels:
      app: recommendation-svc
  policyTypes:
    - Ingress
  ingress:
    # Only accept traffic from search-svc
    - from:
        - podSelector:
            matchLabels:
              app: search-svc
      ports:
        - protocol: TCP
          port: 8080
```

**Step 7: Verify the fix**

```bash
# Verify label fix
kubectl get pods -n search --show-labels
# recommendation-svc should show app=recommendation-svc, NOT app=search-svc

# Test: recommendation-svc should NOT reach the database
kubectl exec -n search deploy/recommendation-svc -- \
  nc -zv -w 3 10.0.240.10 5432 2>&1
# Expected: connection timed out (BLOCKED)

# Test: recommendation-svc SHOULD reach search-svc
kubectl exec -n search deploy/recommendation-svc -- \
  nc -zv -w 3 search-svc.search.svc.cluster.local 8080 2>&1
# Expected: succeeded (ALLOWED)

# Test: recommendation-svc SHOULD reach Redis
kubectl exec -n search deploy/recommendation-svc -- \
  nc -zv -w 3 10.0.244.10 6379 2>&1
# Expected: succeeded (ALLOWED)

# Test: search-svc SHOULD still reach the database
kubectl exec -n search deploy/search-svc -- \
  nc -zv -w 3 10.0.240.10 5432 2>&1
# Expected: succeeded (ALLOWED)

# Test: recommendation-svc should NOT reach the internet
kubectl exec -n search deploy/recommendation-svc -- \
  nc -zv -w 3 8.8.8.8 443 2>&1
# Expected: connection timed out (BLOCKED)
```

---

### 4. Pre-Production Validation Pipeline

**Layer 1: Static Analysis with `kubectl` Dry-Run and `kubeval`**

```yaml
# .bitbucket-pipelines.yml (or equivalent CI)
pipelines:
  pull-requests:
    '**':
      - step:
          name: NetworkPolicy Static Validation
          script:
            # Validate YAML syntax and schema
            - |
              find k8s/ -name "*.yaml" -o -name "*.yml" | while read f; do
                echo "Validating $f..."
                kubeval --strict --kubernetes-version 1.28.0 "$f"
              done
            
            # Ensure every namespace has a default-deny policy
            - |
              for ns_dir in k8s/namespaces/*/; do
                NS=$(basename $ns_dir)
                if ! grep -r "default-deny" "$ns_dir" | grep -q "NetworkPolicy"; then
                  echo "⛔ ERROR: Namespace $NS missing default-deny NetworkPolicy"
                  exit 1
                fi
              done
```

**Layer 2: Policy Linting with `kube-linter` and Custom Rules**

```yaml
            # kube-linter checks for common NetworkPolicy mistakes
            - |
              kube-linter lint k8s/ --config .kube-linter.yaml
```

```yaml
# .kube-linter.yaml
checks:
  include:
    - "no-read-only-root-fs"
    - "run-as-non-root"
  exclude:[]

customChecks:
  - name: "networkpolicy-label-specificity"
    description: "NetworkPolicies must use specific pod selectors, not empty selectors for egress/ingress rules that grant access"
    template: "verify-label-specificity"
    # Custom check: flag any NetworkPolicy that grants access
    # (non-DNS) using podSelector: {} 
```

**Layer 3: Connectivity Matrix Testing with `netpol-analyzer`**

This is the most powerful tool — it analyzes NetworkPolicies and produces a connectivity matrix showing exactly what can talk to what.

```yaml
          - step:
              name: NetworkPolicy Connectivity Analysis
              script:
                # Use NP-Guard / netpol-analyzer to compute connectivity
                - |
                  pip install nca-networkpolicy-analyzer
                  
                  # Analyze current policies
                  nca --pod_list k8s/ --netpol_list k8s/ \
                    --output_format txt \
                    --output_file /tmp/connectivity-matrix.txt
                  
                  echo "=== Current Connectivity Matrix ==="
                  cat /tmp/connectivity-matrix.txt
                  
                  # Compare against expected connectivity (golden file)
                  nca --pod_list k8s/ --netpol_list k8s/ \
                    --expected_connectivity tests/expected-connectivity.yaml \
                    --output_format txt
                  
                  # If diff detected, fail the build
                  if[ $? -ne 0 ]; then
                    echo "⛔ NetworkPolicy connectivity differs from expected!"
                    echo "Review the changes and update expected-connectivity.yaml if intentional."
                    exit 1
                  fi
```

```yaml
# tests/expected-connectivity.yaml — the "golden" connectivity matrix
# This file explicitly states what SHOULD be able to talk to what
# Any deviation fails the build

expected:
  - from:
      namespace: search
      pod: search-svc
    to:
      - target: 10.0.240.0/22:5432    # PostgreSQL
        allowed: true
      - target: 10.0.244.0/22:6379    # Redis
        allowed: true
      - target: search/recommendation-svc:8080
        allowed: true
  
  - from:
      namespace: search
      pod: recommendation-svc
    to:
      - target: search/search-svc:8080
        allowed: true
      - target: 10.0.244.0/22:6379    # Redis
        allowed: true
      - target: 10.0.240.0/22:5432    # PostgreSQL — MUST BE FALSE
        allowed: false
      - target: 0.0.0.0/0:443         # Internet — MUST BE FALSE
        allowed: false
```

**Layer 4: Runtime Connectivity Testing in Staging**

```yaml
          - step:
              name: Staging NetworkPolicy Integration Test
              deployment: staging
              script:
                # After deploying to staging, run actual connectivity tests
                - |
                  # Test matrix: for each service, verify allowed and denied connections
                  
                  echo "=== Testing recommendation-svc connectivity ==="
                  
                  # SHOULD SUCCEED
                  kubectl exec -n search deploy/recommendation-svc -- \
                    nc -zv -w 5 search-svc.search.svc.cluster.local 8080 2>&1 && \
                    echo "✅ recommendation-svc → search-svc: ALLOWED (expected)" || \
                    { echo "⛔ recommendation-svc → search-svc: BLOCKED (unexpected!)"; exit 1; }
                  
                  kubectl exec -n search deploy/recommendation-svc -- \
                    nc -zv -w 5 redis.search.svc.cluster.local 6379 2>&1 && \
                    echo "✅ recommendation-svc → Redis: ALLOWED (expected)" || \
                    { echo "⛔ recommendation-svc → Redis: BLOCKED (unexpected!)"; exit 1; }
                  
                  # SHOULD FAIL
                  kubectl exec -n search deploy/recommendation-svc -- \
                    nc -zv -w 5 10.0.240.10 5432 2>&1 && \
                    { echo "⛔ recommendation-svc → PostgreSQL: ALLOWED (THIS SHOULD BE BLOCKED!)"; exit 1; } || \
                    echo "✅ recommendation-svc → PostgreSQL: BLOCKED (expected)"
                  
                  kubectl exec -n search deploy/recommendation-svc -- \
                    nc -zv -w 5 payment-db.payments.svc.cluster.local 5432 2>&1 && \
                    { echo "⛔ recommendation-svc → payment-db: ALLOWED (CRITICAL VIOLATION!)"; exit 1; } || \
                    echo "✅ recommendation-svc → payment-db: BLOCKED (expected)"
                  
                  echo "=== All connectivity tests passed ==="
```

**Layer 5: OPA/Gatekeeper Policy for Label Uniqueness**

Prevent the root cause — ensure deployment labels are unique per service:

```yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8suniquelabels
spec:
  crd:
    spec:
      names:
        kind: K8sUniqueLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labelKey:
              type: string
              description: "The label key that must be unique per Deployment"
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8suniquelabels

        violation[{"msg": msg}] {
          input.review.kind.kind == "Deployment"
          label_key := input.parameters.labelKey
          label_value := input.review.object.spec.template.metadata.labels[label_key]
          
          # Check if the deployment name matches the label value
          deployment_name := input.review.object.metadata.name
          label_value != deployment_name
          
          msg := sprintf(
            "Deployment '%s' has label '%s: %s' which doesn't match its name. This can cause NetworkPolicy cross-contamination. Set '%s: %s'.",[deployment_name, label_key, label_value, label_key, deployment_name]
          )
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sUniqueLabels
metadata:
  name: require-matching-app-label
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
  parameters:
    labelKey: "app"
```

**Layer 6: Continuous Runtime Auditing**

```yaml
# CronJob that runs every 6 hours to verify NetworkPolicy compliance
apiVersion: batch/v1
kind: CronJob
metadata:
  name: netpol-audit
  namespace: security
spec:
  schedule: "0 */6 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: netpol-auditor
          containers:
            - name: auditor
              image: novamart/netpol-auditor:latest
              command:
                - /bin/sh
                - -c
                - |
                  # Run connectivity probes from every namespace
                  # Compare against expected matrix
                  # Alert if any unexpected connectivity found
                  
                  python3 /app/audit.py \
                    --expected /config/expected-connectivity.yaml \
                    --slack-webhook $SLACK_WEBHOOK \
                    --pagerduty-key $PD_KEY
              env:
                - name: SLACK_WEBHOOK
                  valueFrom:
                    secretKeyRef:
                      name: monitoring-secrets
                      key: slack-webhook
          restartPolicy: OnFailure
```

**Complete Validation Pipeline Summary:**

```text
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   PR Filed   │──►│   Static     │──►│ Connectivity │──►│   Staging    │──►│  Continuous  │
│              │   │   Lint       │   │ Matrix Diff  │   │  Live Test   │   │  Audit       │
├──────────────┤   ├──────────────┤   ├──────────────┤   ├──────────────┤   ├──────────────┤
│ Developer    │   │ kubeval      │   │ nca-analyzer │   │ nc/curl      │   │ CronJob      │
│ pushes code  │   │ kube-linter  │   │ vs golden    │   │ probes in    │   │ every 6hr    │
│              │   │ OPA conftest │   │ matrix       │   │ staging      │   │ full matrix  │
│              │   │              │   │              │   │ cluster      │   │ probe        │
├──────────────┤   ├──────────────┤   ├──────────────┤   ├──────────────┤   ├──────────────┤
│              │   │ Catches:     │   │ Catches:     │   │ Catches:     │   │ Catches:     │
│              │   │ - YAML errs  │   │ - Unintended │   │ - CNI not    │   │ - Drift      │
│              │   │ - Missing    │   │   access     │   │   enforcing  │   │ - Runtime    │
│              │   │   deny-all   │   │ - Label      │   │ - Real-world │   │   misconfig  │
│              │   │ - Schema     │   │   collision  │   │   behavior   │   │ - New svcs   │
│              │   │   violations │   │ - Overly     │   │   mismatch   │   │   without    │
│              │   │              │   │   broad rules│   │              │   │   policies   │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

---

## Q4: Security Incident — Full Response 🔥

### 1. First 10 Actions — In Exact Order

**This is a worst-case scenario: an attacker has `AdministratorAccess` on a service account with programmatic access. Every second matters.**

**Action 1 (Minute 0): Disable the compromised access key IMMEDIATELY**

```bash
# This is the single highest-priority action — stop the bleeding
aws iam update-access-key \
  --user-name ci-deployer \
  --access-key-id $(aws iam list-access-keys --user-name ci-deployer \
    --query 'AccessKeyMetadata[0].AccessKeyId' --output text) \
  --status Inactive

# If there are multiple access keys, disable ALL of them
for key in $(aws iam list-access-keys --user-name ci-deployer \
  --query 'AccessKeyMetadata[*].AccessKeyId' --output text); do
  aws iam update-access-key \
    --user-name ci-deployer \
    --access-key-id $key \
    --status Inactive
  echo "Disabled key: $key"
done
```

**Action 2 (Minute 1): Revoke any active sessions / temporary credentials**

The attacker attached `AdministratorAccess` and may have called `sts:GetSessionToken` or `sts:AssumeRole` to get temporary credentials that survive access key deactivation.

```bash
# Attach an inline deny-all policy that revokes sessions issued before NOW
REVOKE_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

aws iam put-user-policy \
  --user-name ci-deployer \
  --policy-name "EmergencyRevokeAllAccess" \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement":[
      {
        "Effect": "Deny",
        "Action": "*",
        "Resource": "*"
      }
    ]
  }'
```

```bash
# ALSO: If the attacker assumed roles, revoke those role sessions too
# Check CloudTrail for which roles were assumed
ASSUMED_ROLES=$(aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=ci-deployer \
  --start-time $(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ") \
  --query "Events[?EventName=='AssumeRole'].CloudTrailEvent" --output text | \
  python3 -c "import sys,json; [print(json.loads(l)['requestParameters']['roleArn']) for l in sys.stdin if l.strip()]" | \
  sort -u)

for role_arn in $ASSUMED_ROLES; do
  ROLE_NAME=$(echo $role_arn | awk -F'/' '{print $NF}')
  echo "Revoking sessions for assumed role: $ROLE_NAME"
  aws iam put-role-policy \
    --role-name $ROLE_NAME \
    --policy-name "EmergencyRevokeFromCompromise" \
    --policy-document '{
      "Version": "2012-10-17",
      "Statement":[{
        "Effect": "Deny",
        "Action": "*",
        "Resource": "*",
        "Condition": {
          "DateLessThan": {
            "aws:TokenIssueTime": "'"$REVOKE_TIME"'"
          }
        }
      }]
    }'
done
```

**Action 3 (Minute 2): Remove the AdministratorAccess policy the attacker attached**

```bash
aws iam detach-user-policy \
  --user-name ci-deployer \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Also check for any other policies the attacker might have attached
aws iam list-attached-user-policies --user-name ci-deployer
aws iam list-user-policies --user-name ci-deployer

# Remove any suspicious inline policies
aws iam list-user-policies --user-name ci-deployer --query 'PolicyNames[]' --output text | \
  while read policy; do
    echo "Reviewing inline policy: $policy"
    aws iam get-user-policy --user-name ci-deployer --policy-name "$policy"
  done
```

**Action 4 (Minute 3): Check if the attacker created persistence — new users, keys, roles**

```bash
# CRITICAL: The attacker had AdministratorAccess for 45 min
# They could have created backdoor accounts

# Check for recently created IAM users
aws iam list-users --query 'Users[?CreateDate>=`'"$(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ")"'`]'

# Check for recently created access keys on ANY user
for user in $(aws iam list-users --query 'Users[*].UserName' --output text); do
  aws iam list-access-keys --user-name $user \
    --query "AccessKeyMetadata[?CreateDate>=\`$(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ")\`]" \
    --output table
done

# Check for recently created roles
aws iam list-roles \
  --query "Roles[?CreateDate>=\`$(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ")\`].{Name:RoleName,Created:CreateDate,Arn:Arn}"

# Check for recently created Lambda functions (could be a backdoor)
aws lambda list-functions --query "Functions[?LastModified>=\`$(date -d '2 hours ago' -u +"%Y-%m-%dT")\`].{Name:FunctionName,Modified:LastModified}"
```

**Action 5 (Minute 5): Quarantine the source IP at the network level**

```bash
# The API calls come from an unknown IP — block it everywhere
ATTACKER_IP="203.0.113.99"  # From VPC Flow Logs / CloudTrail

# Block at WAF level
aws wafv2 update-ip-set \
  --name "blocked-ips" \
  --scope REGIONAL \
  --id $BLOCKED_IP_SET_ID \
  --addresses "$ATTACKER_IP/32" \
  --lock-token $(aws wafv2 get-ip-set --name blocked-ips --scope REGIONAL --id $BLOCKED_IP_SET_ID --query LockToken --output text)

# Block at NACLs for relevant VPCs
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 50 \
  --protocol "-1" \
  --rule-action deny \
  --ingress \
  --cidr-block "$ATTACKER_IP/32"
```

**Action 6 (Minute 6): Pull the FULL CloudTrail audit for the attacker's session**

```bash
# Every single API call made by ci-deployer in the last 2 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=ci-deployer \
  --start-time $(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ") \
  --max-results 200 \
  --output json > /tmp/ci-deployer-full-audit.json

# Summarize what was done
cat /tmp/ci-deployer-full-audit.json | jq -r '
  .Events[] | 
  [.EventTime, .EventName, .EventSource] | @tsv' | sort

# Count by event type
cat /tmp/ci-deployer-full-audit.json | jq -r '.Events[].EventName' | sort | uniq -c | sort -rn
```

**Action 7 (Minute 8): Assess the 8 secrets that were accessed**

```bash
# Identify exactly which secrets were read
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventName == "GetSecretValue") |
   .CloudTrailEvent | fromjson |
   {
     secret: .requestParameters.secretId,
     time: .eventTime,
     sourceIP: .sourceIPAddress
   }]'

# List all secrets that need rotation
echo "SECRETS REQUIRING IMMEDIATE ROTATION:"
echo "1. payments-db-master-creds"
echo "2. orders-db-master-creds"
echo "3. users-db-master-creds"
echo "4. stripe-api-key"
echo "5. sendgrid-api-key"
echo "6. twilio-api-key"
echo "7. [identify remaining 2]"
echo "8. [identify remaining 2]"
```

**Action 8 (Minute 9): Check for data exfiltration indicators**

```bash
# Check if the attacker accessed S3 buckets
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | select(.EventSource == "s3.amazonaws.com")]'

# Check for EC2 instances launched (potential for data staging)
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | select(.EventName == "RunInstances")]'

# Check for snapshots created (data exfil via snapshot sharing)
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | select(.EventName | test("Snapshot|Share|Copy"))]'

# Check VPC Flow Logs for large data transfers from the attacker IP
aws logs start-query \
  --log-group-name "VPCFlowLogs" \
  --start-time $(date -d '2 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, srcAddr, dstAddr, bytes
    | filter srcAddr = "'"$ATTACKER_IP"'" or dstAddr = "'"$ATTACKER_IP"'"
    | stats sum(bytes) as total_bytes by srcAddr, dstAddr
    | sort total_bytes desc
  '
```

**Action 9 (Minute 10): Disable the ci-deployer console login if one exists**

```bash
# Check if the attacker created a console login for ci-deployer
aws iam get-login-profile --user-name ci-deployer 2>/dev/null && \
  aws iam delete-login-profile --user-name ci-deployer && \
  echo "Console login deleted" || \
  echo "No console login exists"

# Check for MFA devices the attacker might have registered
aws iam list-mfa-devices --user-name ci-deployer
```

**Action 10 (Minute 10): Alert and activate incident response team**

```bash
# Post to #security-incidents
# "🔴 ACTIVE COMPROMISE — ci-deployer service account
# - Access keys DISABLED
# - AdministratorAccess REMOVED  
# - Active sessions REVOKED
# - 8 secrets were accessed and need IMMEDIATE rotation
# - Attacker IP blocked at WAF and NACL
# - Checking for persistence mechanisms (backdoor users/roles)
# - BRIDGE CALL: [URL] — ALL security team join NOW
# - DO NOT redeploy anything via Jenkins until further notice"

# Page the CISO
# Page the database team (for credential rotation)
# Page the payments team (Stripe key compromised)
```

---

### 2. Full Blast Radius Determination

The attacker had `AdministratorAccess` for 45 minutes. This is equivalent to root access to the entire AWS account. The blast radius assessment must be exhaustive.

**Category 1: IAM Persistence**

```bash
# All IAM changes in the last 2 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventSource,AttributeValue=iam.amazonaws.com \
  --start-time $(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ") \
  --output json > /tmp/iam-events.json

# Look for:
cat /tmp/iam-events.json | jq '[.Events[].EventName]' | sort | uniq -c | sort -rn

# Specific persistence checks:
# New users
# New roles  
# New access keys on existing users
# Modified trust policies on existing roles (adding external account)
# New SAML/OIDC providers (federated access backdoor)
# Modified password policies
# Deactivated MFA

aws iam list-saml-providers
aws iam list-open-id-connect-providers

# Check for modified role trust policies — this is a subtle backdoor
for role in $(aws iam list-roles --query 'Roles[*].RoleName' --output text); do
  TRUST=$(aws iam get-role --role-name $role --query 'Role.AssumeRolePolicyDocument' --output json)
  # Flag any trust policy that references an external account ID
  echo "$TRUST" | grep -v "$(aws sts get-caller-identity --query Account --output text)" | \
    grep -q "arn:aws" && echo "⚠️ EXTERNAL TRUST: $role — $TRUST"
done
```

**Category 2: Compute Persistence**

```bash
# New EC2 instances
aws ec2 describe-instances \
  --filters "Name=launch-time,Values=$(date -d '2 hours ago' -u +"%Y-%m-%dT")*" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,Type:InstanceType,State:State.Name,IP:PublicIpAddress,LaunchTime:LaunchTime}'

# New Lambda functions or modifications
aws lambda list-functions \
  --query "Functions[?LastModified>=\`$(date -d '2 hours ago' -u +"%Y-%m-%dT")\`].{Name:FunctionName,Modified:LastModified}"

# Check for Lambda function URL or API Gateway endpoints (backdoor API)
for func in $(aws lambda list-functions --query 'Functions[*].FunctionName' --output text); do
  aws lambda get-function-url-config --function-name $func 2>/dev/null && \
    echo "⚠️ Function URL exists: $func"
done

# New ECS tasks / Fargate tasks
aws ecs list-clusters --query 'clusterArns[]' --output text | while read cluster; do
  aws ecs list-tasks --cluster $cluster --query 'taskArns[]'
done

# Check for modified user data on existing instances (time bomb)
aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --output text | \
  while read iid; do
    aws ec2 describe-instance-attribute --instance-id $iid --attribute userData 2>/dev/null
  done
```

**Category 3: Data Access**

```bash
# S3 bucket access
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventSource == "s3.amazonaws.com") |
   .CloudTrailEvent | fromjson |
   {event: .eventName, bucket: .requestParameters.bucketName, key: .requestParameters.key}]'

# Secrets Manager access (already identified the 8)
# But check for LIST operations too — attacker now knows all secret names
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventSource == "secretsmanager.amazonaws.com") |
   .CloudTrailEvent | fromjson |
   {event: .eventName, secret: .requestParameters.secretId}]'

# RDS snapshots (data exfil via snapshot)
aws rds describe-db-snapshots \
  --query "DBSnapshots[?SnapshotCreateTime>=\`$(date -d '2 hours ago' -u +"%Y-%m-%dT%H:%M:%SZ")\`]"

# Check for shared snapshots — attacker could share to their own account
aws rds describe-db-snapshot-attributes --db-snapshot-identifier <each-snapshot>

# DynamoDB access
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | select(.EventSource == "dynamodb.amazonaws.com")]'

# SSM Parameter Store
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | select(.EventSource == "ssm.amazonaws.com")]'
```

**Category 4: Network Changes**

```bash
# Security group modifications
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventName | test("SecurityGroup|Authorize|Revoke"))]'

# VPC changes (new VPC peering, new endpoints, route table changes)
cat /tmp/ci-deployer-full-audit.json | jq '[.Events[] | 
   select(.EventName | test("VPC|Peering|Route|Endpoint|Gateway"))]'

# Check for transit gateway attachments or VPN connections
aws ec2 describe-transit-gateway-attachments \
  --query "TransitGatewayAttachments[?CreationTime>=\`$(date -d '2 hours ago' -u +"%Y-%m-%dT")\`]"
```

**Category 5: CloudTrail and Monitoring Tampering**

```bash
# Did the attacker disable CloudTrail to cover tracks?
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventName | test("StopLogging|DeleteTrail|PutEventSelectors|UpdateTrail"))]'

# Did they disable GuardDuty?
cat /tmp/ci-deployer-full-audit.json | jq '[.Events[] | 
   select(.EventSource == "guardduty.amazonaws.com")]'

# Did they modify CloudWatch alarms?
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventName | test("DeleteAlarm|DisableAlarm|PutMetricAlarm"))]'

# Check Config rules
cat /tmp/ci-deployer-full-audit.json | jq '
  [.Events[] | 
   select(.EventSource == "config.amazonaws.com")]'
```

**Comprehensive Blast Radius Report:**

```text
╔═══════════════════════════════════════════════════════════════╗
║                    BLAST RADIUS ASSESSMENT                    ║
╠═══════════╦═══════════════════════════════════╦═══════════════╣
║ CATEGORY  ║ FINDINGS                          ║ SEVERITY      ║
╠═══════════╬═══════════════════════════════════╬═══════════════╣
║ Secrets   ║ 8 secrets read (3 DB, 3 API,      ║ CRITICAL      ║
║           ║ 2 TBD) — MUST ROTATE ALL          ║               ║
╠═══════════╬═══════════════════════════════════╬═══════════════╣
║ IAM       ║[list any backdoor users/roles]   ║ CRITICAL      ║
╠═══════════╬═══════════════════════════════════╬═══════════════╣
║ Data      ║[list any S3/RDS access]          ║ HIGH          ║
╠═══════════╬═══════════════════════════════════╬═══════════════╣
║ Compute   ║ [list any new instances/functions]║ HIGH          ║
╠═══════════╬═══════════════════════════════════╬═══════════════╣
║ Network   ║ [list any SG/VPC changes]         ║ MEDIUM        ║
╠═══════════╬═══════════════════════════════════╬═══════════════╣
║ Logging   ║ [any tampering with CloudTrail]   ║ CRITICAL      ║
╚═══════════╩═══════════════════════════════════╩═══════════════╝
```

---

### 3. Secret Rotation Plan

**Priority order matters — rotate based on blast radius and exposure risk:**

```text
TIMELINE FOR SECRET ROTATION
═══════════════════════════════════════════════════════════════
                                                               
IMMEDIATE (Parallel — Hour 0-1):                               
  ├── [TRACK 1] Stripe API Key                                 
  ├── [TRACK 2] SendGrid API Key                               
  └──[TRACK 3] Twilio API Key                                 
                                                               
HOUR 1-2 (Sequential — dependencies):                          
  ├── [TRACK 4] Payments DB master creds                       
  ├── [TRACK 5] Orders DB master creds                         
  └── [TRACK 6] Users DB master creds                          
                                                               
HOUR 2-3 (Verification):                                       
  └── Verify all services healthy with new credentials         
═══════════════════════════════════════════════════════════════
```

**TRACK 1: Stripe API Key (HIGHEST PRIORITY — direct financial risk)**

```bash
# Why first: Attacker could charge cards, create refunds, exfiltrate PII
# Time to weaponize: MINUTES

# Step 1: Rotate in Stripe dashboard IMMEDIATELY
# Go to https://dashboard.stripe.com/apikeys → Roll key
# This instantly invalidates the old key

# Step 2: Update in Secrets Manager
aws secretsmanager update-secret \
  --secret-id prod/stripe-api-key \
  --secret-string '{"api_key":"sk_live_NEW_KEY_HERE"}'

# Step 3: Restart payment-svc pods to pick up new secret
# If using External Secrets Operator or CSI driver with rotation:
kubectl rollout restart deployment/payment-svc -n payments

# Step 4: Verify
kubectl logs -n payments -l app=payment-svc --since=5m | grep -i "stripe"
# Should see successful Stripe API connections

# Step 5: Check Stripe dashboard for any unauthorized activity
# Review: recent charges, refunds, customer data exports, webhook changes
```

**TRACK 2: SendGrid API Key (Parallel with Track 1)**

```bash
# Why urgent: Attacker could send phishing emails FROM NovaMart's domain
# Reputational damage + customer trust

# Step 1: Revoke in SendGrid dashboard → Settings → API Keys → Delete old key
# Create new key with same permissions

# Step 2: Update in Secrets Manager
aws secretsmanager update-secret \
  --secret-id prod/sendgrid-api-key \
  --secret-string '{"api_key":"SG.NEW_KEY_HERE"}'

# Step 3: Restart notification service
kubectl rollout restart deployment/notification-svc -n notifications

# Step 4: Send a test email to verify
# Step 5: Check SendGrid activity for unauthorized sends
```

**TRACK 3: Twilio API Key (Parallel with Tracks 1 & 2)**

```bash
# Similar pattern — rotate in Twilio console, update Secrets Manager, restart pods
# Check Twilio logs for unauthorized SMS/calls

aws secretsmanager update-secret \
  --secret-id prod/twilio-api-key \
  --secret-string '{"account_sid":"AC...","auth_token":"NEW_TOKEN"}'

kubectl rollout restart deployment/notification-svc -n notifications
```

**TRACKS 4-6: Database Credentials (SEQUENTIAL — high-risk operations)**

These are sequential because database credential rotation is more complex and risky — a mistake can take down the entire platform.

```bash
# PAYMENTS DB (Track 4 — highest financial risk)
# ═══════════════════════════════════════════

# Step 1: Create a NEW master user (don't just change the password —
# the attacker might have created additional DB users)

# Connect to RDS and audit existing users first
PGPASSWORD=$OLD_PASS psql -h payments-db.cluster-xyz.us-east-1.rds.amazonaws.com \
  -U master -d payments -c "
    SELECT usename, usecreatedb, usesuper, valuntil 
    FROM pg_user ORDER BY usecreatedb DESC;"

# Check for any recently created users or grants
PGPASSWORD=$OLD_PASS psql -h payments-db.cluster-xyz.us-east-1.rds.amazonaws.com \
  -U master -d payments -c "
    SELECT grantor, grantee, table_schema, privilege_type 
    FROM information_schema.role_table_grants 
    WHERE grantee NOT IN ('master', 'payment_svc_user', 'rds_superuser');"

# Step 2: Change master password
aws rds modify-db-cluster \
  --db-cluster-identifier payments-db-cluster \
  --master-user-password "$(openssl rand -base64 32)" \
  --apply-immediately

# Step 3: Change application user password
PGPASSWORD=$OLD_PASS psql -h payments-db.cluster-xyz.us-east-1.rds.amazonaws.com \
  -U master -d payments -c "
    ALTER USER payment_svc_user WITH PASSWORD '$(openssl rand -base64 32)';"

# Step 4: Remove any suspicious DB users the attacker may have created
PGPASSWORD=$NEW_PASS psql -h payments-db.cluster-xyz.us-east-1.rds.amazonaws.com \
  -U master -d payments -c "
    -- Revoke and drop any unknown users
    -- REVOKE ALL ON ALL TABLES IN SCHEMA public FROM suspicious_user;
    -- DROP USER suspicious_user;"

# Step 5: Update Secrets Manager with new credentials
aws secretsmanager update-secret \
  --secret-id prod/payments-db-master-creds \
  --secret-string '{"username":"master","password":"NEW_MASTER_PASS","host":"payments-db.cluster-xyz.us-east-1.rds.amazonaws.com","port":5432,"dbname":"payments"}'

# Step 6: Restart payment-svc to pick up new DB creds
kubectl rollout restart deployment/payment-svc -n payments

# Step 7: VERIFY — watch for connection errors
kubectl logs -n payments -l app=payment-svc --since=2m | grep -iE "error|connect|refused|auth"

# Step 8: Verify transactions are processing
kubectl exec -n payments deploy/payment-svc -- curl -s localhost:8080/health
```

```bash
# ORDERS DB (Track 5) — same pattern
aws rds modify-db-cluster \
  --db-cluster-identifier orders-db-cluster \
  --master-user-password "$(openssl rand -base64 32)" \
  --apply-immediately

# ... (same steps as payments DB)

kubectl rollout restart deployment/order-svc -n orders
```

```bash
# USERS DB (Track 6) — same pattern but EXTRA CRITICAL
# The users DB contains PII — check for data exfiltration indicators

# Before rotating, check for recent bulk queries
PGPASSWORD=$OLD_PASS psql -h users-db.cluster-xyz.us-east-1.rds.amazonaws.com \
  -U master -d users -c "
    SELECT query, calls, total_exec_time, rows
    FROM pg_stat_statements 
    ORDER BY rows DESC 
    LIMIT 20;"
# Look for: SELECT * FROM users (full table dump)

aws rds modify-db-cluster \
  --db-cluster-identifier users-db-cluster \
  --master-user-password "$(openssl rand -base64 32)" \
  --apply-immediately

kubectl rollout restart deployment/user-svc -n users
```

**Coordination and Communication Plan:**

```text
ROTATION COMMUNICATION
════════════════════════════════════════════════════════════
WHO                     WHEN                 WHAT
────────────────────────┬────────────────────┬─────────────────────────────────────────────────────
#security-incidents     │ T+0 min            │ "Secret rotation starting"
Payment team lead       │ T+0 min            │ "Stripe key rotating NOW"
NOC / On-call           │ T+0 min            │ "Expect brief service disruptions"
Database team           │ T+30 min           │ "DB cred rotation starting"
#engineering-all        │ T+45 min           │ "DB connections may blip"
Stripe support          │ T+60 min           │ "Report unauthorized activity"
CISO                    │ T+120 min          │ "All 8 secrets rotated"
Legal                   │ T+120 min          │ "PII DB was accessed — assess notification reqs"
════════════════════════╧════════════════════╧═════════════════════════════════════════════════════
```

---

### 4. Systemic Fixes — Every Link in the Chain

The attack chain was: **Public S3 bucket → Jenkins backup → ci-deployer access key → overpermissive IAM → AdminAccess escalation → secrets exfil**

Every link must be fixed:

**Link 1: S3 Bucket with Public Read Access**

```hcl
# PREVENT public S3 buckets at the account level
resource "aws_s3_account_public_access_block" "account" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# For EVERY bucket — explicit deny of public access
resource "aws_s3_bucket_public_access_block" "all_buckets" {
  for_each = toset(var.all_bucket_names)
  bucket   = each.value

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# SCPs to prevent anyone from making buckets public
resource "aws_organizations_policy" "deny_public_s3" {
  name    = "deny-public-s3"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Sid       = "DenyPublicS3"
        Effect    = "Deny"
        Action    =[
          "s3:PutBucketPolicy",
          "s3:PutBucketAcl",
          "s3:PutObjectAcl"
        ]
        Resource  = "*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-acl" = "private"
          }
        }
      }
    ]
  })
}

# AWS Config rule to detect and auto-remediate
resource "aws_config_config_rule" "s3_public" {
  name = "s3-bucket-public-read-prohibited"
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
}

resource "aws_config_remediation_configuration" "s3_public" {
  config_rule_name = aws_config_config_rule.s3_public.name
  target_type      = "SSM_DOCUMENT"
  target_id        = "AWS-DisableS3BucketPublicReadWrite"
  automatic        = true
  maximum_automatic_attempts = 3
  retry_attempt_seconds      = 60
}
```

**Link 2: Jenkins Credential Store Backed Up to S3**

```bash
# Credentials should NEVER be in backups
# Fix 1: Jenkins should use external secret management, not its own credential store

# Fix 2: If Jenkins must store credentials, encrypt backups with KMS
# and NEVER back up credential files to S3

# Fix 3: Move to OIDC-based authentication — no long-lived access keys at all
```

```hcl
# Replace ci-deployer ACCESS KEYS with OIDC federation
# Jenkins/Bitbucket Pipelines can assume IAM roles via OIDC
# This eliminates long-lived credentials entirely

resource "aws_iam_openid_connect_provider" "bitbucket" {
  url = "https://api.bitbucket.org/2.0/workspaces/novamart/pipelines-config/identity/oidc"
  
  client_id_list =[
    "ari:cloud:bitbucket::workspace/novamart-workspace-uuid"
  ]
  
  thumbprint_list = ["a]"]  # Bitbucket's OIDC thumbprint
}

resource "aws_iam_role" "ci_deployer_oidc" {
  name = "ci-deployer-oidc"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.bitbucket.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "api.bitbucket.org:sub" = "novamart-workspace-uuid:novamart-repo-uuid:*"
          }
          # ONLY from specific repository, specific branch
          StringLike = {
            "api.bitbucket.org:branch" = "main"
          }
        }
      }
    ]
  })
}

# DELETE all long-lived access keys for ci-deployer
# After OIDC is working, the IAM user should be deleted entirely
```

**Link 3: Overpermissive IAM Policies on ci-deployer**

```hcl
# ci-deployer had iam:Attach* and iam:Put* — NEVER NEEDED
# Principle of least privilege: a CI/CD deployer needs to deploy, not manage IAM

resource "aws_iam_policy" "ci_deployer_minimal" {
  name = "ci-deployer-minimal"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Sid    = "ECRPushPull"
        Effect = "Allow"
        Action =[
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:BatchGetImage",
          "ecr:GetDownloadUrlForLayer"
        ]
        Resource = "arn:aws:ecr:us-east-1:*:repository/novamart/*"
      },
      {
        Sid    = "EKSDeploy"
        Effect = "Allow"
        Action =[
          "eks:DescribeCluster"
        ]
        Resource = "arn:aws:eks:us-east-1:*:cluster/novamart-*"
      },
      {
        Sid    = "ReadSecrets"
        Effect = "Allow"
        Action =[
          "secretsmanager:GetSecretValue"
        ]
        Resource =[
          "arn:aws:secretsmanager:us-east-1:*:secret:ci/*"
        ]
        # ONLY ci-specific secrets, NOT prod database creds
      }
      # NO iam:* permissions
      # NO ec2:* permissions
      # NO s3:* permissions (beyond ECR)
    ]
  })
}

# Permissions boundary — even if someone attaches AdministratorAccess,
# the boundary limits what's actually allowed
resource "aws_iam_policy" "ci_deployer_boundary" {
  name = "ci-deployer-permissions-boundary"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Effect = "Allow"
        Action =[
          "ecr:*",
          "eks:DescribeCluster",
          "secretsmanager:GetSecretValue",
          "sts:GetCallerIdentity"
        ]
        Resource = "*"
      },
      {
        Effect   = "Deny"
        Action   =[
          "iam:*",
          "organizations:*",
          "cloudtrail:*",
          "guardduty:*",
          "config:*"
        ]
        Resource = "*"
        # EVEN IF AdministratorAccess is attached, these are STILL DENIED
      }
    ]
  })
}

resource "aws_iam_user" "ci_deployer" {
  name                 = "ci-deployer"
  permissions_boundary = aws_iam_policy.ci_deployer_boundary.arn
}
```

**Link 4: SCP — Account-Level Guardrails**

```hcl
# SCP applied at the OU level — prevents privilege escalation
resource "aws_organizations_policy" "deny_privilege_escalation" {
  name    = "deny-iam-privilege-escalation"
  type    = "SERVICE_CONTROL_POLICY"
  content = jsonencode({
    Version = "2012-10-17"
    Statement =[
      {
        Sid    = "DenyIAMPrivilegeEscalation"
        Effect = "Deny"
        Action =[
          "iam:AttachUserPolicy",
          "iam:AttachRolePolicy",
          "iam:PutUserPolicy",
          "iam:PutRolePolicy",
          "iam:CreatePolicyVersion",
          "iam:SetDefaultPolicyVersion"
        ]
        Resource = "*"
        Condition = {
          StringNotLike = {
            "aws:PrincipalArn" =[
              "arn:aws:iam::*:role/OrganizationAdminRole",
              "arn:aws:iam::*:role/SecurityTeamRole"
            ]
          }
        }
      },
      {
        Sid    = "DenyCloudTrailDisable"
        Effect = "Deny"
        Action =[
          "cloudtrail:StopLogging",
          "cloudtrail:DeleteTrail",
          "guardduty:DeleteDetector",
          "guardduty:DisassociateFromMasterAccount"
        ]
        Resource = "*"
      }
    ]
  })
}
```

**Link 5: GuardDuty Automated Response**

```hcl
# Same pattern as Q2 — auto-respond to anomalous IAM behavior
resource "aws_cloudwatch_event_rule" "guardduty_iam_anomaly" {
  name = "guardduty-iam-anomaly-response"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type =["GuardDuty Finding"]
    detail = {
      type =[
        { prefix = "Persistence:IAMUser/" },
        { prefix = "PrivilegeEscalation:" },
        { prefix = "UnauthorizedAccess:IAMUser/" }
      ]
      severity = [{ numeric =[">=", 7] }]
    }
  })
}

resource "aws_cloudwatch_event_target" "auto_disable_key" {
  rule = aws_cloudwatch_event_rule.guardduty_iam_anomaly.name
  arn  = aws_lambda_function.auto_disable_iam_key.arn
}
```

```python
# Lambda: Auto-disable access keys on HIGH/CRITICAL IAM findings
import boto3
import json

def handler(event, context):
    finding = event['detail']
    iam = boto3.client('iam')
    
    # Extract the access key and user from the finding
    access_key_id = finding['resource']['accessKeyDetails']['accessKeyId']
    user_name = finding['resource']['accessKeyDetails']['userName']
    
    # 1. Immediately disable the access key
    iam.update_access_key(
        UserName=user_name,
        AccessKeyId=access_key_id,
        Status='Inactive'
    )
    
    # 2. Attach deny-all inline policy
    iam.put_user_policy(
        UserName=user_name,
        PolicyName='GuardDuty-AutoRemediation-DenyAll',
        PolicyDocument=json.dumps({
            "Version": "2012-10-17",
            "Statement":[{
                "Effect": "Deny",
                "Action": "*",
                "Resource": "*"
            }]
        })
    )
    
    # 3. Alert security team
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=SECURITY_TOPIC,
        Subject=f'🔴 AUTO-REMEDIATION: {user_name} access key disabled',
        Message=json.dumps({
            'user': user_name,
            'access_key': access_key_id,
            'finding_type': finding['type'],
            'severity': finding['severity'],
            'action_taken': 'Key disabled + deny-all policy attached',
            'next_steps': 'Investigate CloudTrail, assess blast radius, rotate secrets'
        }, indent=2)
    )
    
    return {'statusCode': 200}
```

**Link 6: Comprehensive Monitoring for IAM Anomalies**

```hcl
# CloudWatch alarm for IAM policy attachments outside business hours
resource "aws_cloudwatch_log_metric_filter" "iam_policy_changes" {
  name           = "iam-policy-changes"
  pattern        = "{ ($.eventName = AttachUserPolicy) || ($.eventName = AttachRolePolicy) || ($.eventName = PutUserPolicy) || ($.eventName = PutRolePolicy) }"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name

  metric_transformation {
    name          = "IAMPolicyChanges"
    namespace     = "SecurityMetrics"
    value         = "1"
    default_value = "0"
  }
}

resource "aws_cloudwatch_metric_alarm" "iam_policy_changes" {
  alarm_name          = "iam-policy-changes-detected"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "IAMPolicyChanges"
  namespace           = "SecurityMetrics"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "IAM policy attachment detected — investigate immediately"
  alarm_actions       =[aws_sns_topic.security_alerts.arn]
}
```

**Complete Systemic Fix Summary:**

```text
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 ATTACK CHAIN vs. SYSTEMIC FIXES                                 │
├──────────────────────────────────┬──────────────────────────────────────────────────────────────┤
│ CHAIN LINK                       │ FIX                                                          │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ S3 bucket with public access     │ → Account-level public access block                          │
│                                  │ + SCP preventing public S3                                   │
│                                  │ + Config auto-remediation                                    │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ Jenkins backup contained creds   │ → Eliminate long-lived keys (use OIDC)                       │
│                                  │ + Never back up credential stores                            │
│                                  │ + Encrypt all backups with KMS                               │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ ci-deployer had iam:Attach*      │ → Least-privilege policy                                     │
│                                  │ + Permissions boundary                                       │
│                                  │ + SCP blocking iam:Attach* for non-admin roles               │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ No detection of privilege        │ → GuardDuty auto-response Lambda                             │
│ escalation                       │ + CloudWatch alarm on IAM changes                            │
│                                  │ + Anomaly detection for off-hours                            │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ Secrets Manager too broad access │ → Per-service secret access policies                         │
│                                  │ + VPC endpoint policies for SM                               │
│                                  │ + Secret resource policies                                   │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ No automated containment         │ → EventBridge → Lambda auto-disable                          │
│                                  │ + <60 second response time vs. manual investigation          │
├──────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ Single account blast radius      │ → Multi-account strategy (prod secrets in separate account)  │
│                                  │ + Cross-account role conditions                              │
└──────────────────────────────────┴──────────────────────────────────────────────────────────────┘
```

The fundamental lesson: **defense in depth means that NO SINGLE compromise gives an attacker escalation to AdministratorAccess.** The combination of least-privilege IAM, permissions boundaries, SCPs, and automated response ensures that even if credentials are stolen, the attacker's actions are limited and detected within seconds, not minutes.
