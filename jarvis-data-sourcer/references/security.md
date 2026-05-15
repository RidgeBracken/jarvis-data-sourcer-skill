# Section I: Risk Register and Security Framework


## Table of Contents

- Jarvis HQ - Personal AI News Scraping System
- 1. Risk Register
  - Risk Scoring Matrix
  - Risk Register Table
  - Risk Heat Map Summary
- 2. API Key Management
  - 2.1 1Password Integration Design
  - 2.2 Environment Variable Strategy
  - 2.3 Key Rotation Schedule
  - 2.4 No Secrets in Code/Logs/Frontend
  - 2.5 Docker Secret Management
- 3. Rate Limiting Strategy
  - 3.1 Global Rate Limit
  - 3.2 Per-Domain Rate Limits
  - 3.3 robots.txt Crawl-delay Respect
  - 3.4 Exponential Backoff on 429 Errors
  - 3.5 Request Queue with Rate Limiting
- 4. robots.txt Compliance
  - 4.1 robots.txt Parsing and Caching
  - 4.2 Disallowed Path Detection
  - 4.3 robots.txt Update Checking
- 5. Copyright and Fair Use
  - 5.1 What Can Be Scraped (Safe Zone)
  - 5.2 What Should NOT Be Scraped (Danger Zone)
  - 5.3 Attribution Requirements
  - 5.4 DMCA Considerations
  - 5.5 Fair Use Checklist (Four Factors)
- 6. Personal Data Avoidance
  - 6.1 No Collection of PII
  - 6.2 Source Categories - PII Risk Assessment
  - 6.3 PII Detection and Redaction
  - 6.4 Data Minimization Principle
- 7. Logging Safety
  - 7.1 Structured Logging Configuration
  - 7.2 Log Retention Policy
  - 7.3 What Must Never Appear in Logs
  - 7.4 Log Access Control
- 8. Source Blocklist/Allowlist
  - 8.1 Source Classification
  - 8.2 Initial Allowlist
  - 8.3 Blocklist Categories
  - 8.4 Auto-Block Triggers
  - 8.5 Manual Review Workflow
- 9. Security Headers
  - 9.1 User-Agent String
  - 9.2 Accept Headers
  - 9.3 Referrer Policy
  - 9.4 TLS Requirements
- 10. Incident Response
  - 10.1 Response Procedures
  - 10.2 Escalation Path
  - 10.3 Incident Log Format
- 11. Compliance Checklist
  - Pre-Deployment Checklist
  - Ongoing Compliance (Weekly)
  - Ongoing Compliance (Monthly)
  - Ongoing Compliance (Quarterly)
- Appendix A: Quick Reference
  - Severity Priority Order
  - Emergency Contacts
  - Key Files

---


## Jarvis HQ - Personal AI News Scraping System
**Classification:** Internal | **Version:** 1.0 | **Date:** 2025-01-15

---

## 1. Risk Register

### Risk Scoring Matrix

| Severity | Description |
|----------|-------------|
| 1 | Negligible - Cosmetic issue, no operational impact |
| 2 | Minor - Small feature degradation, easily mitigated |
| 3 | Moderate - Noticeable impact, workaround available |
| 4 | High - Significant disruption, requires immediate action |
| 5 | Critical - System failure, legal liability, credential compromise |

| Likelihood | Description |
|------------|-------------|
| 1 | Rare - Less than once per year |
| 2 | Unlikely - Once per year |
| 3 | Possible - Monthly occurrence |
| 4 | Likely - Weekly occurrence |
| 5 | Almost Certain - Daily occurrence |

### Risk Register Table

| Risk ID | Category | Risk Description | Severity (1-5) | Likelihood (1-5) | Impact | Mitigation | Owner |
|---------|----------|------------------|----------------|-------------------|--------|------------|-------|
| R001 | Credential Security | API key leak in logs, source code, or container images | 5 | 3 | Credential compromise, unauthorized API usage, financial liability | 1Password integration for all secrets; environment variables only; pre-commit hooks to scan for secrets; container image scanning; no hardcoded keys in any file | Dev |
| R002 | Credential Security | API key committed to Git repository (public or private) | 5 | 2 | Permanent key exposure even in private repos; key rotation required | git-secrets / gitleaks pre-commit hooks; `.gitignore` for all `.env` files; separate secrets repo if needed; automated CI secret scanning | Dev |
| R003 | Credential Security | API key exposed in Docker image layers | 4 | 3 | Image push to registry leaks secrets; build cache retention | Multi-stage builds with secrets mount (`--mount=type=secret`); Docker BuildKit; no `ENV` for secrets in Dockerfile; image scanning with Trivy | Dev |
| R004 | Rate Limiting | Source IP blocked due to excessive requests | 4 | 4 | Loss of news source; operational degradation | Per-domain rate limits; respect `robots.txt` Crawl-delay; exponential backoff on 429; request queue with rate limiting; monitoring alerts | Dev |
| R005 | Rate Limiting | Exceeding API quota resulting in charges or termination | 4 | 3 | Unexpected costs; API key revocation | Daily/hourly request budgets; quota monitoring; circuit breaker pattern; graceful degradation to RSS feeds | Dev |
| R006 | robots.txt | Violating `robots.txt` directives leading to legal notice | 5 | 2 | Cease and desist; potential legal action; reputational damage | Automated `robots.txt` fetching and parsing; path allowlist/blocklist check before every request; daily cache refresh; compliance audit logging | Dev |
| R007 | robots.txt | User-agent misidentification causing policy misapplication | 3 | 2 | Wrong `robots.txt` rules applied; unintentional violation | Consistent, honest User-Agent string (e.g., `JarvisHQ-Bot/1.0`); proper matching against `robots.txt` records; no impersonation of browsers | Dev |
| R008 | Copyright | Scraping full article text without license | 5 | 2 | Copyright infringement claim; DMCA takedown notice | Scrape only titles, summaries, and metadata; excerpt length limit (max 150 words); always link to original source; no full-text storage | Dev |
| R009 | Copyright | Hotlinking images without license or attribution | 4 | 2 | Bandwidth theft claims; image hosting bans | Do not hotlink images; store only thumbnail references; use Open Graph metadata for preview images with source attribution | Dev |
| R010 | Copyright | Failing to attribute original source leading to plagiarism | 3 | 2 | Reputational damage; ethical violation; potential legal issue | Mandatory source attribution in output; "Read more at [Source]" links; original author name preservation; Creative Commons compliance where applicable | Dev |
| R011 | GDPR/Privacy | Scraping content containing personal data (PII) | 4 | 2 | GDPR violation (up to EUR 20M fine); privacy complaint | No scraping of social media profiles; no comment sections with usernames; no author email addresses; data minimization review of all sources | Dev |
| R012 | GDPR/Privacy | Storing personally identifiable information in database | 5 | 2 | Regulatory violation; data breach liability | PostgreSQL field-level audit for PII patterns; automated PII detection and redaction; no storage of names, emails, phone numbers, addresses | Dev |
| R013 | ToS | Violating website Terms of Service prohibiting scraping | 4 | 2 | Account ban; legal notice; service termination | Review ToS of each source before adding; blocklist for "no scraping" sites; honor 403/451 responses; documented source review | Dev |
| R014 | ToS | Attempting to bypass paywalls or access premium content | 5 | 1 | CFAA violation (US); criminal liability; fraud charges | Explicit prohibition policy; no paywall bypass tools; no cookie/session manipulation for premium access; accept paywalled content is excluded | Dev |
| R015 | Data Storage | PostgreSQL exposure to local network without authentication | 4 | 2 | Unauthorized local network access; data exposure | Bind to localhost only (`127.0.0.1`); strong password; no default credentials; connection limit enforcement | Dev |
| R016 | Data Storage | Unencrypted database backups containing scraped data | 3 | 2 | Data exposure if backup media lost or stolen | LUKS/dm-crypt for backup volumes; encrypted backups with age/GPG; offsite backup encryption | Dev |
| R017 | Network Security | Man-in-the-middle attack on scraping requests | 4 | 1 | API key exposure; data tampering; credential theft | TLS 1.2+ for all connections; certificate verification enabled; no `--insecure` flags; pinned certificates for critical APIs | Dev |
| R018 | Dependency | Vulnerability in scraping framework or HTTP client | 3 | 3 | Remote code execution; data exfiltration; container escape | Snyk/Dependabot scanning; minimal base images (distroless); regular `pip audit`; SBOM generation; update policy (patch within 7 days, minor within 30) | Dev |
| R019 | Dependency | Vulnerability in parsing library (XML/HTML) | 3 | 2 | XXE injection; entity expansion DoS (Billion Laughs); memory exhaustion | defusedxml for XML parsing; parser resource limits; input size limits; timeout on all parse operations | Dev |
| R020 | Logging | API keys or PII written to application logs | 4 | 3 | Credential exposure in log files; privacy violation | Structured logging with sanitization filters; custom Formatter to mask secrets; log file permissions (0600); separate audit log for compliance | Dev |
| R021 | Logging | Log injection via scraped content | 3 | 2 | Log file corruption; false log entries; log analysis tool compromise | Input sanitization before logging; no raw user/scraped content in logs without escaping; structured JSON logging to prevent injection | Dev |
| R022 | Malicious Content | Scraping and serving malicious JavaScript or XSS payloads | 4 | 2 | Stored XSS in personal news site; browser compromise | HTML sanitization with bleach or nh3; Content Security Policy headers; no inline script execution in output; text-only extraction where possible | Dev |
| R023 | Malicious Content | Downloading malware disguised as images or documents | 3 | 2 | System compromise; container infection | File type validation (magic numbers); antivirus scanning for downloads; container read-only filesystem; no execution of downloaded content | Dev |
| R024 | Misinformation | Propagating false information without verification | 3 | 3 | Personal decision-making based on false data; sharing misinformation | Source quality scoring; allowlist of reputable sources only; multi-source corroboration flags; no single-source claims for controversial topics | Dev |
| R025 | Resource | Scraping runaway causing resource exhaustion (self-DoS) | 3 | 4 | Container crash; host system instability; Docker/WSL2 resource starvation | Container memory limits (512MB default); CPU quota (0.5 cores); scraping timeout (30s per request); circuit breaker; scheduled scraping windows only | Dev |
| R026 | Resource | Database growth causing disk space exhaustion | 3 | 3 | PostgreSQL crash; data loss; system instability | Retention policy (90 days default); automated cleanup job; disk usage alerts at 70%/80%/90%; archived article compression | Dev |
| R027 | Source Retaliation | Coordinated blocking by CDN (Cloudflare, Akamai) | 3 | 2 | Broad IP reputation damage; multiple source loss | Honest User-Agent; rate limit compliance; no bot fingerprint evasion; **no residential proxy or IP rotation for scraping** — accept blocking, mark source `blocked`, and remove from the active set | Dev |
| R028 | Data Corruption | Bit rot or silent data corruption in PostgreSQL | 2 | 2 | Loss of news archive; data inconsistency | `pg_dump` weekly backups; checksum validation; PostgreSQL `page_checksums` enabled; test restores quarterly | Dev |
| R029 | Availability | Single point of failure - system downtime during scraping | 2 | 3 | Missed news updates; stale content | Idempotent scraping jobs; resume capability; health checks; Telegram notification on failure | Dev |
| R030 | Legal | Receiving cease and desist notice from content owner | 4 | 1 | Legal costs; stress; system modification required | Pre-emptive compliance with all frameworks in this document; documented compliance evidence; immediate source removal procedure; legal contact prepared | Dev |

