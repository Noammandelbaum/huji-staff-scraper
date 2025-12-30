---
name: huji-staff-scraper
description: Automated scraper for Hebrew University staff directory with AWS deployment and monthly email updates
status: backlog
created: 2025-12-30T08:27:26Z
---

# PRD: HUJI Staff Scraper

## Executive Summary

Build an automated web scraping system to collect and maintain a comprehensive database of all Hebrew University of Jerusalem (HUJI) staff members. The system will run monthly on AWS infrastructure, detect changes (additions/removals), and send email notifications with updated CSV exports.

**Target Scale:** ~2,700+ employees across 7 faculties, 315 departments, and ~100 research centers.

## Problem Statement

### What problem are we solving?
There is no single, unified staff directory for the Hebrew University. Staff information is scattered across:
- Individual faculty websites (7 faculties)
- Department-specific pages (315 departments)
- OpenScholar personal profiles
- Research center websites

### Why is this important now?
- Manual collection is impractical given the scale (2,700+ people)
- Staff changes frequently (new hires, departures, role changes)
- Need consistent, up-to-date contact information for outreach
- No existing automated solution captures all staff types

## User Stories

### Primary User: Administrator
**As an** administrator managing outreach campaigns,
**I want** a complete, up-to-date list of all HUJI staff,
**So that** I can contact relevant personnel without missing anyone.

**Acceptance Criteria:**
- CSV contains all discoverable staff members
- Data includes: name, department, faculty, email, website, phone (when available)
- Updates arrive monthly via email
- Changes (added/removed) are clearly highlighted

### Secondary User: Data Consumer
**As a** data consumer,
**I want** historical versions of the staff list,
**So that** I can track changes over time and perform trend analysis.

**Acceptance Criteria:**
- All historical versions are preserved in S3
- Version naming includes timestamp
- Easy to compare between versions

## Requirements

### Functional Requirements

#### FR1: Web Scraping Engine
- Scrape all 7 HUJI faculties and their subdivisions
- Handle different page structures per department
- Extract: name, title/role, department, faculty, email, phone, personal website
- Respect rate limits and implement polite scraping (delays between requests)
- Handle Hebrew and English content

#### FR2: Data Sources Coverage

**Faculties (7):**
| Faculty | Campus | Estimated Staff |
|---------|--------|-----------------|
| Humanities | Mount Scopus | ~200 |
| Social Sciences | Mount Scopus | ~150 |
| Law | Mount Scopus | ~50 |
| Science | Givat Ram | ~300 |
| Medicine | Ein Kerem | ~400 |
| Dental Medicine | Ein Kerem | ~50 |
| Agriculture, Food & Environment | Rehovot | ~150 |

**Schools & Institutes:**
- School of Social Work
- School of Public Policy
- School of Education
- School of Business Administration
- Rothberg International School
- Koret School of Veterinary Medicine
- School of Nutritional Sciences
- Various research centers (~100)

**Staff Types:**
- Tenured academic faculty (~1,200)
- Administrative staff (~1,500)
- Research staff
- Teaching assistants
- Postdocs and visiting scholars

#### FR3: Change Detection
- Compare current scrape with previous version
- Identify: new staff, removed staff, changed details
- Generate change summary for email notification

#### FR4: Output Generation
- CSV format with columns: Name, Department, Faculty, Email, Website, Phone
- UTF-8 encoding (Hebrew support)
- Consistent formatting across all sources

#### FR5: Notification System
- Send email to distribution list on completion
- Include: summary of changes, attached CSV, link to S3
- Send alert emails on scraping failures

#### FR6: Scheduling
- Run automatically on 1st of each month at 10:00 AM Israel time
- Support manual trigger for on-demand runs

### Non-Functional Requirements

#### NFR1: Reliability
- Retry failed requests up to 3 times with exponential backoff
- Continue scraping other sources if one fails
- Log all errors with context for debugging
- Alert on partial failures (some sources failed)

#### NFR2: Performance
- Complete full scrape within 2 hours
- Handle rate limiting gracefully
- Use concurrent requests where safe (different domains)

#### NFR3: Maintainability
- Modular parser architecture (easy to add/update per-site parsers)
- Configuration-driven source list
- Comprehensive logging

#### NFR4: Data Quality
- Deduplicate by email address
- Validate email format
- Handle missing fields gracefully
- Flag suspicious data for review

#### NFR5: Security
- Store AWS credentials securely (Secrets Manager)
- No sensitive data in logs
- HTTPS for all requests

## Technical Architecture

### Components

```
+------------------+     +------------------+     +------------------+
|   EventBridge    | --> |   Lambda/ECS     | --> |       S3         |
|   (Scheduler)    |     |   (Scraper)      |     |   (Storage)      |
+------------------+     +------------------+     +------------------+
                                |                        |
                                v                        v
                         +------------------+     +------------------+
                         |       SES        |     |   CloudWatch     |
                         |   (Email)        |     |   (Logs/Alerts)  |
                         +------------------+     +------------------+
```