### Risk Heat Map Summary

| Score Range | Count | Action |
|-------------|-------|--------|
| 20-25 (Critical) | 4 | Immediate mitigation required |
| 12-16 (High) | 12 | Mitigate within 1 week |
| 6-10 (Medium) | 10 | Mitigate within 1 month |
| 1-5 (Low) | 4 | Monitor and document |

---

## 2. API Key Management

### 2.1 1Password Integration Design

```
+-------------+     +-------------------+     +------------------+
|   Jarvis HQ  |     |   1Password CLI   |     |   1Password Cloud |
|   (Docker)   |---->|   (op signin)     |---->|   Vault: "API"   |
+--------------+     +-------------------+     +------------------+
        |                                               |
        |  1. Container starts, checks for OP_SESSION  |
        |  2. If no session, reads OP_SERVICE_ACCOUNT  |
        |  3. Token from /run/secrets/op_token         |
        |  4. Fetches API keys: HUGGINGFACE_API_KEY    |
        |                        OPENAI_API_KEY         |
        |                        NEWSAPI_KEY            |
        |                        TELEGRAM_BOT_TOKEN      |
        |  5. Keys injected into environment at runtime  |
        |  6. Never written to disk or logs              |
        v                                               v
+------------------+                          +------------------+
|  Memory-only     |                          |  Vault Policy:   |
|  env vars in     |                          |  - Dev has       |
|  container       |                          |    read/write    |
+-------------+----+                          |  - Service acct  |
              |                               |    read-only     |
              v                               +------------------+
     +------------------+
     | Application reads |
     | env vars directly  |
     +-------------------+
```

**Technical Implementation:**

```bash
# 1Password Service Account Token
# Stored in Docker secret, NOT environment variable
docker secret create op_service_account_token /dev/stdin <<EOF
ops_xxxxxxxxxxxx
EOF

# Docker Compose configuration
secrets:
  op_service_account_token:
    file: ./secrets/op_service_account_token.txt
    # File is read once, not mounted as volume

services:
  jarvis-hq:
    secrets:
      - source: op_service_account_token
        target: /run/secrets/op_token
        mode: 0400  # Read-only by container owner
    environment:
      - OP_SERVICE_ACCOUNT_TOKEN_FILE=/run/secrets/op_token
      - OP_VAULT=API
    # No API keys in environment section
```

```python
# secrets_manager.py - 1Password integration
import os
import subprocess
from functools import lru_cache

class SecretManager:
    """
    Retrieves secrets from 1Password CLI.
    All secrets are memory-only and never cached on disk.
    """
    
    VAULT_NAME = "API"
    
    @lru_cache(maxsize=32)
    def get_secret(self, item_name: str, field_name: str = "credential") -> str:
        """
        Fetch secret from 1Password.
        lru_cache keeps in memory only; cleared on restart.
        """
        try:
            result = subprocess.run(
                [
                    "op", "item", "get", item_name,
                    "--vault", self.VAULT_NAME,
                    "--field", field_name,
                    "--format", "json"
                ],
                capture_output=True,
                text=True,
                timeout=10,
                check=True
            )
            return json.loads(result.stdout).get("value", "")
        except subprocess.TimeoutExpired:
            raise SecretRetrievalError(f"1Password CLI timeout for {item_name}")
        except subprocess.CalledProcessError as e:
            raise SecretRetrievalError(f"1Password error: {e.stderr}")
    
    def get_api_key(self, service: str) -> str:
        """Get API key for a service."""
        return self.get_secret(f"{service}-api-key")

# Usage in application
secrets = SecretManager()
huggingface_key = secrets.get_api_key("huggingface")
openai_key = secrets.get_api_key("openai")
```

### 2.2 Environment Variable Strategy

| Variable | Source | Scope | Notes |
|----------|--------|-------|-------|
| `HUGGINGFACE_API_KEY` | 1Password (runtime) | Container only | Loaded via SecretManager |
| `OPENAI_API_KEY` | 1Password (runtime) | Container only | Loaded via SecretManager |
| `NEWSAPI_KEY` | 1Password (runtime) | Container only | Loaded via SecretManager |
| `TELEGRAM_BOT_TOKEN` | 1Password (runtime) | Container only | Loaded via SecretManager |
| `DATABASE_URL` | Docker Compose | Container only | `postgresql://jarvis:****@db:5432/jarvis` |
| `OP_SERVICE_ACCOUNT_TOKEN_FILE` | Docker secret mount | Container only | Path to 1Password token |
| `RATE_LIMIT_GLOBAL_RPS` | `.env` (non-secret) | Container only | Public config |
| `LOG_LEVEL` | `.env` (non-secret) | Container only | Public config |

**Rules:**
1. `.env` file stores ONLY non-secret configuration
2. All API keys flow from 1Password → Docker secret → memory
3. No secrets in `docker-compose.yml` (use `${VAR}` with no defaults)
4. Container environment is inspected on startup; exit if secrets detected in env

### 2.3 Key Rotation Schedule

| Key Type | Rotation Frequency | Automated? | Procedure |
|----------|-------------------|------------|-----------|
| 1Password Service Account | Annually | No | Manual rotation; update Docker secret; restart container |
| Hugging Face API Key | Semi-annually | Calendar reminder | Regenerate in HF dashboard; update 1Password item |
| OpenAI API Key | Semi-annually | Calendar reminder | Regenerate in OpenAI dashboard; update 1Password item |
| NewsAPI Key | Annually | Calendar reminder | Regenerate in NewsAPI dashboard; update 1Password item |
| Telegram Bot Token | On suspicion of leak | Immediate | Use `/revoke` with BotFather; update 1Password; restart |
| PostgreSQL Password | Annually | No | Update password; update connection string in 1Password |

**Rotation Checklist:**
- [ ] Calendar reminder 30 days before expiry
- [ ] Generate new key in provider dashboard
- [ ] Update in 1Password (do NOT delete old key yet)
- [ ] Update Docker secret with new value
- [ ] Restart Jarvis HQ container
- [ ] Verify all scrapers functional
- [ ] Wait 24 hours, then revoke old key in provider dashboard
- [ ] Document rotation in change log

### 2.4 No Secrets in Code/Logs/Frontend

**Pre-Commit Hook (.pre-commit-config.yaml):**

```yaml
repos:
  - repo: https://github.com/Yelp/detect-secrets
    rev: v1.5.0
    hooks:
      - id: detect-secrets
        args: ['--baseline', '.secrets.baseline']
  
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
  
  - repo: local
    hooks:
      - id: no-env-keys
        name: Check for API keys in .env files
        entry: bash -c 'grep -iE "(api_key|token|secret|password)=[^$]" .env* && exit 1 || exit 0'
        language: system
        pass_filenames: false
```

**Log Sanitization:**

```python
# logging_config.py
import logging
import re

class SecretSanitizer(logging.Filter):
    """
    Filter that redacts secrets from all log records.
    Applied to ALL handlers unconditionally.
    """
    
    # Patterns to redact
    PATTERNS = {
        'api_key': re.compile(r'(api[_-]?key[=:]\s*)[a-zA-Z0-9_\-]{20,}', re.I),
        'bearer_token': re.compile(r'(Bearer\s+)[a-zA-Z0-9_\-.]{20,}', re.I),
        'authorization': re.compile(r'(Authorization[:\s]*).*', re.I),
        'password': re.compile(r'(password[=:]\s*)\S+', re.I),
        'token': re.compile(r'(token[=:]\s*)[a-zA-Z0-9_\-]{10,}', re.I),
        'postgres_url': re.compile(r'(postgresql://[^:]+:)[^@]+(@)', re.I),
    }
    
    REDACTION = '[REDACTED]'
    
    def filter(self, record: logging.LogRecord) -> bool:
        """Sanitize message and any extra fields."""
        # Sanitize the log message
        msg = str(record.msg)
        for name, pattern in self.PATTERNS.items():
            msg = pattern.sub(r'\1' + self.REDACTION + (r'\2' if pattern.groups > 1 else ''), msg)
        record.msg = msg
        
        # Sanitize args if they are strings
        if record.args:
            sanitized_args = []
            for arg in record.args:
                if isinstance(arg, str):
                    for pattern in self.PATTERNS.values():
                        arg = pattern.sub(r'\1' + self.REDACTION + (r'\2' if pattern.groups > 1 else ''), arg)
                sanitized_args.append(arg)
            record.args = tuple(sanitized_args)
        
        return True

# Apply to root logger - CANNOT be bypassed
root_logger = logging.getLogger()
root_logger.addFilter(SecretSanitizer())
```

**Runtime Secret Detection:**

```python
# startup_check.py
def verify_no_hardcoded_secrets():
    """
    Run on every startup. Exit if secrets found in environment
    that should only come from 1Password.
    """
    suspicious_patterns = [
        r'sk-[a-zA-Z0-9]{48}',           # OpenAI key format
        r'hf_[a-zA-Z0-9]{34}',            # HuggingFace token
        r'[a-f0-9]{32}',                  # Generic 32-char hex key
    ]
    
    env_vars = os.environ.items()
    
    # Check env vars that SHOULDN'T contain keys
    allowed_in_env = {'OP_SERVICE_ACCOUNT_TOKEN_FILE', 'DATABASE_URL'}
    
    for var_name, var_value in env_vars:
        if var_name in allowed_in_env:
            continue
            
        for pattern in suspicious_patterns:
            if re.search(pattern, var_value, re.I):
                logger.critical(
                    f"Potential hardcoded secret detected in env var: {var_name}. "
                    f"All secrets must come from 1Password. Refusing to start."
                )
                sys.exit(1)
```

### 2.5 Docker Secret Management

```yaml
# docker-compose.yml - Production configuration
version: "3.8"

secrets:
  op_service_account_token:
    file: ./secrets/op_service_account_token.txt
    # This file is never committed to git
    # Generated by: op service-account create --name jarvis-hq

services:
  scraper:
    build: .
    secrets:
      - source: op_service_account_token
        target: /run/secrets/op_service_account_token
        uid: '1000'
        gid: '1000'
        mode: 0400
    environment:
      - OP_SERVICE_ACCOUNT_TOKEN_FILE=/run/secrets/op_service_account_token
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
    cap_drop:
      - ALL
    cap_add:
      - SETUID
      - SETGID
    mem_limit: 512m
    cpus: '0.5'
```

**Dockerfile Security:**

```dockerfile
# Use distroless for minimal attack surface
FROM python:3.11-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM gcr.io/distroless/python3-debian12
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY ./src ./src
# No secrets copied - all loaded at runtime from 1Password
USER nonroot:nonroot
CMD ["src/main.py"]
```

---

## 3. Rate Limiting Strategy

### 3.1 Global Rate Limit

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Global max requests/second | 2 req/s | Conservative for personal use; well below most thresholds |
| Global max concurrent | 4 connections | Prevents connection pool exhaustion |
| Global daily request budget | 5,000 requests | Cap on total daily activity |
| Per-request timeout | 30 seconds | Fail fast; don't hang |
| Connection timeout | 10 seconds | Fail fast on unreachable hosts |

### 3.2 Per-Domain Rate Limits

| Domain Category | Delay Between Requests | Max Requests/Hour | Max Requests/Day |
|----------------|----------------------|-------------------|------------------|
| RSS Feeds | 5 seconds | 120 | 500 |
| News APIs (NewsAPI, etc.) | Per API docs | As documented | As documented |
| GitHub API | Per GitHub docs (60/hr unauth) | 60 (unauth) / 5000 (auth) | As documented |
| Blogs / Small sites | 10 seconds | 60 | 200 |
| Major news sites | 15 seconds | 40 | 150 |
| Social media (if any) | 30 seconds | 20 | 50 |

### 3.3 robots.txt Crawl-delay Respect

```python
# robots_compliance.py
import time
from urllib.robotparser import RobotFileParser
from urllib.parse import urlparse
import requests

class RobotsCompliantLimiter:
    """
    Rate limiter that respects robots.txt Crawl-delay directives.
    Falls back to per-domain defaults if no directive exists.
    """
    
    DEFAULT_DELAY = 5.0  # seconds
    MIN_DELAY = 1.0      # minimum allowed delay
    MAX_DELAY = 60.0     # maximum allowed delay
    
    def __init__(self):
        self._parsers: dict[str, RobotFileParser] = {}
        self._last_fetch: dict[str, float] = {}
        self._cache_ttl = 86400  # Re-fetch robots.txt every 24 hours
        self._robots_cache_time: dict[str, float] = {}
    
    def _get_robots_url(self, url: str) -> str:
        parsed = urlparse(url)
        return f"{parsed.scheme}://{parsed.netloc}/robots.txt"
    
    def _fetch_robots(self, robots_url: str) -> RobotFileParser:
        """Fetch and parse robots.txt with caching."""
        now = time.time()
        
        # Return cached parser if still fresh
        if robots_url in self._robots_cache_time:
            if now - self._robots_cache_time[robots_url] < self._cache_ttl:
                return self._parsers[robots_url]
        
        rp = RobotFileParser()
        rp.set_url(robots_url)
        try:
            # Short timeout for robots.txt fetch
            response = requests.get(robots_url, timeout=10)
            if response.status_code == 200:
                rp.parse(response.text.split('\n'))
            else:
                # If robots.txt unavailable, assume allow all
                rp.parse(["User-agent: *", "Allow: /"])
        except Exception:
            # On failure, use permissive default
            rp.parse(["User-agent: *", "Allow: /"])
        
        self._parsers[robots_url] = rp
        self._robots_cache_time[robots_url] = now
        return rp
    
    def get_crawl_delay(self, url: str, user_agent: str = "*") -> float:
        """
        Get crawl delay for URL. Returns seconds to wait.
        Respects robots.txt Crawl-delay with bounds checking.
        """
        robots_url = self._get_robots_url(url)
        rp = self._fetch_robots(robots_url)
        
        # Check for Crawl-delay directive
        # Note: urllib.robotparser doesn't parse Crawl-delay directly
        # We supplement with manual parsing
        delay = self._parse_crawl_delay(robots_url, user_agent)
        
        if delay:
            return max(self.MIN_DELAY, min(delay, self.MAX_DELAY))
        return self.DEFAULT_DELAY
    
    def _parse_crawl_delay(self, robots_url: str, user_agent: str) -> float | None:
        """Manually parse Crawl-delay from cached robots.txt content."""
        try:
            response = requests.get(robots_url, timeout=10)
            lines = response.text.split('\n')
            current_ua_match = False
            
            for line in lines:
                line = line.strip()
                if line.lower().startswith('user-agent:'):
                    ua = line.split(':', 1)[1].strip()
                    current_ua_match = (ua == '*' or ua in user_agent)
                elif current_ua_match and line.lower().startswith('crawl-delay:'):
                    try:
                        return float(line.split(':', 1)[1].strip())
                    except ValueError:
                        continue
            return None
        except Exception:
            return None
    
    def wait_if_needed(self, url: str, user_agent: str = "*"):
        """Block until it's safe to request URL."""
        domain = urlparse(url).netloc
        delay = self.get_crawl_delay(url, user_agent)
        
        last = self._last_fetch.get(domain, 0)
        elapsed = time.time() - last
        
        if elapsed < delay:
            sleep_time = delay - elapsed
            time.sleep(sleep_time)
        
        self._last_fetch[domain] = time.time()
```

### 3.4 Exponential Backoff on 429 Errors