### Technology Stack
- **Language:** Python 3.11+
- **Scraping:** BeautifulSoup4, requests, aiohttp
- **AWS Services:** Lambda or ECS, S3, SES, EventBridge, Secrets Manager, CloudWatch
- **IaC:** Terraform
- **Testing:** pytest

### Data Flow
1. EventBridge triggers scraper on schedule
2. Scraper fetches all configured sources
3. Parser modules extract staff data per source type
4. Deduplication and validation
5. Compare with previous version from S3
6. Generate CSV and change report
7. Upload to S3 with timestamp
8. Send email via SES
9. Log results to CloudWatch

### File Structure
```
huji-staff-scraper/
├── src/
│   ├── scraper/
│   │   ├── __init__.py
│   │   ├── base.py           # Base scraper class
│   │   ├── client.py         # HTTP client with retry logic
│   │   └── sources.py        # Source URL configuration
│   ├── parsers/
│   │   ├── __init__.py
│   │   ├── base.py           # Base parser interface
│   │   ├── faculty.py        # Generic faculty page parser
│   │   ├── openscholar.py    # OpenScholar profile parser
│   │   └── department.py     # Department-specific parsers
│   ├── processors/
│   │   ├── __init__.py
│   │   ├── dedup.py          # Deduplication logic
│   │   ├── validator.py      # Data validation
│   │   └── diff.py           # Change detection
│   ├── exporters/
│   │   ├── __init__.py
│   │   ├── csv_export.py     # CSV generation
│   │   └── email.py          # Email notification
│   ├── config/
│   │   ├── __init__.py
│   │   ├── settings.py       # Configuration management
│   │   └── sources.yaml      # Source URLs and parser mappings
│   └── main.py               # Entry point
├── aws/
│   └── terraform/
│       ├── main.tf
│       ├── variables.tf
│       ├── lambda.tf         # Or ecs.tf
│       ├── s3.tf
│       ├── ses.tf
│       ├── eventbridge.tf
│       └── outputs.tf
├── tests/
│   ├── test_parsers/
│   ├── test_processors/
│   └── fixtures/
├── data/
│   └── .gitkeep
├── requirements.txt
├── pyproject.toml
├── Dockerfile
└── README.md
```

## Success Criteria

| Metric | Target |
|--------|--------|
| Staff coverage | >90% of discoverable staff |
| Data completeness | Name + Department for 100%, Email for >80% |
| Scrape success rate | >95% of sources per run |
| Monthly uptime | 100% scheduled runs execute |
| Email delivery | 100% of notifications sent |
| Change detection accuracy | >99% (no false positives) |

## Constraints & Assumptions

### Constraints
- Website structure may change without notice (requires parser updates)
- Some staff may not be listed on any public page
- Rate limiting must be respected to avoid IP blocks
- Hebrew text requires proper encoding handling

### Assumptions
- Staff information on public websites is intended to be public
- Email addresses follow standard HUJI format when not explicitly listed
- Department websites will remain accessible
- AWS account with SES production access is available

## Out of Scope

- Real-time updates (only monthly batch)
- Staff photos or profile images
- Research publications or CV data
- Internal/intranet staff directories
- Authentication-protected pages
- Mobile app or web UI
- API for external consumers
- Integration with HR systems

## Dependencies

### External
- HUJI website availability and structure stability
- AWS services (Lambda/ECS, S3, SES, EventBridge)
- Python packages: BeautifulSoup4, requests, boto3

### Internal
- AWS account setup and permissions
- SES email verification for sender domain
- Distribution list email addresses

## Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|------------|------------|
| Website blocks scraper | High | Medium | Proper headers, rate limiting, proxy rotation |
| Website structure changes | Medium | High | Modular parsers, alerts on parsing failures |
| Incomplete data | Medium | Medium | Multiple source cross-reference, manual review |
| AWS cost overrun | Low | Low | Cost alerts, efficient Lambda sizing |

## Milestones

### Phase 1: Core Scraper
- Basic scraping for 3 major faculties
- CSV output
- Local execution

### Phase 2: Full Coverage
- All faculties and departments
- Deduplication and validation
- Change detection

### Phase 3: AWS Deployment
- Lambda/ECS setup
- S3 storage
- EventBridge scheduling

### Phase 4: Notifications
- SES email integration
- Error alerting
- Distribution list support

### Phase 5: Hardening
- Comprehensive logging
- Retry logic
- Parser resilience

## Open Questions

1. Should we attempt to scrape password-protected internal directories?
2. What is the acceptable delay between identifying a website structure change and updating the parser?
3. Should we include emeritus faculty?
4. How should we handle staff who appear in multiple departments?