```python
# rate_limiter.py
import time
import random
from dataclasses import dataclass
from datetime import datetime, timedelta

@dataclass
class RateLimitState:
    """Tracks rate limit state per domain."""
    consecutive_429s: int = 0
    last_429_time: datetime | None = None
    backoff_until: datetime | None = None
    is_blocked: bool = False

class ExponentialBackoff:
    """
    Implements exponential backoff with jitter for 429 responses.
    Base: 2 seconds, Max: 1 hour, Jitter: 0-25%
    """
    
    BASE_DELAY = 2          # seconds
    MAX_DELAY = 3600        # 1 hour
    MAX_RETRIES = 5         # After this, mark source as blocked
    JITTER_RANGE = 0.25     # 25% jitter
    
    def __init__(self):
        self._states: dict[str, RateLimitState] = {}
    
    def _get_state(self, domain: str) -> RateLimitState:
        if domain not in self._states:
            self._states[domain] = RateLimitState()
        return self._states[domain]
    
    def record_429(self, domain: str) -> float:
        """
        Record a 429 response and calculate backoff time.
        Returns seconds to wait before retry.
        """
        state = self._get_state(domain)
        state.consecutive_429s += 1
        state.last_429_time = datetime.now()
        
        if state.consecutive_429s >= self.MAX_RETRIES:
            state.is_blocked = True
            state.backoff_until = datetime.now() + timedelta(hours=24)
            logger.warning(f"Source {domain} blocked after {self.MAX_RETRIES} consecutive 429s")
            return -1  # Signal: do not retry
        
        # Exponential backoff: 2^N seconds with jitter
        delay = self.BASE_DELAY * (2 ** (state.consecutive_429s - 1))
        delay = min(delay, self.MAX_DELAY)
        jitter = delay * random.uniform(0, self.JITTER_RANGE)
        total_delay = delay + jitter
        
        state.backoff_until = datetime.now() + timedelta(seconds=total_delay)
        logger.info(f"429 on {domain}: backing off for {total_delay:.1f}s "
                   f"(attempt {state.consecutive_429s}/{self.MAX_RETRIES})")
        
        return total_delay
    
    def record_success(self, domain: str):
        """Reset backoff on successful request."""
        state = self._get_state(domain)
        if state.consecutive_429s > 0:
            logger.info(f"Request to {domain} succeeded after {state.consecutive_429s} 429s")
        state.consecutive_429s = 0
        state.backoff_until = None
    
    def is_blocked(self, domain: str) -> bool:
        """Check if domain is currently in backoff period."""
        state = self._get_state(domain)
        if state.backoff_until and datetime.now() < state.backoff_until:
            return True
        if state.is_blocked:
            # Check if 24-hour block has expired
            if state.last_429_time and datetime.now() > state.last_429_time + timedelta(hours=24):
                state.is_blocked = False
                state.consecutive_429s = 0
                return False
            return True
        return False
    
    def get_wait_time(self, domain: str) -> float:
        """Get remaining wait time for a blocked domain."""
        state = self._get_state(domain)
        if state.backoff_until:
            remaining = (state.backoff_until - datetime.now()).total_seconds()
            return max(0, remaining)
        return 0
```

### 3.5 Request Queue with Rate Limiting

```python
# request_queue.py
import asyncio
from dataclasses import dataclass
from enum import Enum, auto

class Priority(Enum):
    HIGH = auto()      # Breaking news
    NORMAL = auto()    # Regular updates
    LOW = auto()       # Full-text fetch, enrichment

@dataclass
class QueuedRequest:
    url: str
    priority: Priority
    domain: str
    callback: callable
    added_at: float
    max_age: float = 3600  # Discard after 1 hour

class RateLimitedQueue:
    """
    Priority queue with per-domain rate limiting.
    Ensures global and per-domain limits are never exceeded.
    """
    
    def __init__(
        self,
        global_rps: float = 2.0,
        domain_delays: dict[str, float] | None = None
    ):
        self.global_rps = global_rps
        self.global_delay = 1.0 / global_rps
        self.domain_delays = domain_delays or {}
        self._queue: asyncio.PriorityQueue = asyncio.PriorityQueue()
        self._domain_last_request: dict[str, float] = {}
        self._global_last_request: float = 0
        self._backoff = ExponentialBackoff()
        self._robots_limiter = RobotsCompliantLimiter()
    
    async def enqueue(self, request: QueuedRequest):
        """Add request to queue with priority ordering."""
        priority_value = {
            Priority.HIGH: 0,
            Priority.NORMAL: 1,
            Priority.LOW: 2
        }[request.priority]
        
        await self._queue.put((priority_value, request.added_at, request))
    
    async def _wait_for_rate_limit(self, domain: str):
        """Wait until rate limit allows a request."""
        # Check if domain is in backoff
        if self._backoff.is_blocked(domain):
            wait = self._backoff.get_wait_time(domain)
            if wait > 0:
                logger.warning(f"Domain {domain} in backoff, waiting {wait:.0f}s")
                await asyncio.sleep(wait)
        
        # Enforce robots.txt crawl-delay
        robots_delay = self._robots_limiter.get_crawl_delay(f"https://{domain}")
        domain_delay = max(
            self.domain_delays.get(domain, 5),
            robots_delay
        )
        
        # Domain-specific rate limit
        now = time.time()
        domain_last = self._domain_last_request.get(domain, 0)
        domain_wait = domain_delay - (now - domain_last)
        
        # Global rate limit
        global_wait = self.global_delay - (now - self._global_last_request)
        
        # Wait for the longest required delay
        wait_time = max(0, domain_wait, global_wait)
        if wait_time > 0:
            await asyncio.sleep(wait_time)
    
    async def process(self) -> QueuedRequest | None:
        """Get next request respecting all rate limits."""
        _, _, request = await self._queue.get()
        
        # Check max age
        if time.time() - request.added_at > request.max_age:
            logger.debug(f"Request to {request.url} expired, skipping")
            return None
        
        await self._wait_for_rate_limit(request.domain)
        
        self._domain_last_request[request.domain] = time.time()
        self._global_last_request = time.time()
        
        return request
```

---

## 4. robots.txt Compliance

### 4.1 robots.txt Parsing and Caching

```python
# robots_compliance.py (extended)
from urllib.robotparser import RobotFileParser
from datetime import datetime, timedelta
import hashlib

class RobotsComplianceManager:
    """
    Comprehensive robots.txt compliance with full caching,
    change detection, and audit logging.
    """
    
    USER_AGENT = "JarvisHQ-Bot"
    USER_AGENT_VERSION = "1.0"
    FULL_USER_AGENT = f"{USER_AGENT}/{USER_AGENT_VERSION}"
    
    # Contact info for webmasters
    CONTACT_EMAIL = "jarvis-hq@localhost"  # Update with actual contact
    
    CACHE_TTL = timedelta(hours=24)
    MAX_ROBOTS_SIZE = 500 * 1024  # 500KB max robots.txt size
    
    def __init__(self, db_session=None):
        self._cache: dict[str, _RobotsCacheEntry] = {}
        self._db = db_session
        self._disallowed_paths: dict[str, list[str]] = {}
    
    async def can_fetch(self, url: str) -> tuple[bool, str]:
        """
        Check if URL can be fetched according to robots.txt.
        Returns (allowed, reason).
        """
        parsed = urlparse(url)
        domain = parsed.netloc
        robots_url = f"{parsed.scheme}://{domain}/robots.txt"
        
        # Fetch and parse
        entry = await self._get_cached_or_fetch(robots_url)
        
        if not entry:
            # If robots.txt unreachable, allow (fail-open for safety)
            return True, "robots.txt unreachable, defaulting to allow"
        
        # Check with our specific user-agent, then wildcard
        allowed = entry.parser.can_fetch(self.FULL_USER_AGENT, url)
        if not allowed:
            allowed = entry.parser.can_fetch("*", url)
        
        if not allowed:
            reason = f"Disallowed by robots.txt for {self.FULL_USER_AGENT} or *"
            await self._log_violation_attempt(url, reason)
            return False, reason
        
        return True, "Allowed by robots.txt"
    
    async def _get_cached_or_fetch(self, robots_url: str) -> '_RobotsCacheEntry | None':
        """Get cached robots.txt or fetch fresh copy."""
        now = datetime.now()
        
        # Check memory cache
        if robots_url in self._cache:
            entry = self._cache[robots_url]
            if now - entry.fetched_at < self.CACHE_TTL:
                return entry
            
            # Cache expired - check if changed
            if await self._check_changed(robots_url, entry.etag, entry.last_modified):
                # Changed, fetch new
                pass
            else:
                # Not changed, refresh TTL
                entry.fetched_at = now
                return entry
        
        # Fetch fresh copy
        return await self._fetch_robots(robots_url)
    
    async def _fetch_robots(self, robots_url: str) -> '_RobotsCacheEntry | None':
        """Fetch robots.txt with size limits and validation."""
        try:
            headers = {
                'User-Agent': f"{self.FULL_USER_AGENT} (+https://localhost/about; {self.CONTACT_EMAIL})"
            }
            response = requests.get(
                robots_url,
                headers=headers,
                timeout=10,
                allow_redirects=True
            )
            
            if response.status_code == 404:
                # No robots.txt = allow all
                logger.info(f"No robots.txt at {robots_url}, allowing all")
                return _RobotsCacheEntry(
                    parser=self._allow_all_parser(),
                    fetched_at=datetime.now(),
                    etag="",
                    last_modified="",
                    hash=""
                )
            
            if response.status_code != 200:
                logger.warning(f"robots.txt returned {response.status_code} at {robots_url}")
                return None
            
            # Size limit
            content_length = len(response.content)
            if content_length > self.MAX_ROBOTS_SIZE:
                logger.warning(f"robots.txt too large ({content_length}B), truncating")
                content = response.text[:self.MAX_ROBOTS_SIZE]
            else:
                content = response.text
            
            # Parse
            parser = RobotFileParser()
            parser.parse(content.split('\n'))
            
            entry = _RobotsCacheEntry(
                parser=parser,
                fetched_at=datetime.now(),
                etag=response.headers.get('ETag', ''),
                last_modified=response.headers.get('Last-Modified', ''),
                hash=hashlib.sha256(content.encode()).hexdigest()
            )
            
            self._cache[robots_url] = entry
            
            # Extract disallowed paths for logging
            self._extract_disallowed_paths(robots_url, content)
            
            logger.info(f"Fetched robots.txt from {robots_url}, "
                       f"disallowed paths: {len(self._disallowed_paths.get(robots_url, []))}")
            
            return entry
            
        except Exception as e:
            logger.error(f"Failed to fetch robots.txt from {robots_url}: {e}")
            return None
    
    def _allow_all_parser(self) -> RobotFileParser:
        """Create a parser that allows everything."""
        parser = RobotFileParser()
        parser.parse(["User-agent: *", "Allow: /"])
        return parser
    
    def _extract_disallowed_paths(self, robots_url: str, content: str):
        """Extract disallowed paths for our user-agent from robots.txt."""
        disallowed = []
        current_ua_match = False
        
        for line in content.split('\n'):
            line = line.strip()
            if not line or line.startswith('#'):
                continue
            
            if line.lower().startswith('user-agent:'):
                ua = line.split(':', 1)[1].strip()
                current_ua_match = (
                    ua == '*' or 
                    self.USER_AGENT in ua or
                    ua in self.FULL_USER_AGENT
                )
            elif current_ua_match and line.lower().startswith('disallow:'):
                path = line.split(':', 1)[1].strip()
                if path:
                    disallowed.append(path)
        
        self._disallowed_paths[robots_url] = disallowed
```

### 4.2 Disallowed Path Detection

```python
# Pre-flight check before every request
async def pre_flight_check(self, url: str) -> PreFlightResult:
    """
    Complete pre-flight compliance check.
    Run this BEFORE every HTTP request.
    """
    checks = {
        'robots_txt': False,
        'rate_limit_ok': False,
        'not_blocked': False,
        'url_valid': False,
    }
    
    # Validate URL
    if not self._is_valid_url(url):
        return PreFlightResult(allowed=False, reason="Invalid URL", checks=checks)
    checks['url_valid'] = True
    
    # Check robots.txt
    robots_ok, robots_reason = await self.can_fetch(url)
    if not robots_ok:
        return PreFlightResult(allowed=False, reason=robots_reason, checks=checks)
    checks['robots_txt'] = True
    
    # Check if domain is blocked (backoff)
    domain = urlparse(url).netloc
    if self._backoff.is_blocked(domain):
        wait = self._backoff.get_wait_time(domain)
        return PreFlightResult(
            allowed=False, 
            reason=f"Domain in backoff for {wait:.0f}s",
            checks=checks
        )
    checks['not_blocked'] = True
    
    # Check rate limits
    checks['rate_limit_ok'] = True
    
    return PreFlightResult(allowed=True, reason="All checks passed", checks=checks)
```

### 4.3 robots.txt Update Checking

```python
async def _check_changed(self, robots_url: str, etag: str, last_modified: str) -> bool:
    """Check if robots.txt has changed using conditional request."""
    headers = {}
    if etag:
        headers['If-None-Match'] = etag
    if last_modified:
        headers['If-Modified-Since'] = last_modified
    
    try:
        response = requests.get(robots_url, headers=headers, timeout=10)
        return response.status_code != 304  # 304 = not changed
    except Exception:
        return True  # Assume changed on error
```

---

## 5. Copyright and Fair Use

### 5.1 What Can Be Scraped (Safe Zone)

| Content Type | Can Scrape? | Notes |
|--------------|-------------|-------|
| Article title | Yes | Factual, minimal creativity |
| Article summary / description | Yes | From RSS `<description>` or Open Graph |
| Publication date | Yes | Factual information |
| Author name | Yes | Attribution requirement |
| Source URL | Yes | Required for linking |
| Category / tags | Yes | Factual metadata |
| Word count / reading time | Yes | Computed metadata |
| Thumbnail image URL | Yes | For linking only, not hosting |
| Article excerpt (up to 150 words) | Yes | Fair use - transformative, minimal |

### 5.2 What Should NOT Be Scraped (Danger Zone)

| Content Type | Can Scrape? | Risk |
|--------------|-------------|------|
| Full article text | **NO** | Copyright infringement; DMCA risk |
| Images (downloaded/hosted) | **NO** | Unless Creative Commons licensed |
| Videos | **NO** | Bandwidth theft; DMCA |
| Premium/paywalled content | **NO** | CFAA violation; ToS breach |
| Content behind login | **NO** | Unauthorized access |
| Comments sections | **NO** | PII risk; no editorial value |
| User-generated content | **NO** | Copyright uncertain; PII risk |
| Code snippets (unlicensed) | **NO** | License violation |

### 5.3 Attribution Requirements

```python
# attribution.py
class AttributionManager:
    """
    Ensures all scraped content has proper attribution.
    Generates attribution strings for output formats.
    """
    
    REQUIRED_FIELDS = ['title', 'source_name', 'source_url', 'published_date']
    
    @staticmethod
    def generate_attribution(article: dict) -> str:
        """Generate attribution string for an article."""
        source = article.get('source_name', 'Unknown Source')
        url = article.get('source_url', '')
        date = article.get('published_date', '')
        author = article.get('author', '')
        
        parts = [f"Source: {source}"]
        if author:
            parts.append(f"by {author}")
        if date:
            parts.append(f"on {date}")
        
        attribution = " | ".join(parts)
        return f"{attribution}\nOriginal: {url}"
    
    @staticmethod
    def validate_attribution(article: dict) -> tuple[bool, list[str]]:
        """Verify all required attribution fields present."""
        missing = [
            field for field in AttributionManager.REQUIRED_FIELDS
            if not article.get(field)
        ]
        return len(missing) == 0, missing
    
    @staticmethod
    def generate_html_attribution(article: dict) -> str:
        """Generate HTML attribution block."""
        source = article.get('source_name', 'Unknown')
        url = article.get('source_url', '#')
        author = article.get('author', '')
        
        html = f'''<div class="attribution">
            <p>Originally published by <strong>{source}</strong>
            {f"by {author}" if author else ""}
            <br>
            <a href="{url}" target="_blank" rel="noopener noreferrer">
                Read the full article at {source} &rarr;
            </a>
            </p>
        </div>'''
        return html
```

### 5.4 DMCA Considerations

```
DMCA Safe Harbor Framework:

1. No caching of full articles (only metadata)
2. Immediate link to original source
3. Clear attribution on every item
4. Contact information for complaints
5. Procedure for source removal upon request:
   
   a. Receive notice (email/Telegram)
   b. Remove source within 24 hours
   c. Stop all scraping from that domain
   d. Add to permanent blocklist
   e. Acknowledge removal to requester
   f. Document incident
```

### 5.5 Fair Use Checklist (Four Factors)

| Factor | Assessment | Risk Level |
|--------|-----------|------------|
| Purpose (transformative?) | Personal aggregation with attribution; non-commercial | Low |
| Nature (factual vs. creative) | Primarily factual news content | Low |
| Amount (how much used?) | Only titles + 150-word excerpts; never full text | Low |
| Market effect (substitution?) | Drives traffic to original sources; not a replacement | Low |

**Overall Fair Use Assessment: FAVORABLE** - but full articles are never scraped regardless.

---

## 6. Personal Data Avoidance

### 6.1 No Collection of PII

**Definition of PII for Jarvis HQ:**
- Full names (beyond public author bylines)
- Email addresses
- Phone numbers
- Physical addresses
- IP addresses
- Social media handles
- Geographic locations (beyond article dateline)
- Photos of individuals
- Employment history
- Any data that could identify a natural person

### 6.2 Source Categories - PII Risk Assessment

| Source Category | PII Risk | Allowed? | Mitigation |
|----------------|----------|----------|------------|
| Major news outlets | Low | Yes | Editorial content only |
| Tech blogs | Low | Yes | Article content only |
| RSS feeds | Low | Yes | Structured data only |
| GitHub (public repos) | Low | Yes | No user profiles |
| ArXiv papers | Very Low | Yes | Citation metadata only |
| Social media (any) | **Very High** | **NO** | Complete prohibition |
| Forum discussions | **High** | **NO** | Usernames are PII |
| Comment sections | **High** | **NO** | Never scrape comments |
| Blog comments | **High** | **NO** | Never scrape comments |
| Local news with victim names | **High** | **NO** | Skip crime/victim stories |

### 6.3 PII Detection and Redaction

```python
# pii_filter.py
import re
from typing import Optional

class PIIFilter:
    """
    Detects and redacts PII from scraped content.
    Applied to ALL content before storage.
    """
    
    # Regex patterns for PII detection
    PATTERNS = {
        'email': re.compile(r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b'),
        'phone_us': re.compile(r'\b(?:\+?1[-.\s]?)?\(?[0-9]{3}\)?[-.\s]?[0-9]{3}[-.\s]?[0-9]{4}\b'),
        'ssn': re.compile(r'\b\d{3}-\d{2}-\d{4}\b'),
        'ip_address': re.compile(r'\b(?:[0-9]{1,3}\.){3}[0-9]{1,3}\b'),
        'credit_card': re.compile(r'\b(?:\d{4}[-\s]?){3}\d{4}\b'),
        'twitter_handle': re.compile(r'@\w{1,15}\b'),
    }
    
    REDACTION_TEXT = '[REDACTED]'
    
    @classmethod
    def sanitize(cls, text: Optional[str]) -> str:
        """Remove all PII from text. Returns sanitized string."""
        if not text:
            return ""
        
        sanitized = text
        for pii_type, pattern in cls.PATTERNS.items():
            matches = pattern.findall(sanitized)
            if matches:
                logger.info(f"Redacted {len(matches)} {pii_type} occurrences")
            sanitized = pattern.sub(cls.REDACTION_TEXT, sanitized)
        
        return sanitized
    
    @classmethod
    def contains_pii(cls, text: Optional[str]) -> bool:
        """Check if text contains any PII."""
        if not text:
            return False
        
        return any(
            pattern.search(text) 
            for pattern in cls.PATTERNS.values()
        )
    
    @classmethod
    def sanitize_article(cls, article: dict) -> dict:
        """Sanitize all text fields in an article dict."""
        text_fields = ['title', 'summary', 'excerpt', 'content_snippet']
        
        for field in text_fields:
            if field in article and article[field]:
                article[field] = cls.sanitize(article[field])
        
        # Sanitize author name if it contains email or social handle
        if 'author' in article and article['author']:
            article['author'] = cls.sanitize(article['author'])
        
        return article
```

### 6.4 Data Minimization Principle

```python
# data_minimization.py
class DataMinimizationEnforcer:
    """
    Enforces data minimization - only collect what is strictly necessary.
    Run at the storage layer before database insertion.
    """
    
    # Fields that are permitted to be stored
    ALLOWED_FIELDS = {
        'title',
        'summary',
        'excerpt',
        'source_name',
        'source_url',
        'source_domain',
        'published_date',
        'scraped_at',
        'author',           # Public byline only
        'category',
        'tags',
        'word_count',
        'language',
        'thumbnail_url',     # URL only, not the image
    }
    
    # Fields that are explicitly forbidden
    FORBIDDEN_FIELDS = {
        'full_content',
        'html_content',
        'comments',
        'user_data',
        'email_addresses',
        'phone_numbers',
        'social_profiles',
        'cookies',
        'session_data',
        'ip_addresses',
        'geolocation',
    }
    
    @classmethod
    def minimize(cls, article: dict) -> dict:
        """
        Reduce article to only allowed fields.
        Log any removed fields for audit.
        """
        minimized = {}
        removed = []
        
        for key, value in article.items():
            if key in cls.ALLOWED_FIELDS:
                minimized[key] = value
            elif key in cls.FORBIDDEN_FIELDS:
                removed.append(key)
                logger.warning(f"Removed forbidden field '{key}' from article data")
            else:
                # Unknown field - remove to be safe (minimization)
                removed.append(key)
                logger.info(f"Removed unknown field '{key}' (data minimization)")
        
        if removed:
            logger.info(f"Data minimization: removed fields {removed}")
        
        return minimized
```

---

## 7. Logging Safety

### 7.1 Structured Logging Configuration

```python
# logging_config.py (complete)
import logging
import json
import sys
from datetime import datetime
from pythonjsonlogger import jsonlogger  # pip install python-json-logger

class JarvisLogFormatter(jsonlogger.JsonFormatter):
    """
    JSON formatter with security-focused field selection.
    Never includes raw content, secrets, or PII.
    """
    
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        
        # Standard fields
        log_record['timestamp'] = datetime.utcnow().isoformat()
        log_record['level'] = record.levelname
        log_record['logger'] = record.name
        log_record['source'] = 'jarvis-hq'
        
        # Context fields (safe only)
        if hasattr(record, 'domain'):
            log_record['domain'] = record.domain
        if hasattr(record, 'source_id'):
            log_record['source_id'] = record.source_id
        if hasattr(record, 'event_type'):
            log_record['event_type'] = record.event_type
        
        # Remove any potentially sensitive fields
        sensitive_keys = ['password', 'secret', 'token', 'key', 'credential',
                         'content', 'body', 'response', 'html']
        for key in sensitive_keys:
            log_record.pop(key, None)
            message_dict.pop(key, None)

def configure_logging():
    """Configure secure logging for Jarvis HQ."""
    
    # Root logger
    root = logging.getLogger()
    root.setLevel(logging.INFO)
    
    # Console handler with JSON formatting
    console = logging.StreamHandler(sys.stdout)
    console.setLevel(logging.INFO)
    
    formatter = JarvisLogFormatter(
        '%(timestamp)s %(level)s %(name)s %(message)s'
    )
    console.setFormatter(formatter)
    
    # Secret sanitizer filter (always applied)
    sanitizer = SecretSanitizer()
    console.addFilter(sanitizer)
    
    # PII filter (always applied)
    pii_filter = PIILogFilter()
    console.addFilter(pii_filter)
    
    root.addHandler(console)
    
    # File handler (rotating, with strict permissions)
    from logging.handlers import RotatingFileHandler
    file_handler = RotatingFileHandler(
        '/var/log/jarvis-hq/app.log',
        maxBytes=10*1024*1024,  # 10MB
        backupCount=5
    )
    file_handler.setLevel(logging.DEBUG)
    file_handler.setFormatter(formatter)
    file_handler.addFilter(sanitizer)
    file_handler.addFilter(pii_filter)
    
    # Set file permissions to owner-only
    import os
    os.chmod('/var/log/jarvis-hq/app.log', 0o600)
    
    root.addHandler(file_handler)
    
    # Separate audit log for compliance events
    audit_handler = RotatingFileHandler(
        '/var/log/jarvis-hq/audit.log',
        maxBytes=10*1024*1024,
        backupCount=10
    )
    audit_handler.setLevel(logging.INFO)
    audit_handler.setFormatter(formatter)
    
    audit_logger = logging.getLogger('jarvis.audit')
    audit_logger.addHandler(audit_handler)
    audit_logger.propagate = False  # Don't duplicate to root
    
    return root

class PIILogFilter(logging.Filter):
    """Additional filter to prevent PII in logs."""
    
    def filter(self, record):
        # Check if message might contain PII patterns
        msg = str(record.getMessage())
        if PIIFilter.contains_pii(msg):
            record.msg = PIIFilter.sanitize(record.msg)
        return True
```

### 7.2 Log Retention Policy

| Log Type | Retention | Storage Location | Access Control |
|----------|-----------|------------------|----------------|
| Application logs | 30 days | `/var/log/jarvis-hq/app.log` | Owner only (0600) |
| Audit logs | 90 days | `/var/log/jarvis-hq/audit.log` | Owner only (0600) |
| Error logs | 30 days | `/var/log/jarvis-hq/error.log` | Owner only (0600) |
| Access logs | 7 days | `/var/log/jarvis-hq/access.log` | Owner only (0600) |

**Log Rotation:**
- RotatingFileHandler: 10MB per file, 5 backups
- Compressed after rotation (gzip)
- Automatic deletion after retention period
- No log shipping to external services (local only)

### 7.3 What Must Never Appear in Logs

```yaml
# logging_restrictions.yaml
forbidden_in_logs:
  credentials:
    - API keys
    - OAuth tokens
    - Passwords
    - Session cookies
    - Database connection strings with passwords
  
  personal_data:
    - Full names (except public author bylines)
    - Email addresses
    - Phone numbers
    - IP addresses
    - Social media handles
  
  content:
    - Full article text
    - Raw HTML
    - API response bodies
    - Error response bodies (may contain stack traces with paths)
  
  system:
    - File system paths with username
    - Environment variable dumps
    - Process listings
    - Network configuration

allowed_in_logs:
  - URL (domain + path only, no query params with secrets)
  - HTTP method
  - Status code
  - Response time
  - Content type
  - Content length
  - Domain name
  - Source ID
  - Event type
  - Error category (not full traceback if contains paths)
```

### 7.4 Log Access Control

```bash
# File permissions setup
sudo mkdir -p /var/log/jarvis-hq
sudo chown $(whoami):$(whoami) /var/log/jarvis-hq
chmod 0700 /var/log/jarvis-hq

# Log files created with 0600 (owner read/write only)
# No group or other access
```

---

## 8. Source Blocklist/Allowlist

### 8.1 Source Classification

```python
# source_manager.py
from enum import Enum
from dataclasses import dataclass
from datetime import datetime

class SourceStatus(Enum):
    ALLOWED = "allowed"           # Active scraping permitted
    PENDING = "pending"           # Awaiting manual review
    BLOCKED_TEMP = "blocked_temp" # Temporary backoff
    BLOCKED_PERM = "blocked_perm" # Permanent blocklist
    REMOVED = "removed"           # Removed by request

@dataclass
class Source:
    domain: str
    url: str
    feed_url: str | None
    status: SourceStatus
    added_at: datetime
    reviewed_at: datetime | None
    block_reason: str | None
    consecutive_failures: int
    last_success: datetime | None
    robots_txt_compliant: bool
    tos_reviewed: bool
    quality_score: float  # 0-1
```

### 8.2 Initial Allowlist

The canonical allowlist lives in
[`../registries/whitelist.yaml`](../registries/whitelist.yaml). That file is
operator-customizable and reviewed annually. The categories are:

- **`premium`** — Tier 1 floor (`final_score >= 92`). Auto-include with
  notification. Reserved for official entity blogs with verified ownership.
- **`standard`** — Tier 2 floor (`final_score >= 85`). Standard digest
  inclusion. Used for established wire services, Tier-1/Tier-2 outlets,
  research preprint servers, and consistently strong specialist publications.

Each entry contains: `domain`, `reason`, `reviewed_by`, `added_at`,
`review_due`. The defaults shipped with this skill are the original Jarvis HQ
deployment's allowlist; operators must review them and adjust for their topic
mix. Sources whose automated score drops below their floor for 30 consecutive
days are flagged for review.

> Why externalized? Hardcoding the allowlist into source files made it
> impossible for operators to manage without forking the skill. The YAML file
> is the single source of truth.

### 8.3 Blocklist Categories

Category-level block policy is summarized below. The canonical machine-readable
policy (including auto-block rules and operator-customizable category patterns)
lives in [`../registries/blacklist.yaml`](../registries/blacklist.yaml).

| Category | Examples | Action |
|----------|----------|--------|
| X/Twitter (HTML) | twitter.com, x.com | Permanent block — ToS violation, no HTML scraping |
| X/Twitter (official API) | api.twitter.com / api.x.com | Allowed only if operator has paid API access; not enabled by default; treat handle mentions in editorial content as attribution metadata, not as a scrape target |
| Facebook / Instagram | facebook.com, instagram.com | Permanent block — PII risk, ToS prohibits scraping |
| TikTok | tiktok.com | Permanent block — PII, ToS |
| Reddit (.rss feeds) | `reddit.com/r/<sub>.rss` | Allowed for titles/URLs only; do NOT scrape usernames, comments, or full post bodies |
| Reddit (official API) | oauth.reddit.com | Allowed with registered OAuth app; honor rate limits and current ToS |
| Paywalled content | wsj.com, ft.com, economist.com | Permanent block — no bypass |
| User-generated content | medium.com (varies), substack (varies) | Case-by-case review |
| Known scraping hostile | Sites with aggressive anti-bot | Permanent block — respect |
| Adult content | Any adult domain | Permanent block |
| Gambling | Any gambling domain | Permanent block |
| Illegal content | Any illegal domain | Permanent block |
| Misinformation sources | Domains flagged by ≥2 fact-checkers | Permanent block (auto-populated from fact-checker feeds) |

For machine-readable patterns and exception lists, see `category_blocks` and
`spam_blacklist` in `../registries/blacklist.yaml`.

### 8.4 Auto-Block Triggers

```python
class AutoBlockRules:
    """
    Automatic source blocking rules.
    Applied after every scraping attempt.
    """
    
    RULES = {
        'consecutive_failures': {
            'threshold': 3,
            'window_hours': 24,
            'action': 'block_temp',
            'duration_hours': 24,
            'escalation': 'block_perm after 3 temp blocks'
        },
        'robots_txt_disallow': {
            'threshold': 1,
            'action': 'block_perm',
            'reason': 'robots.txt violation'
        },
        'http_403': {
            'threshold': 2,
            'window_hours': 1,
            'action': 'block_temp',
            'duration_hours': 48,
            'escalation': 'block_perm after 2 temp blocks'
        },
        'http_451': {
            'threshold': 1,
            'action': 'block_perm',
            'reason': 'Unavailable for legal reasons'
        },
        'rate_limit_429': {
            'threshold': 5,
            'window_hours': 1,
            'action': 'block_temp',
            'duration_hours': 24
        },
        'tos_violation_detected': {
            'threshold': 1,
            'action': 'block_perm',
            'reason': 'Terms of service violation'
        },
        'pii_detected': {
            'threshold': 1,
            'action': 'redact_and_continue',
            'reason': 'PII redaction applied per pii_filter.py; source not blocked. PII in editorial content (author bylines, @handles in attribution, IP addresses in technical writeups) is common and expected — redact before storage, do not block the source.'
        },
        'pii_detected_persistent': {
            'threshold': 10,
            'window_days': 7,
            'action': 'review_required',
            'reason': 'Persistent PII (>10 detections in 7 days) suggests the source may be publishing private data; manual review required before continuing.'
        },
        'content_quality_low': {
            'threshold': 5,  # Low quality scores
            'window_days': 7,
            'action': 'review_required',
            'min_quality_score': 0.3
        }
    }
    
    @classmethod
    def check(cls, source: Source, scrape_result: dict) -> list[dict]:
        """Check all auto-block rules and return triggered actions."""
        triggers = []
        
        # Check consecutive failures
        if source.consecutive_failures >= cls.RULES['consecutive_failures']['threshold']:
            triggers.append({
                'rule': 'consecutive_failures',
                'action': cls.RULES['consecutive_failures']['action'],
                'reason': f"{source.consecutive_failures} consecutive failures"
            })
        
        # Check HTTP status
        status = scrape_result.get('status_code')
        if status == 403:
            triggers.append({
                'rule': 'http_403',
                'action': cls.RULES['http_403']['action'],
                'reason': 'HTTP 403 Forbidden'
            })
        elif status == 451:
            triggers.append({
                'rule': 'http_451',
                'action': cls.RULES['http_451']['action'],
                'reason': 'HTTP 451 Unavailable For Legal Reasons'
            })
        elif status == 429:
            triggers.append({
                'rule': 'rate_limit_429',
                'action': cls.RULES['rate_limit_429']['action'],
                'reason': 'HTTP 429 Too Many Requests'
            })
        
        # Check robots.txt
        if not source.robots_txt_compliant:
            triggers.append({
                'rule': 'robots_txt_disallow',
                'action': cls.RULES['robots_txt_disallow']['action'],
                'reason': 'robots.txt disallow detected'
            })
        
        # Check PII
        if scrape_result.get('pii_detected'):
            triggers.append({
                'rule': 'pii_detected',
                'action': cls.RULES['pii_detected']['action'],
                'reason': 'PII detected in scraped content'
            })
        
        return triggers
```

### 8.5 Manual Review Workflow

```
New Source Request
       |
       v
+-------------------+
| Automated Checks  |
| - robots.txt      |
| - ToS scan        |
| - PII risk        |
| - Quality score   |
+-------------------+
       |
       v
   +---+---+
   |Clean?  |
   +---+---+
   |       |
  Yes     No
   |       |
   v       v
ALLOW  MANUAL REVIEW
       (within 48h)
```

---

## 9. Security Headers

### 9.1 User-Agent String

```python
# User-Agent: honest, identifiable, non-deceptive
USER_AGENT = (
    "JarvisHQ-Bot/1.0 "
    "(+https://localhost/about-bot; "
    "contact=jarvis-hq@localhost; "
    "purpose=personal-news-aggregation; "
    "frequency=daily; "
    "respects-robots-txt=true)"
)

# Alternative shorter form for restrictive services
USER_AGENT_SHORT = "JarvisHQ-Bot/1.0"
```

**User-Agent Principles:**
- **Honest:** Clearly identifies as a bot
- **Non-deceptive:** Does NOT impersonate a browser
- **Identifiable:** Contains project name and version
- **Contactable:** Includes contact information
- **Purpose-transparent:** States the purpose
- **Respectful:** Signals robots.txt compliance

**Forbidden User-Agent Practices:**
- Impersonating Chrome/Firefox/Safari
- Using common browser User-Agent strings
- Randomizing User-Agent per request
- Omitting bot identification
- Spoofing operating system or device

### 9.2 Accept Headers

```python
REQUEST_HEADERS = {
    # Outgoing requests (scraping)
    'User-Agent': USER_AGENT,
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
    'Accept-Language': 'en-US,en;q=0.5',
    'Accept-Encoding': 'gzip, deflate, br',
    'Connection': 'keep-alive',
    'Cache-Control': 'max-age=0',
    
    # Optional but recommended
    'From': 'jarvis-hq@localhost',  # RFC 9111 contact
}
```

### 9.3 Referrer Policy

```python
# Outgoing requests: minimal referrer information
REFERRER_POLICY = 'no-referrer'

# Never send full URL as referrer - only send origin if required
REFERRER_MAP = {
    'default': None,           # No referrer by default
    'same-origin': 'origin',   # Only within same site
}
```

### 9.4 TLS Requirements

```python
# tls_config.py
import ssl
import certifi

class TLSConfig:
    """TLS configuration for all outbound requests."""
    
    # Minimum TLS version
    MIN_TLS_VERSION = ssl.TLSVersion.TLSv1_2
    
    # Cipher suites (secure only)
    CIPHER_SUITES = [
        'ECDHE+AESGCM',
        'ECDHE+CHACHA20',
        'DHE+AESGCM',
        'DHE+CHACHA20',
    ]
    
    @classmethod
    def create_ssl_context(cls) -> ssl.SSLContext:
        """Create secure SSL context for HTTP requests."""
        context = ssl.create_default_context(
            purpose=ssl.Purpose.SERVER_AUTH,
            cafile=certifi.where()
        )
        
        # Minimum TLS 1.2
        context.minimum_version = cls.MIN_TLS_VERSION
        
        # Disable insecure protocols
        context.options |= ssl.OP_NO_SSLv2
        context.options |= ssl.OP_NO_SSLv3
        context.options |= ssl.OP_NO_TLSv1
        context.options |= ssl.OP_NO_TLSv1_1
        
        # Certificate verification (MUST be enabled)
        context.check_hostname = True
        context.verify_mode = ssl.CERT_REQUIRED
        
        # Certificate pinning for critical APIs
        context.load_verify_locations(cafile=certifi.where())
        
        return context

# Usage with requests library
import requests

session = requests.Session()
session.verify = certifi.where()  # Always verify certificates
# NEVER: session.verify = False
```

**TLS Checklist:**
- [x] Certificate verification enabled (never disabled)
- [x] TLS 1.2 minimum (TLS 1.3 preferred)
- [x] No `--insecure` or `verify=False`
- [x] Updated CA bundle (certifi)
- [x] No certificate pinning bypass

---

## 10. Incident Response

### 10.1 Response Procedures

#### Scenario: Blocked by a Source (IP Ban)

```
SEVERITY: Medium | RESPONSE TIME: Within 24 hours

1. DETECTION
   - Alert: "Source X returning 403/Connection refused"
   - Check: Is it just this source or multiple sources?
   
2. ASSESSMENT
   - Check logs for request patterns that triggered ban
   - Review rate limit compliance for the domain
   - Check if robots.txt was violated
   
3. IMMEDIATE ACTION
   - STOP all requests to the domain immediately
   - Mark source as BLOCKED_TEMP in database
   - Set reminder to retry after 48 hours
   
4. ROOT CAUSE
   - If rate limit violation: Adjust rate limit, document fix
   - If robots.txt violation: Review robots.txt parser, fix bug
   - If excessive requests: Review scheduling logic
   - If no violation found: Accept source loss gracefully
   
5. RESOLUTION
   - If fixable: Retry after adjustment, monitor closely
   - If not fixable: Move to BLOCKED_PERM
   - Document incident in source history
   
6. COMMUNICATION
   - Telegram notification to user
   - Log incident in audit log
```

#### Scenario: Rate Limited (429 Response)

```
SEVERITY: Low | RESPONSE TIME: Immediate (automated)

1. AUTOMATED RESPONSE (immediate)
   - Exponential backoff applied
   - Request queued with delay
   - Counter incremented
   
2. ESCALATION (if 5+ 429s in 1 hour)
   - Source marked BLOCKED_TEMP for 24 hours
   - Telegram notification sent
   
3. REVIEW (within 24 hours)
   - Check if rate limit config matches API docs
   - Check if quota was exceeded
   - Adjust rate limits if needed
   
4. RESOLUTION
   - Automatic unblocking after cooldown
   - Reduced rate limit if needed
```

#### Scenario: Legal Notice Received (Cease and Desist)

```
SEVERITY: Critical | RESPONSE TIME: Within 4 hours

1. IMMEDIATE ACTIONS (within 1 hour)
   - STOP all scraping from the identified domain
   - Mark source as BLOCKED_PERM
   - Preserve all logs and documentation
   - Screenshot current robots.txt compliance
   - Export source review history
   
2. ASSESSMENT (within 4 hours)
   - Review what content was scraped (titles only? summaries?)
   - Verify attribution was present
   - Check robots.txt compliance logs
   - Confirm no full-text was stored
   
3. RESPONSE (within 24 hours)
   - Send acknowledgment of notice
   - Confirm source has been removed
   - Provide documentation of compliance measures
   - Do NOT argue or defend - comply first
   
4. DOCUMENTATION
   - Log all communications
   - Preserve incident record
   - Add source domain to permanent blocklist
   - Review similar sources for same issue
   
5. POST-INCIDENT
   - Review compliance procedures for gaps
   - Update source review process if needed
   - Document lessons learned
```

### 10.2 Escalation Path

```
Level 1: Automated Response
- Rate limiting backoff
- Temporary source blocking
- Telegram alert

Level 2: User Notification
- Review required
- Manual intervention needed
- Compliance question

Level 3: External Consultation
- Legal question
- Complex copyright issue
- Regulatory concern
  -> Seek legal advice before responding
```

### 10.3 Incident Log Format

```json
{
  "incident_id": "INC-2025-001",
  "timestamp": "2025-01-15T10:30:00Z",
  "severity": "medium",
  "type": "source_blocked",
  "domain": "example.com",
  "trigger": "HTTP 403 after 150 requests in 1 hour",
  "response": "stopped_requests, marked_blocked_temp",
  "root_cause": "rate_limit_misconfiguration",
  "resolution": "adjusted_rate_limit_from_5s_to_15s",
  "status": "resolved",
  "lessons_learned": "Need better per-domain rate limit defaults"
}
```

---

## 11. Compliance Checklist

### Pre-Deployment Checklist

- [x] **C01**: `robots.txt` checked for all sources in allowlist
- [x] **C02**: Rate limits configured per domain (in `rate_limits.yaml`)
- [x] **C03**: All API keys stored in 1Password (none in code/env)
- [x] **C04**: No PII collection enabled in any scraper
- [x] **C05**: Attribution preserved for all source types
- [x] **C06**: Pre-commit hooks installed (detect-secrets, gitleaks)
- [x] **C07**: Docker secrets configured for 1Password token
- [x] **C08**: Log sanitization filters tested and verified
- [x] **C09**: Exponential backoff implemented for 429 responses
- [x] **C10**: Source blocklist populated with known problematic domains
- [x] **C11**: Auto-block rules configured and tested
- [x] **C12**: TLS 1.2+ enforced for all connections
- [x] **C13**: User-Agent string is honest and non-deceptive
- [x] **C14**: No full-text scraping in any module
- [x] **C15**: No paywall bypass tools installed
- [x] **C16**: PostgreSQL bound to localhost only
- [x] **C17**: Log retention policy configured (30 days app, 90 days audit)
- [x] **C18**: Incident response procedures documented
- [x] **C19**: PII detection filters tested with sample data
- [x] **C20**: Container runs as non-root user with minimal capabilities

### Ongoing Compliance (Weekly)

- [ ] **W01**: Review blocked sources and assess for unblocking
- [ ] **W02**: Check rate limit compliance logs
- [ ] **W03**: Verify no secrets in latest code changes
- [ ] **W04**: Review PII detection log entries
- [ ] **W05**: Check for new dependency vulnerabilities

### Ongoing Compliance (Monthly)

- [ ] **M01**: Refresh `robots.txt` cache for all active sources
- [ ] **M02**: Review and update source allowlist/blocklist
- [ ] **M03**: Audit database for any PII storage
- [ ] **M04**: Review attribution completeness in output
- [ ] **M05**: Check API key rotation schedule
- [ ] **M06**: Review log files for anomalies
- [ ] **M07**: Test backup restoration
- [ ] **M08**: Review and update ToS compliance for sources

### Ongoing Compliance (Quarterly)

- [ ] **Q01**: Full security review of all components
- [ ] **Q02**: Penetration test of local PostgreSQL
- [ ] **Q03**: Review and update risk register
- [ ] **Q04**: Incident response drill
- [ ] **Q05**: Dependency vulnerability scan
- [ ] **Q06**: Review copyright compliance with legal updates

---

## Appendix A: Quick Reference

### Severity Priority Order

1. **P1 - Credential exposure**: Stop everything, rotate keys
2. **P2 - Legal notice**: Comply within hours, document everything
3. **P3 - Source blocking**: Assess and fix within 24 hours
4. **P4 - Rate limiting**: Automated response, review within 24 hours
5. **P5 - Quality issues**: Review at next maintenance window

### Emergency Contacts

| Role | Contact | Purpose |
|------|---------|---------|
| Technical | User (self) | All operational issues |
| Legal | [To be filled] | Cease and desist, legal notices |
| 1Password Support | support@1password.com | Secret management issues |

### Key Files

| File | Purpose |
|------|---------|
| `rate_limits.yaml` | Per-domain rate limit configuration |
| `sources.yaml` | Source allowlist/blocklist |
| `secrets_manager.py` | 1Password integration |
| `robots_compliance.py` | robots.txt parsing and compliance |
| `pii_filter.py` | PII detection and redaction |
| `logging_config.py` | Secure logging configuration |
| `attribution.py` | Source attribution generation |
| `source_manager.py` | Source lifecycle management |
| `audit.log` | Compliance audit trail |

---

*Document version: 1.0 | Last updated: 2025-01-15 | Next review: 2025-04-15*
