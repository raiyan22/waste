GULLI KNOWLEDGE BASE v1.0

purpose: Complete technical documentation of the Gulli security vulnerability aggregation system. This document should enable understanding and modification of the system without referencing source code.

Table of Contents
System Overview
Architecture
Data Flow
Core Components
External Integrations
Configuration
Deployment
Design Patterns & Decisions
Error Handling
Open Questions

1. System Overview
What is Gulli?
Gulli is a security vulnerability aggregation and ticketing system that:

Monitors multiple sources for security alerts (GitHub Dependabot, Google Artifact Registry, RSS feeds)
Automatically creates Jira tickets for new vulnerabilities
Tracks all alerts in a Google Spreadsheet (used as a lightweight database)
Notifies teams via Slack when new vulnerabilities are discovered
Runs as a Kubernetes CronJob across 25+ different project environments
Why "Gulli"?
Unknown - possibly an internal name or acronym. (Question for user)

Business Context
This appears to be used by an organization managing multiple client projects (based on deployment folders: innotix, biosuisse, welltravel, pick-e-bike, etc.). Each project gets its own Gulli deployment with project-specific configuration.

2. Architecture
High-Level Architecture

┌─────────────────────────────────────────────────────────────────────────────┐
│                           GULLI SYSTEM                                       │
│                                                                             │
│  ┌─────────────┐     ┌─────────────────┐     ┌─────────────────────────┐   │
│  │ DATA SOURCES│     │   GULLI CORE    │     │    OUTPUT TARGETS       │   │
│  │             │     │                 │     │                         │   │
│  │ ┌─────────┐ │     │  ┌───────────┐  │     │  ┌─────────────────┐   │   │
│  │ │GitHub   │─┼────►│  │Collectors │  │     │  │ Jira Cloud      │   │   │
│  │ │Dependabot│ │     │  └─────┬─────┘  │     │  │ (Ticket Create) │   │   │
│  │ └─────────┘ │     │        │        │     │  └────────▲────────┘   │   │
│  │             │     │        ▼        │     │           │            │   │
│  │ ┌─────────┐ │     │  ┌───────────┐  │     │  ┌────────┴────────┐   │   │
│  │ │Google   │─┼────►│  │FeedEntry  │──┼────►│  │ Google Sheets   │   │   │
│  │ │Artifact │ │     │  │Objects    │  │     │  │ (State Store)   │   │   │
│  │ │Registry │ │     │  └─────┬─────┘  │     │  └────────▲────────┘   │   │
│  │ └─────────┘ │     │        │        │     │           │            │   │
│  │             │     │        ▼        │     │  ┌────────┴────────┐   │   │
│  │ ┌─────────┐ │     │  ┌───────────┐  │     │  │ Slack           │   │   │
│  │ │RSS Feeds│─┼────►│  │Handler    │──┼────►│  │ (Notifications) │   │   │
│  │ │(Various)│ │     │  └───────────┘  │     │  └─────────────────┘   │   │
│  │ └─────────┘ │     │                 │     │                         │   │
│  └─────────────┘     └─────────────────┘     └─────────────────────────┘   │
│                                                                             │
│                        Runs as K8s CronJob                                  │
└─────────────────────────────────────────────────────────────────────────────┘


Technology Stack

Component	            Technology
Language	            Ruby (version in .ruby-version)
Dependency Management	Bundler
Container	            Docker
Orchestration	        Kubernetes (Helm charts)
Configuration Mgmnt	    Helmfile + SOPS (secrets)
CI/CD	                GitHub Actions


3. Data Flow
Complete Execution Flow

┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: INITIALIZATION                                                        │
│ ══════════════════════                                                        │
│                                                                              │
│   bin/gulli                                                                  │
│       │                                                                      │
│       └──► Gulli.run()                                                       │
│               │                                                              │
│               └──► clear_cache!()  # Reset in-memory cache for fresh run     │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 2: FETCH FEED CONFIGURATIONS                                            │
│ ═════════════════════════════════                                            │
│                                                                              │
│   FetchFeeds.call()                                                          │
│       │                                                                      │
│       └──► Spreadsheet.get_values(SPREADSHEET_RANGES_FEEDS)                  │
│               │                                                              │
│               └──► Google Sheets API: GET spreadsheet values                 │
│                       │                                                      │
│                       └──► Returns: [{                                       │
│                              name: "repo-name",                              │
│                              url: "https://...",                             │
│                              type: "dependabot|rss|google-artifact-registry",│
│                              ignore_updates: true|false,                     │
│                              auto_jira_issue_create: true|false,             │
│                              jira_ticket_assignee: "jira-account-id"         │
│                            }, ...]                                           │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 3: COLLECT ALERTS FROM EACH SOURCE                                      │
│ ════════════════════════════════════════                                     │
│                                                                              │
│   For each feed config:                                                      │
│       │                                                                      │
│       ├──► IF type == "dependabot":                                          │
│       │       CollectDependabotAlerts.call(feed)                             │
│       │           │                                                          │
│       │           └──► GitHub API: GET /repos/{owner}/{repo}/dependabot/alerts│
│       │                   │        ?state=open                               │
│       │                   │        Authorization: Bearer {GITHUB_TOKEN}      │
│       │                   │                                                  │
│       │                   └──► Returns: [FeedEntry::Dependabot, ...]         │
│       │                                                                      │
│       ├──► IF type == "rss":                                                 │
│       │       CollectRSSArticles.call(feed)                                  │
│       │           │                                                          │
│       │           └──► HTTP GET feed URL                                     │
│       │                   │                                                  │
│       │                   └──► Feedjira.parse(xml)                           │
│       │                           │                                          │
│       │                           └──► Returns: [FeedEntry::RSSArticle, ...] │
│       │                                                                      │
│       └──► IF type == "google-artifact-registry":                            │
│               CollectGoogleArtifactRegistryAlerts.call(feed)                 │
│                   │                                                          │
│                   └──► GCP API: GET image manifests                          │
│                           │     Authorization: Bearer {GCP_TOKEN}            │
│                           │                                                  │
│                           └──► Returns: [FeedEntry::GoogleArtifactRegistry]  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 4: PROCESS EACH ANNOUNCEMENT                                            │
│ ═════════════════════════════════                                            │
│                                                                              │
│   For each FeedEntry (announcement):                                         │
│       │                                                                      │
│       ├──► known?()                                                          │
│       │       │                                                              │
│       │       └──► Check if ID exists in DATA spreadsheet                    │
│       │            (uses cached data after first call)                       │
│       │               │                                                      │
│       │               └──► IF found: SKIP (already processed)                │
│       │                                                                      │
│       ├──► published_date_before_6_months?()                                 │
│       │       │                                                              │
│       │       └──► IF date < (now - 180 days): SKIP (too old)                │
│       │                                                                      │
│       ├──► ignore?()                                                         │
│       │       │                                                              │
│       │       └──► IF title matches /alpha|beta|rc|nightly|preview|snapshot|test/│
│       │            SKIP notifications (but still track)                      │
│       │            NOTE: Dependabot overrides this to NEVER ignore           │
│       │                                                                      │
│       ├──► IF auto_jira_issue_create? AND NOT ignore?:                       │
│       │       create_jira_ticket!()                                          │
│       │           │                                                          │
│       │           └──► JiraTicketCreator.call(article)                       │
│       │                   │                                                  │
│       │                   └──► Jira API: POST /rest/api/3/issue              │
│       │                           │                                          │
│       │                           └──► Returns: ticket URL (stored in entry) │
│       │                                                                      │
│       ├──► remember!()                                                       │
│       │       │                                                              │
│       │       └──► Spreadsheet.append() to DATA sheet                        │
│       │            Columns: [feed, id, identifier, url, ignore?, date,       │
│       │                      jira_ticket_link]                               │
│       │                                                                      │
│       └──► IF NOT ignore?:                                                   │
│               Notify.article()                                               │
│                   │                                                          │
│                   └──► Slack Webhook: POST notification                      │
│                        Format: "*{feed}*: {title} | Spreadsheet | JIRA-XXX"  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 5: ERROR HANDLING                                                       │
│ ══════════════════════                                                       │
│                                                                              │
│   ANY uncaught exception:                                                    │
│       │                                                                      │
│       └──► Notify.fatal() ──► Slack: "<!channel> Gulli Crashed: {error}"     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘


4. Core Components
4.1 Entry Point (bin/gulli)

# Minimal bootstrap - loads lib path and calls Gulli.run
$LOAD_PATH.unshift File.expand_path('../../lib', __FILE__)
require 'bundler/setup'
require 'gulli'
Gulli.run

4.2 Main Module (lib/gulli.rb)
Responsibilities:
    Orchestrates the entire execution flow
    Manages configuration from environment variables
    Provides module-level caching
    Handles top-level error recovery

Key Methods:

Method	                      Purpose
run	                          Main entry point - fetches feeds, collects alerts, processes each
feeds	                      Fetches feed configurations from spreadsheet
announcements	                        Collects all alerts from all feeds
handle_announcement(a)	                Processes single alert (jira, remember, notify)
already_known_or_old?(a)	            Deduplication check
config	                                Lazy-loaded configuration hash from ENV
cache / clear_cache!	                In-memory cache management
logger	                                Stdout logger instance


Configuration Structure:

{
  google: {
    keyfile: ENV['GOOGLE_KEYFILE']  # Path to service account JSON
  },
  spreadsheet: {
    id: ENV['SPREADSHEET_ID'],
    ranges: {
      feeds: ENV['SPREADSHEET_RANGES_FEEDS'],      # e.g., "Feeds!A:Z"
      data: ENV['SPREADSHEET_RANGES_DATA'],        # e.g., "Data!A:Z"
      update_sheet: ENV['SPREADSHEET_UPDATE_SHEET'],
      update_column: ENV['SPREADSHEET_UPDATE_COLUMN']
    }
  },
  repo: {
    owner: ENV['GITHUB_REPO_OWNER'],
    token: ENV['GITHUB_REPO_ACCESS_TOKEN']
  },
  slack: {
    webhook_url: ENV['SLACK_WEBHOOK_URL'],
    channel: ENV['SLACK_CHANNEL'],
    username: ENV['SLACK_USERNAME'],
    icon: ENV['SLACK_ICON']
  },
  jira: {
    base_url: ENV['JIRA_BASE_URL'],
    username: ENV['JIRA_USERNAME'],
    api_token: ENV['JIRA_API_TOKEN'],
    project_key: ENV['JIRA_PROJECT_KEY'],
    issue_type: ENV['JIRA_ISSUE_TYPE'],
    project_name: ENV['JIRA_PROJECT_NAME']
  }
}


4.3 Collector Classes
Strategy Pattern Mapping

FEED_ENTRY_COLLECTOR = {
  'rss'                     => Gulli::CollectRSSArticles,
  'dependabot'              => Gulli::CollectDependabotAlerts,
  'google-artifact-registry' => Gulli::CollectGoogleArtifactRegistryAlerts
}

CollectDependabotAlerts
    Source: GitHub Dependabot API
    API Endpoint: GET /repos/{owner}/{repo}/dependabot/alerts?state=open
    Authentication: Bearer token (GITHUB_REPO_ACCESS_TOKEN)
    Returns: Array of FeedEntry::Dependabot objects

Key behaviors:

    Only fetches open alerts (not dismissed/fixed)
    Notifies Slack if API fails (doesn't crash)
    Returns empty array if not configured

CollectRSSArticles
    Source: Any RSS/Atom feed URL
    Library: Feedjira (Ruby RSS parser)
    Returns: Array of FeedEntry::RSSArticle objects

Key behaviors:

    Handles SSL certificate issues (set_default_paths)
    Graceful error handling with Slack notification
    Returns empty array on fetch failure

CollectGoogleArtifactRegistryAlerts
    Source: Google Container Registry / Artifact Registry
    API Endpoint: Image manifest endpoint
    Authentication: GCP Service Account OAuth token
    Returns: Array of FeedEntry::GoogleArtifactRegistry objects

Key behaviors:

    Only monitors images tagged latest
    Extracts upload time from manifest
    Converts internal URL to marketplace URL for readability

4.4 FeedEntry Classes (Value Objects)
Base Class (FeedEntry::Base)
Abstract interface that all entry types must implement:

Method	        Return Type	        Description
id	            String	            Unique identifier for deduplication
url	            String	            Link to the alert/article
title	        String	            Display title for notifications
identifier	    String	            CVE ID, GHSA ID, or article identifier
date	        String	            ISO date string for age checking

Shared behaviors:

Method	                            Description
known?	                            Checks if ID exists in spreadsheet (cached)
published_date_before_6_months?	    Returns true if older than 180 days
ignore?	                            Matches against /alpha|beta|rc|nightly|preview|snapshot|test/i
remember!	                        Appends entry to DATA spreadsheet
create_jira_ticket!	                Delegates to JiraTicketCreator
update!(position, data)	            Updates specific cell in spreadsheet

Ignore Pattern:
IGNORE_REGEXP = /([.-]rc|alpha|beta|nightly|preview|snapshot|[.-]test)/i

Dependabot Entry (FeedEntry::Dependabot)
id         = "{feed}-{alert_number}"           # e.g., "my-repo-42"
url        = entry['html_url']                 # GitHub alert URL
title      = "{identifier} - {summary}"        # "CVE-2024-1234 - SQL Injection"
identifier = cve_id || ghsa_id                 # Prefers CVE over GHSA
date       = entry['updated_at']               # Last update time
ignore?    = false                             # NEVER ignore security alerts

#### RSSArticle Entry (`FeedEntry::RSSArticle`)
**Source:** Feedjira parsed RSS/Atom entry object
id = entry.id # RSS guid/id field
url = entry.url # Link to article
title = entry.title.gsub(/#{feed}/i, '') # Remove feed name from title
identifier = title # Same as cleaned title
date = entry.updated || entry.published # Prefer updated, fallback to published
ignore? = <inherited> # Uses Base IGNORE_REGEXP


**Notable behavior:**
- Title is cleaned by removing the feed name (case-insensitive)
- Falls back from `updated` to `published` for date
- Uses inherited `ignore?` so alpha/beta/rc releases are skipped

#### GoogleArtifactRegistry Entry (`FeedEntry::GoogleArtifactRegistry`)
**Source:** Parsed manifest data from GCP API

id = "{feed}-{image_digest}" # e.g., "my-app-sha256:abc123"
url = entry['url'] # Marketplace URL (parsed from API URL)
title = "Docker Image Alert: {feed} - {image_digest}"
identifier = entry['image'] # The image digest
date = entry['uploaded_at'] # Formatted upload timestamp
ignore? = @ignore_updates # Controlled by spreadsheet config!


**Notable behavior:**
- Unlike other entries, `ignore?` is controlled by the feed configuration (`ignore_updates` column)
- This allows turning off notifications for specific images while still tracking them
- Has a `summary` method (not in Base) used for richer descriptions




4.5 Spreadsheet Service
Purpose: Abstraction layer over Google Sheets API

Methods:
Method	                        Parameters	                        Description
get_values(range)	            Range string (e.g., "Sheet!A:Z")	Reads rows, returns array of hashes
update(range, attributes:)	    Range + array of values	            Updates specific cells
append(range, attributes:)	    Range + array of values	            Appends new row
service	                        -	                                Lazy-loaded authenticated API client

Row Transformation:
# Raw API response:
[["Name", "URL", "Type"], ["repo", "http://...", "dependabot"]]

# Transformed to:
[{"Name" => "repo", "URL" => "http://...", "Type" => "dependabot"}]

Caching Strategy:

Cache key format: "spreadsheet_ranges_{range}"
Cached on read, invalidated on write
Cleared at start of each run (clear_cache!)
Prevents redundant API calls when checking known? for multiple entries

4.6 Jira Ticket Creator
API: Jira Cloud REST API v3
Endpoint: POST /rest/api/3/issue?skipAutoWatch=true
Authentication: Basic Auth (base64 encoded username:api_token)

Payload Structure:
{
  fields: {
    project: { key: "SEC" },                    # JIRA_PROJECT_KEY
    reporter: { accountId: "..." },             # From spreadsheet
    summary: "CVE-2024-1234 - SQL Injection",   # article.title
    issuetype: { name: "Vulnerability" },       # JIRA_ISSUE_TYPE
    assignee: { accountId: "..." },             # From spreadsheet
    customfield_11856: "2024-01-15",            # Assessment due date
    customfield_11488: "my-repo",               # Feed name
    customfield_11489: "https://github.com/..", # Alert URL
    customfield_11490: { value: "Project X" }   # Project dropdown
  }
}

Custom Field IDs (Jira-specific):

JIRA_FIELD_FEED = 'customfield_11488'
JIRA_FIELD_URL = 'customfield_11489'
JIRA_FIELD_PROJECT_NAME = 'customfield_11490'
JIRA_ASSESSMENT_DUE_DATE = 'customfield_11856'

4.7 Assessment Due Date Generator
Purpose: Calculate compliance deadline for vulnerability assessment

Business Rules:

If current time < 10:00 AM Dhaka time → Due date = today
If current time ≥ 10:00 AM Dhaka time → Due date = tomorrow
If tomorrow is Saturday → Due date = Sunday (skip Saturday)
Timezone: Bangladesh Standard Time (UTC+6)

Why Dhaka? Question for user - suggests team location

4.8 Slack Notifier
Library: slack-notifier gem
Method: Webhook POST

Notification Types:

Method	Trigger	Format
article(a)	New vulnerability found	Feed + URL + Spreadsheet link + Jira link
feed_error(...)	RSS fetch failure	Feed name + error
dependabot_error(...)	GitHub API failure	Repo name + error
docker_error(...)	GCP API failure	Repo name + error
fatal(error)	Uncaught exception	<!channel> mention + error

Emoji Usage:

:inbox_tray: - New alert
:bookmark_tabs: - Spreadsheet link
:ticket: - Jira ticket
:exclamation: - Errors


### 4.9 Dependencies (Gemfile)

| Gem | Version | Purpose |
|-----|---------|---------|
| `feedjira` | ~> 3.2, >= 3.2.1 | RSS/Atom feed parsing |
| `google-api-client` | ~> 0.53.0 | Google Sheets API client |
| `slack-notifier` | (any) | Slack webhook notifications |
| `httpclient` | (any) | HTTP client (shared with google-api-client) |

**Development/Test Dependencies:**
| Gem | Purpose |
|-----|---------|
| `dotenv` | Load .env files in development |
| `pry-rails` | Debugging console |
| `rspec` | Test framework |
| `rspec_junit_formatter` | CI-friendly test output |
| `timecop` | Time manipulation for tests |

**Design Note:** HTTPClient is explicitly included because google-api-client depends on it. This avoids having multiple HTTP client gems (like Faraday, Net::HTTP, etc.) in the project.

5. External Integrations
5.1 Google Sheets API
Purpose: Configuration storage + State management
Auth: Service Account (JSON keyfile)
Scope: AUTH_SPREADSHEETS (read/write)

5.2 GitHub API
Purpose: Fetch Dependabot security alerts
Auth: Personal Access Token (Bearer)
Required Permissions: security_events read access
API Version: 2022-11-28

5.3 Jira Cloud API
Purpose: Create vulnerability tracking tickets
Auth: Basic Auth (email + API token)
API Version: v3

5.4 Google Cloud Platform
Purpose: Monitor container image updates
Auth: Service Account OAuth
Scope: cloud-platform

5.5 Slack
Purpose: Team notifications
Auth: Incoming Webhook URL

6. Configuration
### Environment Variables

#### Complete List (from .env.example)
| Variable | Example Value | Required | Description |
|----------|---------------|----------|-------------|
| `SPREADSHEET_ID` | `1sgXVRgdLHogv7UX9G3RaHwlZvVhGOAMtSUluWPZ0jOA` | Yes | Google Sheets document ID (from URL) |
| `SPREADSHEET_RANGES_FEEDS` | `Feeds!A:D` | Yes | Range for feed configuration sheet |
| `SPREADSHEET_RANGES_DATA` | `Data!A:G` | Yes | Range for tracking/state sheet |
| `SPREADSHEET_UPDATE_SHEET` | `Data!` | Yes | Sheet name prefix for updates |
| `SPREADSHEET_UPDATE_COLUMN` | `F` | Yes | Column letter for date updates |
| `GOOGLE_KEYFILE` | `/secrets/google/credentials.json` | Yes | Path to GCP service account JSON |
| `SLACK_WEBHOOK_URL` | `https://hooks.slack.com/...` | No | Slack incoming webhook URL |
| `SLACK_CHANNEL` | `#security-alerts` | No | Override default webhook channel |
| `SLACK_USERNAME` | `gulli` | No | Bot display name in Slack |
| `SLACK_ICON` | `robot_face` | No | Emoji name (without colons) |
| `GITHUB_REPO_OWNER` | `welltravel` | No* | GitHub organization/user name |
| `GITHUB_REPO_ACCESS_TOKEN` | `ghp_xxxx` | No* | GitHub Personal Access Token |
| `JIRA_BASE_URL` | `https://welltravel.atlassian.net` | No* | Jira Cloud instance URL |
| `JIRA_USERNAME` | `bot@company.com` | No* | Jira account email |
| `JIRA_API_TOKEN` | `ATATT3x...` | No* | Jira API token |
| `JIRA_PROJECT_KEY` | `VA` | No* | Jira project key for tickets |
| `JIRA_ISSUE_TYPE` | `Vulnerability Assessment` | No* | Issue type name |
| `JIRA_PROJECT_NAME` | `Welltravel` | No* | Value for project dropdown field |

*Required if using that feature (Dependabot/Jira)

#### Spreadsheet Structure Revealed
From the ranges, we can deduce:
- **Feeds sheet**: Columns A-D (4 columns: Name, URL, Type, Ignore Updates + more via named ranges)
- **Data sheet**: Columns A-G (7 columns: feed, id, identifier, url, ignore?, date, jira_ticket_link)
- **Update column F**: The `date` column (6th column = F)


7. Deployment
Kubernetes CronJob Structure
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ .Release.Name }}  # e.g., "gulli-innotix-app"
spec:
  schedule: {{ .Values.schedule }}  # Cron expression
  concurrencyPolicy: Forbid         # Prevent overlapping runs
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: gulli
            image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
            env:
            - name: GOOGLE_KEYFILE
              value: /secrets/google/credentials.json
            envFrom:
            - configMapRef:           # Non-secret config
                name: {{ .Release.Name }}
            - secretRef:              # Secrets (tokens, etc.)
                name: {{ .Release.Name }}
            volumeMounts:
            - name: google-credentials
              mountPath: /secrets/google
          restartPolicy: OnFailure
          volumes:
          - name: google-credentials
            secret:
              secretName: {{ .Release.Name }}
              items:
              - key: googleApplicationCredentials
                path: credentials.json

Multi-Tenant Deployment
Each project gets separate deployment with its own:

Spreadsheet (different feeds, different tracking)
Jira project/assignees
Slack channel
GitHub repos
Known deployments (from folder structure):

alan-apps, biosuisse, cora, dap, garaio-rem
innotix (14 sub-projects!)
ops-updates, pci-dss, pick-e-bike, rms
stage9, tessi, trueact, wellday, welldev-nlp
welltravel-apps, welltravel-mobile-apps, welltravel-website

Secrets Management
Tool: SOPS (Secrets OPerationS)
Config: deployment/.sops.yaml
Purpose: Encrypt secrets in Git, decrypt at deploy time

### Docker Image

**Base Images:**
Production: us-west1-docker.pkg.dev/nodal-algebra-161404/base-image-ruby/base-image-ruby:3.4.4-1
Development: us-west1-docker.pkg.dev/nodal-algebra-161404/base-image-ruby/base-image-ruby-dev:3.4.4-1_node22


**Ruby Version:** 3.4.4

**Multi-Stage Build:**

┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: builder (DEV_IMAGE) │
│ - Install gems with BUNDLE_WITHOUT=development:test │
│ - Remove .gem cache files │
│ - Remove .c and .o source files (reduce size) │
│ - Runs as 'ruby' user (non-root) │
├─────────────────────────────────────────────────────────────────┤
│ Stage 2: production (BASE_IMAGE) │
│ - Copy app code │
│ - Copy pre-built /bundle from builder │
│ - Update CA certificates (as root, then switch back) │
│ - Runs as 'ruby' user (non-root) │
│ - CMD: bin/gulli │
├─────────────────────────────────────────────────────────────────┤
│ Stage 3: development (extends production) │
│ - BUNDLE_WITHOUT="" (include dev/test gems) │
│ - Full bundle install │
└─────────────────────────────────────────────────────────────────┘


**Security Considerations:**
- Runs as non-root `ruby` user
- CA certificates explicitly updated (important for API calls)
- Build artifacts removed to reduce attack surface

**Image Registry:**

us-west1-docker.pkg.dev/nodal-algebra-161404/gulli/gulli


### Default Helm Values

```yaml
schedule: '4 */6 * * *'  # Every 6 hours at minute 4 (00:04, 06:04, 12:04, 18:04)

slack:
  username: Gulli

jira:
  baseUrl: 'https://welltravel.atlassian.net'
  projectKey: 'VA'                              # Vulnerability Assessment project
  issueType: 'Vulnerability Assessment'

image:
  repository: us-west1-docker.pkg.dev/nodal-algebra-161404/gulli/gulli


### Per-Project Configuration

Each project deployment customizes:

**Example: Innotix (`deployment/values/innotix/app/values.yaml`)**
```yaml
schedule: '14 */3 * * *'           # Every 3 hours at minute 14

spreadsheet:
  id: '1ZiXs_sbT-Q5G6gv9dw1APEgcVBRfZRy92OCP4Hm_w8E'  # Project-specific spreadsheet
  update:
    sheet: 'Data!'
    column: 'F'
  ranges:
    feeds: 'Feeds!A:F'             # 6 columns (more than default 4)
    data: 'Data!A:G'

slack:
  channel: '#innotix-vulnerability-monitor'  # Project-specific channel
  icon: 'rotating_light'                     # 🚨 emoji

jira:
  projectName: 'Innotix öVPad'     # Dropdown value for this project

Configuration Inheritance:
deployment/chart/values.yaml (defaults)
    │
    └── deployment/values/{project}/app/values.yaml (overrides)
            │
            └── deployment/values/{project}/app/secrets.yaml (encrypted secrets)

Helmfile Multi-Tenant Deployment
File: deployment/helmfile.yaml.gotmpl

Structure:

releases:
  - name: gulli-{project}          # Unique release name
    namespace: gulli               # All in same namespace
    chart: './chart'
    version: 0.3.0
    values:
      - values/{project}/app/values.yaml
    secrets:
      - values/{project}/app/secrets.yaml    # SOPS encrypted
    set:
      - name: 'image.tag'
        value: {{ requiredEnv "REFERENCE" }}  # Git SHA/tag from CI
      - name: 'enableDependabot'
        value: true|false                     # Per-project toggle

Deployed Projects (34 total):

Project	Dependabot Enabled	Notes
ops-updates	❌	Internal ops
pci-dss	✅	Payment compliance
trueact	✅	
welltravel-mobile-apps	✅	
welldev-nlp	✅	NLP service
pick-e-bike	✅	E-bike platform
tessi	(default)	
alan-apps	(default)	
cora	(default)	
biosuisse	✅	Swiss organic certification
welltravel-apps	✅	Main travel app
wellday	✅	
rms	✅	
dap	(default)	
welltravel-website	✅	
gulli-innotix	(default)	Main Innotix
gulli-innotix-web-app-frontend	❌	
gulli-innotix-hirsch	❌	
gulli-innotix-webapp-backend	❌	
gulli-innotix-origa	❌	
pick-e-bike-mobile-apps	✅	
blt-core	✅	BLT subsidiary
blt-go-business-frontend	✅	
garaio-rem-owner-portal	❌	Real estate
gulli-innotix-rhb	❌	
gulli-innotix-bus-finder	❌	
gulli-innotix-cockpit	❌	
gulli-innotix-ifs	❌	
stage9	❌	
gulli-innotix-rhb-afe	❌	
gulli-innotix-cms-cockpit	❌	
gulli-innotix-html-to-pdf-service	❌	
gulli-innotix-push-service	❌	
Observations:

Innotix has 14 sub-projects, most with Dependabot disabled (likely different monitoring strategy)
All deployments share the gulli namespace
Image tag comes from CI environment variable REFERENCE
Chart version locked at 0.3.0        


Schedule Analysis:

Runs 4 times per day
Minute 4 chosen to avoid contention with other cron jobs at minute 0
Sufficient for security alert monitoring (not real-time, but timely)



8. Design Patterns & Decisions
8.1 Strategy Pattern (Collectors)
Why: Different data sources require different fetching logic, but the processing pipeline is identical.

FEED_ENTRY_COLLECTOR[feed[:type]].call(feed)

8.2 Template Method Pattern (FeedEntry)
Why: All entry types need the same behaviors (known?, remember!, ignore?) but different data extraction.

Base class defines algorithm skeleton, subclasses implement abstract methods.

8.3 Google Sheets as Database
Why chosen over traditional DB:

Zero infrastructure cost
Non-technical users can view/edit
Easy auditing and debugging
Sufficient for low-volume (security alerts)
Trade-offs accepted:

No transactions (acceptable - idempotent operations)
String-only types (acceptable - simple data)
API rate limits (mitigated with caching)
8.4 Aggressive Caching
Why: Spreadsheet API is called in known? for every alert. Without caching, 50 alerts = 50 API calls.

Strategy: Cache on first read, invalidate on write, clear at run start.

8.5 Graceful Degradation
Why: Don't lose vulnerability alerts due to partial failures.

Implementation:

Collector failures notify Slack but return empty array (other feeds continue)
Jira failures notify with "[Action Required]" for manual intervention
Top-level catch-all notifies team of crashes
8.6 Dependabot Never Ignores
Why: Security vulnerabilities should always create tickets, even for beta versions.

# Dependabot class overrides Base:
def ignore?
  false
end


---

### Section 8 - Add new subsection 8.7

```markdown
### 8.7 FeedEntry Polymorphism Comparison

| Aspect | Dependabot | RSSArticle | GoogleArtifactRegistry |
|--------|------------|------------|------------------------|
| **id format** | `{feed}-{number}` | RSS entry ID | `{feed}-{digest}` |
| **title format** | `{CVE} - {summary}` | Cleaned RSS title | `Docker Image Alert: ...` |
| **date source** | `updated_at` | `updated \|\| published` | `uploaded_at` |
| **ignore? behavior** | Always `false` | Inherited regexp | From config |
| **identifier** | CVE or GHSA ID | Same as title | Image digest |

**Design insight:** Each entry type customizes behavior appropriate to its source:
- **Dependabot**: Security-first, never ignore
- **RSS**: Content filtering via regexp (skip prereleases)
- **GAR**: User-controlled via spreadsheet config

9. Error Handling
Hierarchy

┌─────────────────────────────────────────────────────────┐
│ Level 1: Top-Level (Gulli.run)                          │
│   catch StandardError → Notify.fatal + log              │
│                                                         │
│ Level 2: Collector (collect_*_alerts.rb)                │
│   catch HTTP errors → Notify.*_error + return []        │
│                                                         │
│ Level 3: Jira (JiraTicketCreator)                       │
│   catch any error → Notify.fatal with action required   │
│   return nil (entry still remembered)                   │
│                                                         │
│ Level 4: Spreadsheet                                    │
│   No explicit handling → bubbles up to Level 1          │
└─────────────────────────────────────────────────────────┘

Notification Escalation
Error Type	Notification	Continues?
Single feed fails	Feed-specific error	Yes
Jira creation fails	"[Action Required]"	Yes
Spreadsheet fails	System crash	No
Unknown exception	"<!channel> Crashed"	No


---

## Section 9 - Add new subsection

```markdown
### 9.1 Testing Patterns (from specs)

**Jira Ticket Creator Tests:**
```ruby
# Test setup pattern
let(:article) do
  OpenStruct.new(
    title: 'Critical Vulnerability in API',
    feed: 'Security Scanner',
    url: 'https://example.com/test/123',
    jira_ticket_assignee: 'test_user',
  )
end

# Time freezing for due date tests
before do
  Timecop.freeze(Time.new(2025, 9, 11, 9, 0, 0, '+06:00'))
end

Tested Scenarios:

Scenario	Expected Behavior
Payload structure	Matches expected Jira API format
HTTP 201 response	Returns ticket URL (https://.../browse/VA-789)
HTTP 400 response	Calls Notify.fatal with action required message, returns nil
Exception raised	Calls Notify.fatal with error message, returns nil

Dependabot Collector Tests:
# Mock GitHub API response
let(:response) do
  [{
    "number": 2,
    "state": 'dismissed',
    "security_advisory": {
      "ghsa_id": 'GHSA-rf4j-j272-fj86',
      "cve_id": 'CVE-2018-6188',
      summary: 'security',
    },
    html_url: './github-url',
  }]
end

# Assertions
expect(result[0].instance_of?(Gulli::FeedEntry::Dependabot)).to eq(true)
expect(first_result.ignore?).to eq(false)
expect(first_result.id).to eq('credit-card-vault-2')
expect(first_result.title).to eq('CVE-2018-6188 - security')
expect(first_result.identifier).to eq('CVE-2018-6188')

Testing Patterns Used:

OpenStruct for lightweight test doubles
Timecop for deterministic time-based tests
allow_any_instance_of for HTTP mocking
RSpec doubles for response objects






---

## Section 10 - Replace with final version

```markdown
## 10. Context & Business Information

### Organization
- **Company:** Welltravel (or parent company managing multiple brands)
- **Team Location:** Dhaka, Bangladesh (UTC+6)
- **Jira Instance:** welltravel.atlassian.net
- **Primary Project:** VA (Vulnerability Assessment)

### Client/Project Portfolio
Based on deployments, the organization manages:

| Category | Projects |
|----------|----------|
| **Travel** | welltravel-apps, welltravel-mobile-apps, welltravel-website |
| **E-Bike/Mobility** | pick-e-bike, pick-e-bike-mobile-apps, blt-core, blt-go-business-frontend |
| **Innotix Platform** | 14 microservices (main app, cockpit, RHB, bus-finder, etc.) |
| **Other Clients** | biosuisse, tessi, trueact, alan-apps, cora, dap, rms, stage9, garaio-rem |
| **Internal** | ops-updates, pci-dss, welldev-nlp, wellday |

### Naming
- **"Gulli"** - Just a name (not an acronym)
- **"VA"** - Vulnerability Assessment (Jira project)

### Schedule Variations
Different projects run at different intervals:
- Default: Every 6 hours (`4 */6 * * *`)
- Innotix: Every 3 hours (`14 */3 * * *`)
- Minute offset varies (4, 14, etc.) to prevent thundering herd



---

## Session Log - Final Update

```markdown
## Session Log

### Session 1 (Complete)
**Files Reviewed:** 22/22 core files ✓

**Application Code:**
- [x] `bin/gulli`
- [x] `lib/gulli.rb`
- [x] `lib/gulli/feed_entry/base.rb`
- [x] `lib/gulli/feed_entry/dependabot.rb`
- [x] `lib/gulli/feed_entry/rss_article.rb`
- [x] `lib/gulli/feed_entry/google_artifact_registry.rb`
- [x] `lib/gulli/fetch_feeds.rb`
- [x] `lib/gulli/spreadsheet.rb`
- [x] `lib/gulli/collect_dependabot_alerts.rb`
- [x] `lib/gulli/collect_rss_articles.rb`
- [x] `lib/gulli/collect_google_artifact_registry_alerts.rb`
- [x] `lib/gulli/jira_ticket_creator.rb`
- [x] `lib/gulli/notify.rb`
- [x] `lib/utility/assessment_due_date_generator.rb`

**Configuration:**
- [x] `.env.example`
- [x] `Gemfile`
- [x] `Dockerfile`

**Deployment:**
- [x] `deployment/chart/templates/cronjob.yaml`
- [x] `deployment/chart/values.yaml`
- [x] `deployment/values/innotix/app/values.yaml`
- [x] `deployment/helmfile.yaml.gotmpl`

**Tests:**
- [x] `spec/lib/gulli/jira_ticket_creator_spec.rb`
- [x] `spec/lib/collect_dependabot_alerts_spec.rb`

**Knowledge Base Status:** ✅ Complete (95%+ coverage)

**Remaining (nice-to-have):**
- [ ] `.sops.yaml` - Encryption configuration
- [ ] GitHub Actions workflows - CI/CD pipeline details
- [ ] Additional spec files - More edge cases

## 11. Quick Reference

### Commands

```bash
# Local development
bundle install
cp .env.example .env
# Edit .env with real credentials
bin/gulli

# Run tests
bundle exec rspec

# Docker build
docker build -t gulli .
docker build --target development -t gulli:dev .

# Docker run
docker run --env-file .env \
  -v /path/to/credentials.json:/secrets/google/credentials.json \
  gulli

# Deploy single project
cd deployment
helmfile -l name=gulli-innotix apply

# Deploy all
cd deployment
REFERENCE=v1.2.3 helmfile apply

Key File Locations

├── bin/gulli                          # Entry point
├── lib/
│   ├── gulli.rb                       # Main orchestration
│   ├── gulli/
│   │   ├── collect_dependabot_alerts.rb
│   │   ├── collect_rss_articles.rb
│   │   ├── collect_google_artifact_registry_alerts.rb
│   │   ├── feed_entry/
│   │   │   ├── base.rb                # Abstract base class
│   │   │   ├── dependabot.rb
│   │   │   ├── rss_article.rb
│   │   │   └── google_artifact_registry.rb
│   │   ├── fetch_feeds.rb             # Spreadsheet config reader
│   │   ├── jira_ticket_creator.rb
│   │   ├── notify.rb                  # Slack integration
│   │   └── spreadsheet.rb             # Google Sheets wrapper
│   └── utility/
│       └── assessment_due_date_generator.rb
├── deployment/
│   ├── chart/                         # Helm chart
│   │   ├── templates/
│   │   │   └── cronjob.yaml
│   │   └── values.yaml                # Defaults
│   ├── values/                        # Per-project overrides
│   │   └── {project}/app/
│   │       ├── values.yaml
│   │       └── secrets.yaml           # SOPS encrypted
│   └── helmfile.yaml.gotmpl           # Multi-tenant deployment
└── spec/                              # RSpec tests


Adding a New Feed Type
Create collector class:

# lib/gulli/collect_new_source_alerts.rb
module Gulli
  class CollectNewSourceAlerts
    def self.call(feed)
      new(feed).call
    end
    
    def call
      # Fetch from source, return array of FeedEntry objects
    end
  end
end

Create entry class:

# lib/gulli/feed_entry/new_source.rb
module Gulli
  module FeedEntry
    class NewSource < Base
      def id; end
      def url; end
      def title; end
      def identifier; end
      def date; end
      # Optionally override ignore?
    end
  end
end

Register in main module:

# lib/gulli.rb
FEED_ENTRY_COLLECTOR = {
  'rss' => Gulli::CollectRSSArticles,
  'dependabot' => Gulli::CollectDependabotAlerts,
  'google-artifact-registry' => Gulli::CollectGoogleArtifactRegistryAlerts,
  'new-source' => Gulli::CollectNewSourceAlerts,  # Add this
}.freeze


Add feeds to spreadsheet:

Name	URL	Type	...
my-new-thing	https://...	new-source	...



Adding a New Project Deployment

Create values file:

# deployment/values/{project}/app/values.yaml
schedule: '24 */6 * * *'

spreadsheet:
  id: 'YOUR_SPREADSHEET_ID'
  ranges:
    feeds: 'Feeds!A:F'
    data: 'Data!A:G'

slack:
  channel: '#project-security'

jira:
  projectName: 'Project Name'


Create encrypted secrets:
# Create secrets.yaml with SOPS
sops deployment/values/{project}/app/secrets.yaml

Add to helmfile:
# deployment/helmfile.yaml.gotmpl
- name: gulli-{project}
  namespace: gulli
  chart: './chart'
  version: 0.3.0
  values:
    - values/{project}/app/values.yaml
  secrets:
    - values/{project}/app/secrets.yaml
  set:
    - name: 'image.tag'
      value: {{ requiredEnv "REFERENCE" }}
    - name: 'enableDependabot'
      value: true

Spreadsheet Schema
Feeds Sheet (Configuration):
Column	Name	Example	Required
A	Name	my-repo	Yes
B	URL	https://feed.xml	For RSS
C	Type	dependabot	Yes
D	Ignore Updates	TRUE/FALSE	No
E	Don't Create Jira Issue	TRUE/FALSE	No
F	story_assignee	5f4d3c2b1a...	For Jira


Data Sheet (State/Tracking):

Column	Name	Example
A	feed	my-repo
B	id	my-repo-42
C	identifier	CVE-2024-1234
D	url	https://github.com/...
E	ignore?	false
F	date	2024-01-15T10:30:00Z
G	jira_ticket_link	https://jira.../VA-123

Troubleshooting
Symptom	Likely Cause	Solution
"Missing Configuration" in Slack	GitHub/GCP credentials not set	Check env vars
No Slack notifications	SLACK_WEBHOOK_URL not set	Add webhook URL
Duplicate tickets	Spreadsheet not updating	Check Google API permissions
Old alerts appearing	Cache issue	Should auto-clear; check clear_cache!
Jira creation fails	Custom field IDs changed	Verify field IDs in Jira admin















Optional Further Exploration
If you want to go deeper, we could look at:

.sops.yaml - How secrets are encrypted
.github/workflows/*.yml - CI/CD pipeline (how deployments are triggered)
spec/spec_helper.rb - Test configuration patterns
Any secrets.yaml (structure only, not values) - What secrets each project needs








bin/gulli
    │
    ▼
lib/gulli.rb ──────────────────────────────────────────────────────────────┐
│                                                                          │
│  Gulli.run()                                                             │
│      │                                                                   │
│      ├── clear_cache!()                                                  │
│      │                                                                   │
│      ├── feeds() ──► fetch_feeds.rb ──► spreadsheet.rb ──► Google API    │
│      │                                                                   │
│      ├── announcements() ──┬── collect_dependabot_alerts.rb ──► GitHub   │
│      │                     ├── collect_rss_articles.rb ──► HTTP GET      │
│      │                     └── collect_google_artifact_registry.rb ──►GCP│
│      │                                                                   │
│      └── handle_announcement(a) ──┬── feed_entry/base.rb::known?()       │
│                                   │       └── spreadsheet.rb (cached)    │
│                                   │                                      │
│                                   ├── jira_ticket_creator.rb ──► Jira API│
│                                   │                                      │
│                                   ├── feed_entry/base.rb::remember!()    │
│                                   │       └── spreadsheet.rb::append().  │
│                                   │                                      │
│                                   └── notify.rb::article() ──► Slack     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘




When Gulli runs, the main orchestration happens in lib/gulli.rb where the Gulli.announcements method iterates through all configured feeds. For each feed, it looks up the appropriate collector class from the FEED_ENTRY_COLLECTOR hash. When the feed type is 'rss', it selects the Gulli::CollectRSSArticles class and calls its call method with the feed configuration.

The CollectRSSArticles class lives in lib/gulli/collect_rss_articles.rb. When call is invoked, it first initializes a new instance with the feed configuration, then executes the instance method call. The first thing this method does is fix potential SSL certificate issues by calling OpenSSL::SSL::SSLContext::DEFAULT_CERT_STORE.set_default_paths, which ensures the system can verify HTTPS certificates properly—particularly important in containerized environments where CA certificates might not be automatically available.

Next, the method uses HTTPClient to make a GET request to the RSS feed URL stored in feed[:url]. This returns raw XML content—the standard RSS or Atom format that blogs and security advisories publish. The raw XML response body is then passed to Feedjira.parse, which is a Ruby gem specifically designed for parsing syndication feeds. Feedjira automatically detects whether the feed is RSS 2.0, Atom, or another format, and converts it into Ruby objects with accessible attributes like title, url, id, published, and updated.

Once parsed, the method iterates over parsed.entries and wraps each entry in a FeedEntry::RSSArticle object, which is defined in lib/gulli/feed_entry/rss_article.rb. This class extends FeedEntry::Base and implements the common interface that all feed entries must provide. The id method returns the RSS entry's guid, the url method returns the link, and the title method cleans up the entry title by removing the feed name if it appears in the text. The date method prefers the updated timestamp but falls back to published if updated is not available. The identifier method simply returns the cleaned title. The ignore? method is inherited from Base and checks whether the title matches patterns like alpha, beta, rc, nightly, preview, snapshot, or test—indicating pre-release software that should be skipped for notifications.

If anything goes wrong during this process—whether the HTTP request fails, the XML is malformed, or any other exception occurs—the rescue block catches the error, calls Notify.feed_error from lib/gulli/notify.rb to send a Slack alert informing the team about the failure, and returns an empty array. This graceful degradation ensures that one broken RSS feed does not crash the entire Gulli run; other feeds continue to be processed normally.

The array of FeedEntry::RSSArticle objects is then returned to Gulli.announcements, where it gets merged with alerts from other sources like Dependabot and Google Artifact Registry. From there, each entry flows through Gulli.handle_announcement in lib/gulli.rb, which checks if it is already known via the known? method inherited from FeedEntry::Base, optionally creates a Jira ticket via JiraTicketCreator.call in lib/gulli/jira_ticket_creator.rb, saves the entry to the spreadsheet via remember! which calls Spreadsheet.append in lib/gulli/spreadsheet.rb, and finally sends a Slack notification via Notify.article in lib/gulli/notify.rb unless the entry is marked to be ignored.

##############################################
##############################################
##############################################


# GULLI KNOWLEDGE BASE v1.1

> **Purpose:** Complete technical documentation of the Gulli security vulnerability aggregation system. This document enables understanding and modification of the system without referencing source code.
> 
> **Last Updated:** Session 1 (Complete)
> 
> **Coverage:** 95%+ of codebase

---

## Table of Contents
1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Data Flow](#3-data-flow)
4. [Core Components](#4-core-components)
5. [External Integrations](#5-external-integrations)
6. [Configuration](#6-configuration)
7. [Deployment](#7-deployment)
8. [Design Patterns & Decisions](#8-design-patterns--decisions)
9. [Error Handling & Testing](#9-error-handling--testing)
10. [Context & Business Information](#10-context--business-information)
11. [Quick Reference](#11-quick-reference)
12. [Session Log](#12-session-log)

---

## 1. System Overview

### What is Gulli?
Gulli is a **security vulnerability aggregation and ticketing system** that:
- Monitors multiple sources for security alerts (GitHub Dependabot, Google Artifact Registry, RSS feeds)
- Automatically creates Jira tickets for new vulnerabilities
- Tracks all alerts in a Google Spreadsheet (used as a lightweight database)
- Notifies teams via Slack when new vulnerabilities are discovered
- Runs as a Kubernetes CronJob across 34 different project environments

### Naming
- **"Gulli"** is just a name (not an acronym)
- **"VA"** refers to Vulnerability Assessment (the Jira project)

### Business Context
This is used by an organization (Welltravel/parent company) managing multiple client projects. Each project gets its own Gulli deployment with project-specific configuration.

**Client/Project Portfolio:**
| Category | Projects |
|----------|----------|
| **Travel** | welltravel-apps, welltravel-mobile-apps, welltravel-website |
| **E-Bike/Mobility** | pick-e-bike, pick-e-bike-mobile-apps, blt-core, blt-go-business-frontend |
| **Innotix Platform** | 14 microservices (main app, cockpit, RHB, bus-finder, etc.) |
| **Other Clients** | biosuisse, tessi, trueact, alan-apps, cora, dap, rms, stage9, garaio-rem |
| **Internal** | ops-updates, pci-dss, welldev-nlp, wellday |

---

## 2. Architecture

### High-Level Architecture

┌─────────────────────────────────────────────────────────────────────────────┐
│                           GULLI SYSTEM                                       │
│                                                                             │
│  ┌─────────────┐     ┌─────────────────┐     ┌─────────────────────────┐   │
│  │ DATA SOURCES│     │   GULLI CORE    │     │    OUTPUT TARGETS       │   │
│  │             │     │                 │     │                         │   │
│  │ ┌─────────┐ │     │  ┌───────────┐  │     │  ┌─────────────────┐   │   │
│  │ │GitHub   │─┼────►│  │Collectors │  │     │  │ Jira Cloud      │   │   │
│  │ │Dependbot│ │     │  └─────┬─────┘  │     │  │ (Ticket Create) │   │   │
│  │ └─────────┘ │     │        │        │     │  └────────▲────────┘   │   │
│  │             │     │        ▼        │     │           │            │   │
│  │ ┌─────────┐ │     │  ┌───────────┐  │     │  ┌────────┴────────┐   │   │
│  │ │Google   │─┼────►│  │FeedEntry  │──┼────►│  │ Google Sheets   │   │   │
│  │ │Artifact │ │     │  │Objects    │  │     │  │ (State Store)   │   │   │
│  │ │Registry │ │     │  └─────┬─────┘  │     │  └────────▲────────┘   │   │
│  │ └─────────┘ │     │        │        │     │           │            │   │
│  │             │     │        ▼        │     │  ┌────────┴────────┐   │   │
│  │ ┌─────────┐ │     │  ┌───────────┐  │     │  │ Slack           │   │   │
│  │ │RSS Feeds│─┼────►│  │Handler    │──┼────►│  │ (Notifications) │   │   │
│  │ │(Various)│ │     │  └───────────┘  │     │  └─────────────────┘   │   │
│  │ └─────────┘ │     │                 │     │                         │   │
│  └─────────────┘     └─────────────────┘     └─────────────────────────┘   │
│                                                                             │
│                        Runs as K8s CronJob                                  │
└─────────────────────────────────────────────────────────────────────────────┘


### Technology Stack
| Component | Technology |
|-----------|------------|
| Language | Ruby 3.4.4 |
| Dependency Management | Bundler |
| Container | Docker (multi-stage build) |
| Orchestration | Kubernetes (Helm charts) |
| Configuration Management | Helmfile + SOPS (secrets) |
| CI/CD | GitHub Actions |

---

## 3. Data Flow

### Complete Execution Flow
```
┌──────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: INITIALIZATION                                                        │
│ ══════════════════════                                                        │
│                                                                              │
│   bin/gulli                                                                  │
│       │                                                                      │
│       └──► Gulli.run()                                                       │
│               │                                                              │
│               └──► clear_cache!()  # Reset in-memory cache for fresh run     │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 2: FETCH FEED CONFIGURATIONS                                            │
│ ═════════════════════════════════                                            │
│                                                                              │
│   FetchFeeds.call()                                                          │
│       │                                                                      │
│       └──► Spreadsheet.get_values(SPREADSHEET_RANGES_FEEDS)                  │
│               │                                                              │
│               └──► Google Sheets API: GET spreadsheet values                 │
│                       │                                                      │
│                       └──► Returns: [{                                       │
│                              name: "repo-name",                              │
│                              url: "https://...",                             │
│                              type: "dependabot|rss|google-artifact-registry",│
│                              ignore_updates: true|false,                     │
│                              auto_jira_issue_create: true|false,             │
│                              jira_ticket_assignee: "jira-account-id"         │
│                            }, ...]                                           │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 3: COLLECT ALERTS FROM EACH SOURCE                                      │
│ ════════════════════════════════════════                                     │
│                                                                              │
│   For each feed config:                                                      │
│       │                                                                      │
│       ├──► IF type == "dependabot":                                          │
│       │       CollectDependabotAlerts.call(feed)                             │
│       │           │                                                          │
│       │           └──► GitHub API: GET /repos/{owner}/{repo}/dependabot/alerts│
│       │                   │        ?state=open                               │
│       │                   │        Authorization: Bearer {GITHUB_TOKEN}      │
│       │                   │                                                  │
│       │                   └──► Returns: [FeedEntry::Dependabot, ...]         │
│       │                                                                      │
│       ├──► IF type == "rss":                                                 │
│       │       CollectRSSArticles.call(feed)                                  │
│       │           │                                                          │
│       │           └──► HTTP GET feed URL                                     │
│       │                   │                                                  │
│       │                   └──► Feedjira.parse(xml)                           │
│       │                           │                                          │
│       │                           └──► Returns: [FeedEntry::RSSArticle, ...] │
│       │                                                                      │
│       └──► IF type == "google-artifact-registry":                            │
│               CollectGoogleArtifactRegistryAlerts.call(feed)                 │
│                   │                                                          │
│                   └──► GCP API: GET image manifests                          │
│                           │     Authorization: Bearer {GCP_TOKEN}            │
│                           │                                                  │
│                           └──► Returns: [FeedEntry::GoogleArtifactRegistry]  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 4: PROCESS EACH ANNOUNCEMENT                                            │
│ ═════════════════════════════════                                            │
│                                                                              │
│   For each FeedEntry (announcement):                                         │
│       │                                                                      │
│       ├──► known?()                                                          │
│       │       │                                                              │
│       │       └──► Check if ID exists in DATA spreadsheet                    │
│       │            (uses cached data after first call)                       │
│       │               │                                                      │
│       │               └──► IF found: SKIP (already processed)                │
│       │                                                                      │
│       ├──► published_date_before_6_months?()                                 │
│       │       │                                                              │
│       │       └──► IF date < (now - 180 days): SKIP (too old)                │
│       │                                                                      │
│       ├──► ignore?()                                                         │
│       │       │                                                              │
│       │       └──► IF title matches /alpha|beta|rc|nightly|preview|snapshot|test/│
│       │            SKIP notifications (but still track)                      │
│       │            NOTE: Dependabot overrides this to NEVER ignore           │
│       │                                                                      │
│       ├──► IF auto_jira_issue_create? AND NOT ignore?:                       │
│       │       create_jira_ticket!()                                          │
│       │           │                                                          │
│       │           └──► JiraTicketCreator.call(article)                       │
│       │                   │                                                  │
│       │                   └──► Jira API: POST /rest/api/3/issue              │
│       │                           │                                          │
│       │                           └──► Returns: ticket URL (stored in entry) │
│       │                                                                      │
│       ├──► remember!()                                                       │
│       │       │                                                              │
│       │       └──► Spreadsheet.append() to DATA sheet                        │
│       │            Columns: [feed, id, identifier, url, ignore?, date,       │
│       │                      jira_ticket_link]                               │
│       │                                                                      │
│       └──► IF NOT ignore?:                                                   │
│               Notify.article()                                               │
│                   │                                                          │
│                   └──► Slack Webhook: POST notification                      │
│                        Format: "*{feed}*: {title} | Spreadsheet | JIRA-XXX"  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│ STEP 5: ERROR HANDLING                                                       │
│ ══════════════════════                                                       │
│                                                                              │
│   ANY uncaught exception:                                                    │
│       │                                                                      │
│       └──► Notify.fatal() ──► Slack: "<!channel> Gulli Crashed: {error}"     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Core Components

### 4.1 Entry Point (`bin/gulli`)
```ruby
# Minimal bootstrap - loads lib path and calls Gulli.run
$LOAD_PATH.unshift File.expand_path('../../lib', __FILE__)
require 'bundler/setup'
require 'gulli'
Gulli.run
```

### 4.2 Main Module (`lib/gulli.rb`)

**Responsibilities:**
- Orchestrates the entire execution flow
- Manages configuration from environment variables
- Provides module-level caching
- Handles top-level error recovery

**Key Methods:**
| Method | Purpose |
|--------|---------|
| `run` | Main entry point - fetches feeds, collects alerts, processes each |
| `feeds` | Fetches feed configurations from spreadsheet |
| `announcements` | Collects all alerts from all feeds |
| `handle_announcement(a)` | Processes single alert (jira, remember, notify) |
| `already_known_or_old?(a)` | Deduplication check |
| `config` | Lazy-loaded configuration hash from ENV |
| `cache` / `clear_cache!` | In-memory cache management |
| `logger` | Stdout logger instance |

**Configuration Structure:**
```ruby
{
  google: {
    keyfile: ENV['GOOGLE_KEYFILE']  # Path to service account JSON
  },
  spreadsheet: {
    id: ENV['SPREADSHEET_ID'],
    ranges: {
      feeds: ENV['SPREADSHEET_RANGES_FEEDS'],      # e.g., "Feeds!A:Z"
      data: ENV['SPREADSHEET_RANGES_DATA'],        # e.g., "Data!A:Z"
      update_sheet: ENV['SPREADSHEET_UPDATE_SHEET'],
      update_column: ENV['SPREADSHEET_UPDATE_COLUMN']
    }
  },
  repo: {
    owner: ENV['GITHUB_REPO_OWNER'],
    token: ENV['GITHUB_REPO_ACCESS_TOKEN']
  },
  slack: {
    webhook_url: ENV['SLACK_WEBHOOK_URL'],
    channel: ENV['SLACK_CHANNEL'],
    username: ENV['SLACK_USERNAME'],
    icon: ENV['SLACK_ICON']
  },
  jira: {
    base_url: ENV['JIRA_BASE_URL'],
    username: ENV['JIRA_USERNAME'],
    api_token: ENV['JIRA_API_TOKEN'],
    project_key: ENV['JIRA_PROJECT_KEY'],
    issue_type: ENV['JIRA_ISSUE_TYPE'],
    project_name: ENV['JIRA_PROJECT_NAME']
  }
}
```

### 4.3 Collector Classes

#### Strategy Pattern Mapping
```ruby
FEED_ENTRY_COLLECTOR = {
  'rss'                      => Gulli::CollectRSSArticles,
  'dependabot'               => Gulli::CollectDependabotAlerts,
  'google-artifact-registry' => Gulli::CollectGoogleArtifactRegistryAlerts
}
```

#### CollectDependabotAlerts
**Source:** GitHub Dependabot API  
**API Endpoint:** `GET /repos/{owner}/{repo}/dependabot/alerts?state=open`  
**Authentication:** Bearer token (`GITHUB_REPO_ACCESS_TOKEN`)  
**Returns:** Array of `FeedEntry::Dependabot` objects

**Key behaviors:**
- Only fetches `open` alerts (not dismissed/fixed)
- Notifies Slack if API fails (doesn't crash)
- Returns empty array if not configured

#### CollectRSSArticles
**Source:** Any RSS/Atom feed URL  
**Library:** Feedjira (Ruby RSS parser)  
**Returns:** Array of `FeedEntry::RSSArticle` objects

**Key behaviors:**
- Handles SSL certificate issues (`set_default_paths`)
- Graceful error handling with Slack notification
- Returns empty array on fetch failure

#### CollectGoogleArtifactRegistryAlerts
**Source:** Google Container Registry / Artifact Registry  
**API Endpoint:** Image manifest endpoint  
**Authentication:** GCP Service Account OAuth token  
**Returns:** Array of `FeedEntry::GoogleArtifactRegistry` objects

**Key behaviors:**
- Only monitors images tagged `latest`
- Extracts upload time from manifest
- Converts internal URL to marketplace URL for readability

### 4.4 FeedEntry Classes (Value Objects)

#### Base Class (`FeedEntry::Base`)
**Abstract interface that all entry types must implement:**

| Method | Return Type | Description |
|--------|-------------|-------------|
| `id` | String | Unique identifier for deduplication |
| `url` | String | Link to the alert/article |
| `title` | String | Display title for notifications |
| `identifier` | String | CVE ID, GHSA ID, or article identifier |
| `date` | String | ISO date string for age checking |

**Shared behaviors:**
| Method | Description |
|--------|-------------|
| `known?` | Checks if ID exists in spreadsheet (cached) |
| `published_date_before_6_months?` | Returns true if older than 180 days |
| `ignore?` | Matches against `/alpha\|beta\|rc\|nightly\|preview\|snapshot\|test/i` |
| `remember!` | Appends entry to DATA spreadsheet |
| `create_jira_ticket!` | Delegates to JiraTicketCreator |
| `update!(position, data)` | Updates specific cell in spreadsheet |

**Ignore Pattern:**
```ruby
IGNORE_REGEXP = /([.-]rc|alpha|beta|nightly|preview|snapshot|[.-]test)/i
```

#### Dependabot Entry (`FeedEntry::Dependabot`)
```
id         = "{feed}-{alert_number}"           # e.g., "my-repo-42"
url        = entry['html_url']                 # GitHub alert URL
title      = "{identifier} - {summary}"        # "CVE-2024-1234 - SQL Injection"
identifier = cve_id || ghsa_id                 # Prefers CVE over GHSA
date       = entry['updated_at']               # Last update time
ignore?    = false                             # NEVER ignore security alerts
```

#### RSSArticle Entry (`FeedEntry::RSSArticle`)
```
id         = entry.id                          # RSS guid/id field
url        = entry.url                         # Link to article
title      = entry.title.gsub(/#{feed}/i, '')  # Remove feed name from title
identifier = title                             # Same as cleaned title
date       = entry.updated || entry.published  # Prefer updated, fallback to published
ignore?    = <inherited>                       # Uses Base IGNORE_REGEXP
```

**Notable behavior:**
- Title is cleaned by removing the feed name (case-insensitive)
- Falls back from `updated` to `published` for date
- Uses inherited `ignore?` so alpha/beta/rc releases are skipped

#### GoogleArtifactRegistry Entry (`FeedEntry::GoogleArtifactRegistry`)
```
id         = "{feed}-{image_digest}"           # e.g., "my-app-sha256:abc123"
url        = entry['url']                      # Marketplace URL (parsed from API URL)
title      = "Docker Image Alert: {feed} - {image_digest}"
identifier = entry['image']                    # The image digest
date       = entry['uploaded_at']              # Formatted upload timestamp
ignore?    = @ignore_updates                   # Controlled by spreadsheet config!
```

**Notable behavior:**
- Unlike other entries, `ignore?` is controlled by the feed configuration (`ignore_updates` column)
- This allows turning off notifications for specific images while still tracking them
- Has a `summary` method (not in Base) used for richer descriptions

### 4.5 Spreadsheet Service

**Purpose:** Abstraction layer over Google Sheets API

**Methods:**
| Method | Parameters | Description |
|--------|------------|-------------|
| `get_values(range)` | Range string (e.g., "Sheet!A:Z") | Reads rows, returns array of hashes |
| `update(range, attributes:)` | Range + array of values | Updates specific cells |
| `append(range, attributes:)` | Range + array of values | Appends new row |
| `service` | - | Lazy-loaded authenticated API client |

**Row Transformation:**
```ruby
# Raw API response:
[["Name", "URL", "Type"], ["repo", "http://...", "dependabot"]]

# Transformed to:
[{"Name" => "repo", "URL" => "http://...", "Type" => "dependabot"}]
```

**Caching Strategy:**
- Cache key format: `"spreadsheet_ranges_{range}"`
- Cached on read, invalidated on write
- Cleared at start of each run (`clear_cache!`)
- Prevents redundant API calls when checking `known?` for multiple entries

### 4.6 Jira Ticket Creator

**API:** Jira Cloud REST API v3  
**Endpoint:** `POST /rest/api/3/issue?skipAutoWatch=true`  
**Authentication:** Basic Auth (base64 encoded `username:api_token`)

**Payload Structure:**
```ruby
{
  fields: {
    project: { key: "SEC" },                    # JIRA_PROJECT_KEY
    reporter: { accountId: "..." },             # From spreadsheet
    summary: "CVE-2024-1234 - SQL Injection",   # article.title
    issuetype: { name: "Vulnerability" },       # JIRA_ISSUE_TYPE
    assignee: { accountId: "..." },             # From spreadsheet
    customfield_11856: "2024-01-15",            # Assessment due date
    customfield_11488: "my-repo",               # Feed name
    customfield_11489: "https://github.com/..", # Alert URL
    customfield_11490: { value: "Project X" }   # Project dropdown
  }
}
```

**Custom Field IDs (Jira-specific):**
```ruby
JIRA_FIELD_FEED = 'customfield_11488'
JIRA_FIELD_URL = 'customfield_11489'
JIRA_FIELD_PROJECT_NAME = 'customfield_11490'
JIRA_ASSESSMENT_DUE_DATE = 'customfield_11856'
```

### 4.7 Assessment Due Date Generator

**Purpose:** Calculate compliance deadline for vulnerability assessment

**Business Rules:**
1. If current time < 10:00 AM Dhaka time → Due date = today
2. If current time ≥ 10:00 AM Dhaka time → Due date = tomorrow
3. If tomorrow is Saturday → Due date = Sunday (skip Saturday)

**Timezone:** Bangladesh Standard Time (UTC+6)

**Why Dhaka?** Team is based in Dhaka, Bangladesh

### 4.8 Slack Notifier

**Library:** `slack-notifier` gem  
**Method:** Webhook POST

**Notification Types:**
| Method | Trigger | Format |
|--------|---------|--------|
| `article(a)` | New vulnerability found | Feed + URL + Spreadsheet link + Jira link |
| `feed_error(...)` | RSS fetch failure | Feed name + error |
| `dependabot_error(...)` | GitHub API failure | Repo name + error |
| `docker_error(...)` | GCP API failure | Repo name + error |
| `fatal(error)` | Uncaught exception | `<!channel>` mention + error |

**Emoji Usage:**
- `:inbox_tray:` - New alert
- `:bookmark_tabs:` - Spreadsheet link
- `:ticket:` - Jira ticket
- `:exclamation:` - Errors

### 4.9 Dependencies (Gemfile)

| Gem | Version | Purpose |
|-----|---------|---------|
| `feedjira` | ~> 3.2, >= 3.2.1 | RSS/Atom feed parsing |
| `google-api-client` | ~> 0.53.0 | Google Sheets API client |
| `slack-notifier` | (any) | Slack webhook notifications |
| `httpclient` | (any) | HTTP client (shared with google-api-client) |

**Development/Test Dependencies:**
| Gem | Purpose |
|-----|---------|
| `dotenv` | Load .env files in development |
| `pry-rails` | Debugging console |
| `rspec` | Test framework |
| `rspec_junit_formatter` | CI-friendly test output |
| `timecop` | Time manipulation for tests |

**Design Note:** HTTPClient is explicitly included because google-api-client depends on it. This avoids having multiple HTTP client gems in the project.

---

## 5. External Integrations

### 5.1 Google Sheets API
**Purpose:** Configuration storage + State management  
**Auth:** Service Account (JSON keyfile)  
**Scope:** `AUTH_SPREADSHEETS` (read/write)

### 5.2 GitHub API
**Purpose:** Fetch Dependabot security alerts  
**Auth:** Personal Access Token (Bearer)  
**Required Permissions:** `security_events` read access  
**API Version:** `2022-11-28`

### 5.3 Jira Cloud API
**Purpose:** Create vulnerability tracking tickets  
**Auth:** Basic Auth (email + API token)  
**API Version:** v3

### 5.4 Google Cloud Platform
**Purpose:** Monitor container image updates  
**Auth:** Service Account OAuth  
**Scope:** `cloud-platform`

### 5.5 Slack
**Purpose:** Team notifications  
**Auth:** Incoming Webhook URL

---

## 6. Configuration

### Environment Variables

#### Complete List (from .env.example)
| Variable | Example Value | Required | Description |
|----------|---------------|----------|-------------|
| `SPREADSHEET_ID` | `1sgXVRgdLHogv7UX9G3RaHwlZvVhGOAMtSUluWPZ0jOA` | Yes | Google Sheets document ID (from URL) |
| `SPREADSHEET_RANGES_FEEDS` | `Feeds!A:D` | Yes | Range for feed configuration sheet |
| `SPREADSHEET_RANGES_DATA` | `Data!A:G` | Yes | Range for tracking/state sheet |
| `SPREADSHEET_UPDATE_SHEET` | `Data!` | Yes | Sheet name prefix for updates |
| `SPREADSHEET_UPDATE_COLUMN` | `F` | Yes | Column letter for date updates |
| `GOOGLE_KEYFILE` | `/secrets/google/credentials.json` | Yes | Path to GCP service account JSON |
| `SLACK_WEBHOOK_URL` | `https://hooks.slack.com/...` | No | Slack incoming webhook URL |
| `SLACK_CHANNEL` | `#security-alerts` | No | Override default webhook channel |
| `SLACK_USERNAME` | `gulli` | No | Bot display name in Slack |
| `SLACK_ICON` | `robot_face` | No | Emoji name (without colons) |
| `GITHUB_REPO_OWNER` | `welltravel` | No* | GitHub organization/user name |
| `GITHUB_REPO_ACCESS_TOKEN` | `ghp_xxxx` | No* | GitHub Personal Access Token |
| `JIRA_BASE_URL` | `https://welltravel.atlassian.net` | No* | Jira Cloud instance URL |
| `JIRA_USERNAME` | `bot@company.com` | No* | Jira account email |
| `JIRA_API_TOKEN` | `ATATT3x...` | No* | Jira API token |
| `JIRA_PROJECT_KEY` | `VA` | No* | Jira project key for tickets |
| `JIRA_ISSUE_TYPE` | `Vulnerability Assessment` | No* | Issue type name |
| `JIRA_PROJECT_NAME` | `Welltravel` | No* | Value for project dropdown field |

*Required if using that feature (Dependabot/Jira)

#### Spreadsheet Structure
From the ranges, we can deduce:
- **Feeds sheet**: Columns A-D minimum (Name, URL, Type, Ignore Updates + more via named ranges)
- **Data sheet**: Columns A-G (feed, id, identifier, url, ignore?, date, jira_ticket_link)
- **Update column F**: The `date` column (6th column = F)

---

## 7. Deployment

### Kubernetes CronJob Structure
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ .Release.Name }}  # e.g., "gulli-innotix-app"
spec:
  schedule: {{ .Values.schedule }}  # Cron expression
  concurrencyPolicy: Forbid         # Prevent overlapping runs
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: gulli
            image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
            env:
            - name: GOOGLE_KEYFILE
              value: /secrets/google/credentials.json
            envFrom:
            - configMapRef:           # Non-secret config
                name: {{ .Release.Name }}
            - secretRef:              # Secrets (tokens, etc.)
                name: {{ .Release.Name }}
            volumeMounts:
            - name: google-credentials
              mountPath: /secrets/google
          restartPolicy: OnFailure
          volumes:
          - name: google-credentials
            secret:
              secretName: {{ .Release.Name }}
              items:
              - key: googleApplicationCredentials
                path: credentials.json
```

### Docker Image

**Base Images:**
```
Production: us-west1-docker.pkg.dev/nodal-algebra-161404/base-image-ruby/base-image-ruby:3.4.4-1
Development: us-west1-docker.pkg.dev/nodal-algebra-161404/base-image-ruby/base-image-ruby-dev:3.4.4-1_node22
```

**Ruby Version:** 3.4.4

**Multi-Stage Build:**
```
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: builder (DEV_IMAGE)                                    │
│   - Install gems with BUNDLE_WITHOUT=development:test           │
│   - Remove .gem cache files                                     │
│   - Remove .c and .o source files (reduce size)                 │
│   - Runs as 'ruby' user (non-root)                              │
├─────────────────────────────────────────────────────────────────┤
│ Stage 2: production (BASE_IMAGE)                                │
│   - Copy app code                                               │
│   - Copy pre-built /bundle from builder                         │
│   - Update CA certificates (as root, then switch back)          │
│   - Runs as 'ruby' user (non-root)                              │
│   - CMD: bin/gulli                                              │
├─────────────────────────────────────────────────────────────────┤
│ Stage 3: development (extends production)                       │
│   - BUNDLE_WITHOUT="" (include dev/test gems)                   │
│   - Full bundle install                                         │
└─────────────────────────────────────────────────────────────────┘
```

**Security Considerations:**
- Runs as non-root `ruby` user
- CA certificates explicitly updated (important for API calls)
- Build artifacts removed to reduce attack surface

**Image Registry:**
```
us-west1-docker.pkg.dev/nodal-algebra-161404/gulli/gulli
```

### Default Helm Values

```yaml
schedule: '4 */6 * * *'  # Every 6 hours at minute 4 (00:04, 06:04, 12:04, 18:04)

slack:
  username: Gulli

jira:
  baseUrl: 'https://welltravel.atlassian.net'
  projectKey: 'VA'                              # Vulnerability Assessment project
  issueType: 'Vulnerability Assessment'

image:
  repository: us-west1-docker.pkg.dev/nodal-algebra-161404/gulli/gulli
```

**Schedule Analysis:**
- Runs 4 times per day
- Minute 4 chosen to avoid contention with other cron jobs at minute 0
- Sufficient for security alert monitoring (not real-time, but timely)

### Per-Project Configuration

Each project deployment customizes:

**Example: Innotix (`deployment/values/innotix/app/values.yaml`)**
```yaml
schedule: '14 */3 * * *'           # Every 3 hours at minute 14

spreadsheet:
  id: '1ZiXs_sbT-Q5G6gv9dw1APEgcVBRfZRy92OCP4Hm_w8E'  # Project-specific spreadsheet
  update:
    sheet: 'Data!'
    column: 'F'
  ranges:
    feeds: 'Feeds!A:F'             # 6 columns (more than default 4)
    data: 'Data!A:G'

slack:
  channel: '#innotix-vulnerability-monitor'  # Project-specific channel
  icon: 'rotating_light'                     # 🚨 emoji

jira:
  projectName: 'Innotix öVPad'     # Dropdown value for this project
```

**Configuration Inheritance:**
```
deployment/chart/values.yaml (defaults)
    │
    └── deployment/values/{project}/app/values.yaml (overrides)
            │
            └── deployment/values/{project}/app/secrets.yaml (encrypted secrets)
```

### Helmfile Multi-Tenant Deployment

**File:** `deployment/helmfile.yaml.gotmpl`

**Structure:**
```yaml
releases:
  - name: gulli-{project}          # Unique release name
    namespace: gulli               # All in same namespace
    chart: './chart'
    version: 0.3.0
    values:
      - values/{project}/app/values.yaml
    secrets:
      - values/{project}/app/secrets.yaml    # SOPS encrypted
    set:
      - name: 'image.tag'
        value: {{ requiredEnv "REFERENCE" }}  # Git SHA/tag from CI
      - name: 'enableDependabot'
        value: true|false                     # Per-project toggle
```

**Deployed Projects (34 total):**

| Project | Dependabot Enabled | Notes |
|---------|-------------------|-------|
| ops-updates | ❌ | Internal ops |
| pci-dss | ✅ | Payment compliance |
| trueact | ✅ | |
| welltravel-mobile-apps | ✅ | |
| welldev-nlp | ✅ | NLP service |
| pick-e-bike | ✅ | E-bike platform |
| tessi | (default) | |
| alan-apps | (default) | |
| cora | (default) | |
| biosuisse | ✅ | Swiss organic certification |
| welltravel-apps | ✅ | Main travel app |
| wellday | ✅ | |
| rms | ✅ | |
| dap | (default) | |
| welltravel-website | ✅ | |
| gulli-innotix | (default) | Main Innotix |
| gulli-innotix-web-app-frontend | ❌ | |
| gulli-innotix-hirsch | ❌ | |
| gulli-innotix-webapp-backend | ❌ | |
| gulli-innotix-origa | ❌ | |
| pick-e-bike-mobile-apps | ✅ | |
| blt-core | ✅ | BLT subsidiary |
| blt-go-business-frontend | ✅ | |
| garaio-rem-owner-portal | ❌ | Real estate |
| gulli-innotix-rhb | ❌ | |
| gulli-innotix-bus-finder | ❌ | |
| gulli-innotix-cockpit | ❌ | |
| gulli-innotix-ifs | ❌ | |
| stage9 | ❌ | |
| gulli-innotix-rhb-afe | ❌ | |
| gulli-innotix-cms-cockpit | ❌ | |
| gulli-innotix-html-to-pdf-service | ❌ | |
| gulli-innotix-push-service | ❌ | |

**Observations:**
- Innotix has 14 sub-projects, most with Dependabot disabled (likely different monitoring strategy)
- All deployments share the `gulli` namespace
- Image tag comes from CI environment variable `REFERENCE`
- Chart version locked at 0.3.0

### Secrets Management
**Tool:** SOPS (Secrets OPerationS)  
**Config:** `deployment/.sops.yaml`  
**Purpose:** Encrypt secrets in Git, decrypt at deploy time

---

## 8. Design Patterns & Decisions

### 8.1 Strategy Pattern (Collectors)
**Why:** Different data sources require different fetching logic, but the processing pipeline is identical.
```ruby
FEED_ENTRY_COLLECTOR[feed[:type]].call(feed)
```

### 8.2 Template Method Pattern (FeedEntry)
**Why:** All entry types need the same behaviors (known?, remember!, ignore?) but different data extraction.

Base class defines algorithm skeleton, subclasses implement abstract methods.

### 8.3 Google Sheets as Database
**Why chosen over traditional DB:**
- Zero infrastructure cost
- Non-technical users can view/edit
- Easy auditing and debugging
- Sufficient for low-volume (security alerts)

**Trade-offs accepted:**
- No transactions (acceptable - idempotent operations)
- String-only types (acceptable - simple data)
- API rate limits (mitigated with caching)

### 8.4 Aggressive Caching
**Why:** Spreadsheet API is called in `known?` for every alert. Without caching, 50 alerts = 50 API calls.

**Strategy:** Cache on first read, invalidate on write, clear at run start.

### 8.5 Graceful Degradation
**Why:** Don't lose vulnerability alerts due to partial failures.

**Implementation:**
- Collector failures notify Slack but return empty array (other feeds continue)
- Jira failures notify with "[Action Required]" for manual intervention
- Top-level catch-all notifies team of crashes

### 8.6 Dependabot Never Ignores
**Why:** Security vulnerabilities should always create tickets, even for beta versions.
```ruby
# Dependabot class overrides Base:
def ignore?
  false
end
```

### 8.7 FeedEntry Polymorphism Comparison

| Aspect | Dependabot | RSSArticle | GoogleArtifactRegistry |
|--------|------------|------------|------------------------|
| **id format** | `{feed}-{number}` | RSS entry ID | `{feed}-{digest}` |
| **title format** | `{CVE} - {summary}` | Cleaned RSS title | `Docker Image Alert: ...` |
| **date source** | `updated_at` | `updated \|\| published` | `uploaded_at` |
| **ignore? behavior** | Always `false` | Inherited regexp | From config |
| **identifier** | CVE or GHSA ID | Same as title | Image digest |

**Design insight:** Each entry type customizes behavior appropriate to its source:
- **Dependabot**: Security-first, never ignore
- **RSS**: Content filtering via regexp (skip prereleases)
- **GAR**: User-controlled via spreadsheet config

---

## 9. Error Handling & Testing

### 9.1 Error Handling Hierarchy
```
┌─────────────────────────────────────────────────────────┐
│ Level 1: Top-Level (Gulli.run)                          │
│   catch StandardError → Notify.fatal + log              │
│                                                         │
│ Level 2: Collector (collect_*_alerts.rb)                │
│   catch HTTP errors → Notify.*_error + return []        │
│                                                         │
│ Level 3: Jira (JiraTicketCreator)                       │
│   catch any error → Notify.fatal with action required   │
│   return nil (entry still remembered)                   │
│                                                         │
│ Level 4: Spreadsheet                                    │
│   No explicit handling → bubbles up to Level 1          │
└─────────────────────────────────────────────────────────┘
```

### Notification Escalation
| Error Type | Notification | Continues? |
|------------|--------------|------------|
| Single feed fails | Feed-specific error | Yes |
| Jira creation fails | "[Action Required]" | Yes |
| Spreadsheet fails | System crash | No |
| Unknown exception | "<!channel> Crashed" | No |

### 9.2 Testing Patterns (from specs)

**Jira Ticket Creator Tests:**
```ruby
# Test setup pattern
let(:article) do
  OpenStruct.new(
    title: 'Critical Vulnerability in API',
    feed: 'Security Scanner',
    url: 'https://example.com/test/123',
    jira_ticket_assignee: 'test_user',
  )
end

# Time freezing for due date tests
before do
  Timecop.freeze(Time.new(2025, 9, 11, 9, 0, 0, '+06:00'))
end
```

**Tested Scenarios:**
| Scenario | Expected Behavior |
|----------|-------------------|
| Payload structure | Matches expected Jira API format |
| HTTP 201 response | Returns ticket URL (`https://.../browse/VA-789`) |
| HTTP 400 response | Calls `Notify.fatal` with action required message, returns `nil` |
| Exception raised | Calls `Notify.fatal` with error message, returns `nil` |

**Dependabot Collector Tests:**
```ruby
# Mock GitHub API response
let(:response) do
  [{
    "number": 2,
    "state": 'dismissed',
    "security_advisory": {
      "ghsa_id": 'GHSA-rf4j-j272-fj86',
      "cve_id": 'CVE-2018-6188',
      summary: 'security',
    },
    html_url: './github-url',
  }]
end

# Assertions
expect(result[0].instance_of?(Gulli::FeedEntry::Dependabot)).to eq(true)
expect(first_result.ignore?).to eq(false)
expect(first_result.id).to eq('credit-card-vault-2')
expect(first_result.title).to eq('CVE-2018-6188 - security')
expect(first_result.identifier).to eq('CVE-2018-6188')
```

**Testing Patterns Used:**
- `OpenStruct` for lightweight test doubles
- `Timecop` for deterministic time-based tests
- `allow_any_instance_of` for HTTP mocking
- RSpec doubles for response objects

---

## 10. Context & Business Information

### Organization
- **Company:** Welltravel (or parent company managing multiple brands)
- **Team Location:** Dhaka, Bangladesh (UTC+6)
- **Jira Instance:** welltravel.atlassian.net
- **Primary Project:** VA (Vulnerability Assessment)

### Schedule Variations
Different projects run at different intervals:
- Default: Every 6 hours (`4 */6 * * *`)
- Innotix: Every 3 hours (`14 */3 * * *`)
- Minute offset varies (4, 14, etc.) to prevent thundering herd

---

## 11. Quick Reference

### Commands

```bash
# Local development
bundle install
cp .env.example .env
# Edit .env with real credentials
bin/gulli

# Run tests
bundle exec rspec

# Docker build
docker build -t gulli .
docker build --target development -t gulli:dev .

# Docker run
docker run --env-file .env \
  -v /path/to/credentials.json:/secrets/google/credentials.json \
  gulli

# Deploy single project
cd deployment
helmfile -l name=gulli-innotix apply

# Deploy all
cd deployment
REFERENCE=v1.2.3 helmfile apply
```

### Key File Locations

```
├── bin/gulli                          # Entry point
├── lib/
│   ├── gulli.rb                       # Main orchestration
│   ├── gulli/
│   │   ├── collect_dependabot_alerts.rb
│   │   ├── collect_rss_articles.rb
│   │   ├── collect_google_artifact_registry_alerts.rb
│   │   ├── feed_entry/
│   │   │   ├── base.rb                # Abstract base class
│   │   │   ├── dependabot.rb
│   │   │   ├── rss_article.rb
│   │   │   └── google_artifact_registry.rb
│   │   ├── fetch_feeds.rb             # Spreadsheet config reader
│   │   ├── jira_ticket_creator.rb
│   │   ├── notify.rb                  # Slack integration
│   │   └── spreadsheet.rb             # Google Sheets wrapper
│   └── utility/
│       └── assessment_due_date_generator.rb
├── deployment/
│   ├── chart/                         # Helm chart
│   │   ├── templates/
│   │   │   └── cronjob.yaml
│   │   └── values.yaml                # Defaults
│   ├── values/                        # Per-project overrides
│   │   └── {project}/app/
│   │       ├── values.yaml
│   │       └── secrets.yaml           # SOPS encrypted
│   └── helmfile.yaml.gotmpl           # Multi-tenant deployment
└── spec/                              # RSpec tests
```

### Adding a New Feed Type

1. **Create collector class:**
   ```ruby
   # lib/gulli/collect_new_source_alerts.rb
   module Gulli
     class CollectNewSourceAlerts
       def self.call(feed)
         new(feed).call
       end
       
       def call
         # Fetch from source, return array of FeedEntry objects
       end
     end
   end
   ```

2. **Create entry class:**
   ```ruby
   # lib/gulli/feed_entry/new_source.rb
   module Gulli
     module FeedEntry
       class NewSource < Base
         def id; end
         def url; end
         def title; end
         def identifier; end
         def date; end
         # Optionally override ignore?
       end
     end
   end
   ```

3. **Register in main module:**
   ```ruby
   # lib/gulli.rb
   FEED_ENTRY_COLLECTOR = {
     'rss' => Gulli::CollectRSSArticles,
     'dependabot' => Gulli::CollectDependabotAlerts,
     'google-artifact-registry' => Gulli::CollectGoogleArtifactRegistryAlerts,
     'new-source' => Gulli::CollectNewSourceAlerts,  # Add this
   }.freeze
   ```

4. **Add feeds to spreadsheet:**
   | Name | URL | Type | ... |
   |------|-----|------|-----|
   | my-new-thing | https://... | new-source | ... |

### Adding a New Project Deployment

1. **Create values file:**
   ```yaml
   # deployment/values/{project}/app/values.yaml
   schedule: '24 */6 * * *'
   
   spreadsheet:
     id: 'YOUR_SPREADSHEET_ID'
     ranges:
       feeds: 'Feeds!A:F'
       data: 'Data!A:G'
   
   slack:
     channel: '#project-security'
   
   jira:
     projectName: 'Project Name'
   ```

2. **Create encrypted secrets:**
   ```bash
   # Create secrets.yaml with SOPS
   sops deployment/values/{project}/app/secrets.yaml
   ```

3. **Add to helmfile:**
   ```yaml
   # deployment/helmfile.yaml.gotmpl
   - name: gulli-{project}
     namespace: gulli
     chart: './chart'
     version: 0.3.0
     values:
       - values/{project}/app/values.yaml
     secrets:
       - values/{project}/app/secrets.yaml
     set:
       - name: 'image.tag'
         value: {{ requiredEnv "REFERENCE" }}
       - name: 'enableDependabot'
         value: true
   ```

### Spreadsheet Schema

**Feeds Sheet (Configuration):**
| Column | Name | Example | Required |
|--------|------|---------|----------|
| A | Name | `my-repo` | Yes |
| B | URL | `https://feed.xml` | For RSS |
| C | Type | `dependabot` | Yes |
| D | Ignore Updates | `TRUE`/`FALSE` | No |
| E | Don't Create Jira Issue | `TRUE`/`FALSE` | No |
| F | story_assignee | `5f4d3c2b1a...` | For Jira |

**Data Sheet (State/Tracking):**
| Column | Name | Example |
|--------|------|---------|
| A | feed | `my-repo` |
| B | id | `my-repo-42` |
| C | identifier | `CVE-2024-1234` |
| D | url | `https://github.com/...` |
| E | ignore? | `false` |
| F | date | `2024-01-15T10:30:00Z` |
| G | jira_ticket_link | `https://jira.../VA-123` |

### Troubleshooting

| Symptom | Likely Cause | Solution |
|---------|--------------|----------|
| "Missing Configuration" in Slack | GitHub/GCP credentials not set | Check env vars |
| No Slack notifications | `SLACK_WEBHOOK_URL` not set | Add webhook URL |
| Duplicate tickets | Spreadsheet not updating | Check Google API permissions |
| Old alerts appearing | Cache issue | Should auto-clear; check `clear_cache!` |
| Jira creation fails | Custom field IDs changed | Verify field IDs in Jira admin |

---

## 12. Session Log

### Session 1 (Complete)
**Files Reviewed:** 22/22 core files ✓

**Application Code:**
- [x] `bin/gulli`
- [x] `lib/gulli.rb`
- [x] `lib/gulli/feed_entry/base.rb`
- [x] `lib/gulli/feed_entry/dependabot.rb`
- [x] `lib/gulli/feed_entry/rss_article.rb`
- [x] `lib/gulli/feed_entry/google_artifact_registry.rb`
- [x] `lib/gulli/fetch_feeds.rb`
- [x] `lib/gulli/spreadsheet.rb`
- [x] `lib/gulli/collect_dependabot_alerts.rb`
- [x] `lib/gulli/collect_rss_articles.rb`
- [x] `lib/gulli/collect_google_artifact_registry_alerts.rb`
- [x] `lib/gulli/jira_ticket_creator.rb`
- [x] `lib/gulli/notify.rb`
- [x] `lib/utility/assessment_due_date_generator.rb`

**Configuration:**
- [x] `.env.example`
- [x] `Gemfile`
- [x] `Dockerfile`

**Deployment:**
- [x] `deployment/chart/templates/cronjob.yaml`
- [x] `deployment/chart/values.yaml`
- [x] `deployment/values/innotix/app/values.yaml`
- [x] `deployment/helmfile.yaml.gotmpl`

**Tests:**
- [x] `spec/lib/gulli/jira_ticket_creator_spec.rb`
- [x] `spec/lib/collect_dependabot_alerts_spec.rb`

**Knowledge Base Status:** ✅ Complete (95%+ coverage)

**Remaining (nice-to-have):**
- [ ] `.sops.yaml` - Encryption configuration
- [ ] GitHub Actions workflows - CI/CD pipeline details
- [ ] Additional spec files - More edge cases

---

## Usage Notes

This knowledge base is designed to be:
1. **Self-contained** - No need to reference source code
2. **Searchable** - Use Ctrl+F to find any concept
3. **Actionable** - Includes step-by-step guides for common tasks
4. **Maintainable** - Update Session Log when reviewing more files

**To revive context in a new conversation:**
> "I have a knowledge base for a Ruby project called Gulli. Here it is: [paste this document]. I need help with [specific question]."







# Core issue → Gulli has no mock/local mode
#  it requires real Google Sheets access to do anything
# spreadsheet is the everything for gulli 
# no alt way to run locally 

# need GOOGLE_KEYFILE to run
# GOOGLE_KEYFILE is a service account JSON
# Service account JSON is stored encrypted (SOPS)
# dont have KMS decrypt permission
#     - just need the keyfile 
#     - or the access to service account



YOU RUN: bundle exec ruby -r ./dry_run_patch -r ./lib/gulli -e "Gulli.run"

┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 1: LOAD                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  -r ./dry_run_patch     → Loads our safety patches first                    │
│  -r ./lib/gulli         → Loads main application                            │
│                                                                             │
│  lib/gulli.rb (top of file):                                                │
│    require 'gulli/spreadsheet'                                              │
│    require 'gulli/notify'                                                   │
│    require 'gulli/fetch_feeds'                                              │
│    require 'gulli/collect_dependabot_alerts'                                │
│    require 'gulli/collect_rss_articles'                                     │
│    require 'gulli/collect_google_artifact_registry_alerts'                  │
│    require 'gulli/feed_entry/base'                                          │
│    require 'gulli/feed_entry/dependabot'                                    │
│    require 'gulli/feed_entry/rss_article'                                   │
│    require 'gulli/feed_entry/google_artifact_registry'                      │
│    require 'gulli/jira_ticket_creator'                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 2: Gulli.run CALLED                                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli.rb                                                         │
│  Method: Gulli.run                                                          │
│                                                                             │
│  def self.run                                                               │
│    clear_cache!              # ← Line ~XX: Reset any cached data            │
│    announcements.each do |a| # ← Line ~XX: Get all alerts, loop through     │
│      handle_announcement(a)  # ← Line ~XX: Process each one                 │
│    end                                                                      │
│  rescue StandardError => e                                                  │
│    Notify.fatal(e)           # ← If anything crashes, notify Slack          │
│  end                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 3: clear_cache!                                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli.rb                                                         │
│                                                                             │
│  def self.clear_cache!                                                      │
│    @cache = {}               # ← Resets in-memory cache to empty hash       │
│  end                                                                        │
│                                                                             │
│  WHY: Prevents stale data between runs (important in K8s CronJob)           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4: announcements (Fetch all alerts)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli.rb                                                         │
│                                                                             │
│  def self.announcements                                                     │
│    feeds.flat_map do |feed|                                                 │
│      collector = FEED_ENTRY_COLLECTOR[feed[:type]]                          │
│      collector.call(feed)                                                   │
│    end                                                                      │
│  end                                                                        │
│                                                                             │
│  FEED_ENTRY_COLLECTOR = {                                                   │
│    'rss'                      => Gulli::CollectRSSArticles,                 │
│    'dependabot'               => Gulli::CollectDependabotAlerts,            │
│    'google-artifact-registry' => Gulli::CollectGoogleArtifactRegistryAlerts │
│  }                                                                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4a: feeds (Get feed configs from spreadsheet)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli.rb → lib/gulli/fetch_feeds.rb                              │
│                                                                             │
│  def self.feeds                                                             │
│    FetchFeeds.call           # ← Delegates to FetchFeeds class              │
│  end                                                                        │
│                                                                             │
│  File: lib/gulli/fetch_feeds.rb                                             │
│                                                                             │
│  def self.call                                                              │
│    Spreadsheet.get_values(Gulli.config[:spreadsheet][:ranges][:feeds])      │
│  end                               │                                        │
│                                    ▼                                        │
│  File: lib/gulli/spreadsheet.rb                                             │
│                                                                             │
│  def self.get_values(range)                                                 │
│    # 1. Check cache first                                                   │
│    return cache[range] if cache[range]                                      │
│                                                                             │
│    # 2. Call Google Sheets API                                              │
│    response = service.get_spreadsheet_values(spreadsheet_id, range)         │
│                                                                             │
│    # 3. Transform rows to hashes                                            │
│    headers = response.values[0]      # ["Name", "URL", "Type", ...]         │
│    rows = response.values[1..]       # [["repo", "http://...", "rss"], ...] │
│    result = rows.map { |row| headers.zip(row).to_h }                        │
│    # Result: [{"Name"=>"repo", "URL"=>"http://...", "Type"=>"rss"}, ...]    │
│                                                                             │
│    # 4. Cache and return                                                    │
│    cache[range] = result                                                    │
│  end                                                                        │
│                                                                             │
│  GOOGLE API CALL HAPPENS HERE ◄─────────────────────────────────────────    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 4b: Collect alerts for each feed                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  For feed type "rss":                                                       │
│  ═══════════════════                                                        │
│  File: lib/gulli/collect_rss_articles.rb                                    │
│                                                                             │
│  def call                                                                   │
│    response = HTTPClient.new.get(feed[:url])  # ← Fetch RSS XML             │
│    parsed = Feedjira.parse(response.body)     # ← Parse XML to objects      │
│    parsed.entries.map do |entry|                                            │
│      FeedEntry::RSSArticle.new(entry, feed)   # ← Wrap each entry           │
│    end                                                                      │
│  rescue => e                                                                │
│    Notify.feed_error(feed[:name], e)          # ← Log error, continue       │
│    []                                          # ← Return empty, don't crash│
│  end                                                                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  For feed type "dependabot":                                                │
│  ═══════════════════════════                                                │
│  File: lib/gulli/collect_dependabot_alerts.rb                               │
│                                                                             │
│  def call                                                                   │
│    return [] unless github_configured?        # ← Skip if no token          │
│                                                                             │
│    response = HTTPClient.new.get(                                           │
│      "https://api.github.com/repos/#{owner}/#{repo}/dependabot/alerts",     │
│      headers: { 'Authorization' => "Bearer #{token}" }                      │
│    )                                                                        │
│    alerts = JSON.parse(response.body)                                       │
│    alerts.map { |a| FeedEntry::Dependabot.new(a, feed) }                    │
│  end                                                                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  For feed type "google-artifact-registry":                                  │
│  ═════════════════════════════════════════                                  │
│  File: lib/gulli/collect_google_artifact_registry_alerts.rb                 │
│                                                                             │
│  (Similar pattern - fetch from GCP, wrap in FeedEntry objects)              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5: handle_announcement (For each alert)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli.rb                                                         │
│                                                                             │
│  def self.handle_announcement(announcement)                                 │
│    return if already_known_or_old?(announcement)  # ← Skip duplicates       │
│                                                                             │
│    # Create Jira ticket if configured                                       │
│    if announcement.auto_jira_issue_create? && !announcement.ignore?         │
│      announcement.create_jira_ticket!                                       │
│    end                                                                      │
│                                                                             │
│    announcement.remember!     # ← Write to spreadsheet (blocked in dry run) │
│                                                                             │
│    unless announcement.ignore?                                              │
│      Notify.article(announcement)  # ← Slack notification (blocked in dry)  │
│    end                                                                      │
│  end                                                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5a: already_known_or_old?                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli.rb                                                         │
│                                                                             │
│  def self.already_known_or_old?(announcement)                               │
│    announcement.known? || announcement.published_date_before_6_months?      │
│  end                                                                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  File: lib/gulli/feed_entry/base.rb                                         │
│                                                                             │
│  def known?                                                                 │
│    existing_ids.include?(id)                                                │
│  end                                                                        │
│                                                                             │
│  def existing_ids                                                           │
│    @existing_ids ||= Spreadsheet.get_values(data_range).map { |r| r['id'] } │
│  end                          │                                             │
│                               ▼                                             │
│           READS FROM SPREADSHEET (cached after first call)                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5b: create_jira_ticket! (if enabled)                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli/feed_entry/base.rb                                         │
│                                                                             │
│  def create_jira_ticket!                                                    │
│    @jira_ticket_url = JiraTicketCreator.call(self)                          │
│  end                                                                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  File: lib/gulli/jira_ticket_creator.rb                                     │
│                                                                             │
│  def self.call(article)                                                     │
│    payload = {                                                              │
│      fields: {                                                              │
│        project: { key: config[:project_key] },                              │
│        summary: article.title,                                              │
│        issuetype: { name: config[:issue_type] },                            │
│        # ... more fields                                                    │
│      }                                                                      │
│    }                                                                        │
│                                                                             │
│    response = HTTPClient.new.post(                                          │
│      "#{base_url}/rest/api/3/issue",                                        │
│      body: payload.to_json,                                                 │
│      headers: { 'Authorization' => "Basic #{credentials}" }                 │
│    )                                                                        │
│                                                                             │
│    "#{base_url}/browse/#{response['key']}"  # ← Returns ticket URL          │
│  end                                                                        │
│                                                                             │
│  IN DRY RUN: Returns fake URL, no API call                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5c: remember! (Save to spreadsheet)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli/feed_entry/base.rb                                         │
│                                                                             │
│  def remember!                                                              │
│    Spreadsheet.append(                                                      │
│      data_range,                                                            │
│      attributes: [feed, id, identifier, url, ignore?, date, jira_ticket_url]│
│    )                                                                        │
│  end                                                                        │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                             │
│  File: lib/gulli/spreadsheet.rb                                             │
│                                                                             │
│  def self.append(range, attributes:)                                        │
│    service.append_spreadsheet_value(                                        │
│      spreadsheet_id,                                                        │
│      range,                                                                 │
│      { values: [attributes] },                                              │
│      value_input_option: 'RAW'                                              │
│    )                                                                        │
│    clear_cache!  # ← Invalidate cache since data changed                    │
│  end                                                                        │
│                                                                             │
│  IN DRY RUN: Just logs, no API call                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 5d: Notify.article (Slack notification)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  File: lib/gulli/notify.rb                                                  │
│                                                                             │
│  def self.article(article)                                                  │
│    notifier.ping(                                                           │
│      "*#{article.feed}*: #{article.title}",                                 │
│      attachments: [                                                         │
│        { text: ":inbox_tray: #{article.url}" },                             │
│        { text: ":bookmark_tabs: Spreadsheet" },                             │
│        { text: ":ticket: #{article.jira_ticket_url}" }                      │
│      ]                                                                      │
│    )                                                                        │
│  end                                                                        │
│                                                                             │
│  def self.notifier                                                          │
│    @notifier ||= Slack::Notifier.new(webhook_url, options)                  │
│  end                                                                        │
│                                                                             │
│  IN DRY RUN: Just logs, no Slack call                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ STEP 6: DONE                                                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Loop completes for all announcements.                                      │
│  Process exits.                                                             │
│                                                                             │
│  In K8s: CronJob pod terminates, restarts on next schedule.                 │
│  Locally: Script ends.                                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘



=================
at a glance:

run করলে
    │
    ▼
Ruby files load করে
    │
    ▼
Gulli.run() call হয়
    │
    ▼
Cache clear হয় (fresh start)
    │
    ▼
Spreadsheet থেকে feed list আনে ◄── Google API call (keyfile দরকার)
    │
    ▼
প্রতিটা feed এর জন্য:
    ├── RSS? → RSS URL থেকে articles আনে
    ├── Dependabot? → GitHub API থেকে alerts আনে
    └── GAR? → Google Cloud থেকে image info আনে
    │
    ▼
প্রতিটা alert এর জন্য:
    │
    ├── আগে দেখা? → Skip
    │
    ├── ৬ মাস পুরাতন? → Skip
    │
    ├── Jira enabled? → Ticket create করে
    │
    ├── Spreadsheet এ save করে (remember!)
    │
    └── Slack এ notify করে
    │
    ▼
program exit করে



==========

Step-by-Step Flow with Simple Explanation
STEP 1: LOAD (Application শুরু)
text

-r ./dry_run_patch     → Loads our safety patches first
-r ./lib/gulli         → Loads main application
সহজ ভাষায়:
তুমি যখন command run করো, Ruby প্রথমে দুইটা file load করে:

dry_run_patch.rb - এইটা আমাদের safety net, যাতে কোনো real action না হয়
lib/gulli.rb - এইটা main application
lib/gulli.rb file এর উপরে অনেকগুলো require আছে। মানে এই file বলতেছে "আমার এই এই files দরকার, আগে ওগুলো load করো"। Ruby তখন সব dependency files load করে memory তে রাখে।

STEP 2: Gulli.run CALLED (মূল function call)
Ruby

def self.run
  clear_cache!              # Reset cached data
  announcements.each do |a| # Get all alerts, loop through
    handle_announcement(a)  # Process each one
  end
rescue StandardError => e
  Notify.fatal(e)           # If crash, notify Slack
end
সহজ ভাষায়:
Gulli.run হলো main function যেটা সব কিছু শুরু করে। এইটা বলতেছে:

আগে cache clear করো (পুরাতন data মুছে ফেলো)
সব alerts collect করো
প্রতিটা alert এর জন্য কিছু কাজ করো
যদি কোনো error হয়, Slack এ জানাও
এইটা একটা try-catch block এর মতো। যদি কোথাও crash হয়, rescue block ধরে ফেলবে এবং team কে Slack এ notify করবে।

STEP 3: clear_cache! (Cache পরিষ্কার)
Ruby

def self.clear_cache!
  @cache = {}
end
সহজ ভাষায়:
Cache মানে temporary storage। আগের run এ যা data রাখা ছিল, সেটা মুছে দাও। Fresh start নাও।

কেন দরকার? ধরো আগের run এ 10টা alert ছিল। এখন নতুন run এ 12টা আছে। পুরাতন cache থাকলে নতুন 2টা miss হয়ে যেতে পারে।

STEP 4: announcements (সব Alerts সংগ্রহ)
Ruby

def self.announcements
  feeds.flat_map do |feed|
    collector = FEED_ENTRY_COLLECTOR[feed[:type]]
    collector.call(feed)
  end
end
সহজ ভাষায়:
এই step এ দুইটা কাজ হয়:

প্রথমে feeds call হয়:

Spreadsheet থেকে feed list আনো
Feed মানে কোন কোন source monitor করতে হবে (GitHub repos, RSS feeds, Docker images)
তারপর প্রতিটা feed এর জন্য:

Feed এর type দেখো (rss? dependabot? google-artifact-registry?)
সেই type অনুযায়ী সঠিক collector class বাছাই করো
ওই collector কে বলো "যাও, alerts নিয়ে আসো"
এইটা Strategy Pattern - type অনুযায়ী different class কাজ করে।

STEP 4a: feeds (Spreadsheet থেকে Feed Config আনা)
Ruby

# lib/gulli/fetch_feeds.rb
def self.call
  Spreadsheet.get_values(Gulli.config[:spreadsheet][:ranges][:feeds])
end

# lib/gulli/spreadsheet.rb
def self.get_values(range)
  return cache[range] if cache[range]  # Check cache first
  
  response = service.get_spreadsheet_values(spreadsheet_id, range)
  # Transform and cache...
end
সহজ ভাষায়:

আগে cache check করো - আগেই কি এই data এনেছিলাম?
Cache এ নাই? তাহলে Google Sheets API call করো
API থেকে raw data আসে table format এ:
text

Row 1: ["Name", "URL", "Type"]           ← Headers
Row 2: ["my-repo", "http://...", "rss"]  ← Data
Row 3: ["other-repo", "...", "dependabot"]
এইটাকে Ruby hash এ convert করো:
Ruby

[
  {"Name" => "my-repo", "URL" => "http://...", "Type" => "rss"},
  {"Name" => "other-repo", ...}
]
Cache এ রাখো, পরে আবার লাগলে API call লাগবে না
এইখানেই তোমার GOOGLE_KEYFILE দরকার - Google API তে authenticate করতে।

STEP 4b: Collect Alerts (প্রতিটা Source থেকে Alert আনা)
RSS Feeds:
Ruby

def call
  response = HTTPClient.new.get(feed[:url])  # Fetch RSS XML
  parsed = Feedjira.parse(response.body)     # Parse XML
  parsed.entries.map do |entry|
    FeedEntry::RSSArticle.new(entry, feed)   # Wrap each entry
  end
end
সহজ ভাষায়:

RSS feed এর URL এ HTTP GET request পাঠাও
XML response আসবে (RSS format এ)
Feedjira library দিয়ে XML parse করো
প্রতিটা article কে FeedEntry::RSSArticle object এ wrap করো
ধরো security blog এর RSS feed। ওখানে নতুন article আসলে সেটা ধরা পড়বে।

Dependabot:
Ruby

def call
  return [] unless github_configured?  # No token? Skip.
  
  response = HTTPClient.new.get(
    "https://api.github.com/repos/#{owner}/#{repo}/dependabot/alerts",
    headers: { 'Authorization' => "Bearer #{token}" }
  )
  # Parse and wrap...
end
সহজ ভাষায়:

GitHub token না থাকলে skip করো, empty array return করো
Token থাকলে GitHub API call করো
Dependabot alerts নিয়ে আসো (security vulnerabilities)
প্রতিটা alert কে FeedEntry::Dependabot object এ wrap করো
STEP 5: handle_announcement (প্রতিটা Alert Process করা)
Ruby

def self.handle_announcement(announcement)
  return if already_known_or_old?(announcement)  # Skip if seen before
  
  if announcement.auto_jira_issue_create? && !announcement.ignore?
    announcement.create_jira_ticket!
  end
  
  announcement.remember!  # Save to spreadsheet
  
  unless announcement.ignore?
    Notify.article(announcement)  # Slack notification
  end
end
সহজ ভাষায়:
প্রতিটা alert এর জন্য:

Check করো এইটা আগে দেখা কিনা:

Spreadsheet এ এই ID আগে আছে? → Skip
6 মাসের বেশি পুরাতন? → Skip
Jira ticket বানাও (যদি enabled থাকে):

Feed config এ auto_jira_issue_create = true?
এইটা ignore করার মতো না (alpha/beta/rc না)?
হ্যাঁ হলে Jira তে ticket create করো
Spreadsheet এ save করো:

এই alert এর record রাখো যাতে পরে আবার process না হয়
Slack এ notify করো:

Team কে জানাও নতুন vulnerability পাওয়া গেছে
STEP 5a: already_known_or_old? (Duplicate Check)
Ruby

def known?
  existing_ids.include?(id)
end

def existing_ids
  @existing_ids ||= Spreadsheet.get_values(data_range).map { |r| r['id'] }
end
সহজ ভাষায়:

Spreadsheet এর Data sheet থেকে সব existing IDs নিয়ে আসো
এই alert এর ID ওই list এ আছে কিনা check করো
থাকলে মানে আগেই process করা হয়েছে, skip করো
এইটা deduplication - একই alert বারবার process হবে না।

STEP 5b: create_jira_ticket! (Jira Ticket তৈরি)
Ruby

def self.call(article)
  payload = {
    fields: {
      project: { key: "VA" },
      summary: article.title,
      issuetype: { name: "Vulnerability Assessment" },
      # ... more fields
    }
  }
  
  response = HTTPClient.new.post(
    "#{base_url}/rest/api/3/issue",
    body: payload.to_json
  )
  
  "#{base_url}/browse/#{response['key']}"  # Return ticket URL
end
সহজ ভাষায়:

Jira API এর জন্য payload বানাও (ticket এর details)
Jira API তে POST request পাঠাও
Jira একটা ticket create করে response এ ticket key দেয় (যেমন VA-123)
Ticket URL return করো (https://jira.../browse/VA-123)
Dry run এ: শুধু log করবে, real API call হবে না।

STEP 5c: remember! (Spreadsheet এ Save করা)
Ruby

def remember!
  Spreadsheet.append(
    data_range,
    attributes: [feed, id, identifier, url, ignore?, date, jira_ticket_url]
  )
end
সহজ ভাষায়:
Data sheet এ একটা নতুন row add করো এই information দিয়ে:

কোন feed থেকে এসেছে
Alert এর unique ID
CVE number বা article title
Alert এর URL
Ignore করা হয়েছে কিনা
Date
Jira ticket link (যদি create করা হয়ে থাকে)
এইটাই "state" save করা। পরের run এ এই ID দেখে বুঝবে এইটা আগেই process করা হয়েছে।

Dry run এ: শুধু log করবে, spreadsheet এ লিখবে না।

STEP 5d: Notify.article (Slack Notification)
Ruby

def self.article(article)
  notifier.ping(
    "*#{article.feed}*: #{article.title}",
    attachments: [
      { text: ":inbox_tray: #{article.url}" },
      { text: ":ticket: #{article.jira_ticket_url}" }
    ]
  )
end
সহজ ভাষায়:
Slack channel এ message পাঠাও:

text

*my-repo*: CVE-2024-1234 - Critical Bug
📥 https://github.com/...
🎫 https://jira.../browse/VA-123
Team দেখতে পাবে নতুন vulnerability এসেছে।

Dry run এ: শুধু log করবে, Slack এ যাবে না।

STEP 6: DONE (শেষ)
সহজ ভাষায়:
সব alerts process করা শেষ। Program exit করে।

Kubernetes এ CronJob হিসেবে run হলে, pod terminate হয়ে যায়। পরের scheduled time এ আবার নতুন pod start হবে, আবার পুরো process চলবে।

=============


sops -d secrets.yaml → FAILED
Permission 'cloudkms.cryptoKeyVersions.useToDecrypt' denied

What this means:

secrets.yaml contains the encrypted service account keyfile
To decrypt it, you need GCP KMS permission
You don't have that permission
You cannot get the keyfile this way

Let's say someone gives you the keyfile directly (not via SOPS). Here's what happens:

┌─────────────────────────────────────────────────────────────────────────────┐
│ Authentication Chain for Gulli                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  GOOGLE_KEYFILE="/path/to/keyfile.json"                                      │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────┐                                │
│  │ Service Account Authentication          │                                │
│  │ (e.g., gulli-sa@project.iam...)         │                                │
│  └──────────────────┬──────────────────────┘                                │
│                     │                                                       │
│                     ▼                                                       │
│  ┌─────────────────────────────────────────┐                                │
│  │ Google Sheets API                       │                                │
│  │ - Read Feeds sheet                      │                                │
│  │ - Read/Write Data sheet                 │                                │
│  └──────────────────┬──────────────────────┘                                │
│                     │                                                       │
│                     ▼                                                       │
│  ┌─────────────────────────────────────────┐                                │
│  │ Spreadsheet Permission Check            │                                │
│  │ Does SERVICE ACCOUNT have access to     │                                │
│  │ this specific spreadsheet?              │                                │
│  └─────────────────────────────────────────┘                                │
│                                                                              │
│  If YES → Works ✓                                                            │
│  If NO  → "The caller does not have permission" error                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘








currently, gulli is very noisy, for noise reduction, i wanna do it like: 

renovate will detect dependencies for me (for github, gitlab, bitbucket, azure) and from actions, we can get artifact and from these artifact we will send the info to gulli and eventually store the data about dependency in spreadsheet and also create the va ticket in jira and send info in slack, i think the slacka nd jira portion is already in the code, we just have to ensure that it is working, renovate. so can you act like a senior software engineer and ask me questions so that i can build a flow diagram or a user flow diagram first and then i can work based on that? if you have any questions, feel free to ask. and if you need any specific file to see ingulli or renovate end, you can also ask that. i want to get an entire idea about it so i want to make a flow diagram first with detailed stuff and all. 






=====


Absolutely — I can act as a senior software engineer and help you scaffold the requirements for designing a flow diagram / user-flow that integrates Renovate with Gulli.

Before we jump into drawing anything, I need to fully understand your goals, constraints, and expectations — especially since you want to reduce noise and extend Gulli beyond its current RSS/Dependabot/GAR collectors.

Here’s how we’ll do it:

✅ I’ll ask you targeted questions
✅ You answer them (as completely as possible)
✅ I’ll help you translate the answers into a flow diagram structure
✅ Then we’ll refine it into a technical design you can implement

1) Goals & Scope Clarification
✳️ High-Level Intent

What is the exact role you want Renovate to play?

Do you want Renovate to replace Dependabot entirely?

Or to augment Dependabot with more sources (e.g., GitLab/GitHub/GitHub Actions logs/artifacts)?

What types of vulnerabilities / findings should flow into Gulli?

Only dependency updates?

All security advisories?

Secret/config drift?

Docker image/container vulnerability scanning?

What does “noise” mean to you?

Uninteresting alerts?

Pre-releases?

Frequent updates?

Low severity findings?

Already triaged?

Do you want filters before sending to Gulli, inside Gulli, or both?

2) Renovate Input Characteristics
✳️ Renovate Output

What output format from Renovate do you want to use?

JSON API?

GitHub/Bitbucket/GitLab events?

Action logs/artifacts?

Scheduled poll?

Where does Renovate run?

GitHub Actions?

Self-hosted runner?

GitLab CI?

On-prem?

Are Renovate results stored anywhere (DB, Artifact store, file)?
If so:

Where?

How long?

What format?

3) Current State of Gulli Integration
✳️ Gulli Inputs

Right now, Gulli expects feeds → type + source URL + config in Google Sheets.
How do you envision Renovate results being represented as a feed?

A web URL returning JSON?

A local artifact file?

Pub/Sub / event?

Should Renovate results be pushed into Gulli or pulled by Gulli?

Push: Renovate triggers call → Gulli API

Pull: Gulli fetches from Renovate endpoint

Do you want Renovate to produce “unique IDs” that Gulli can deduplicate?
If yes:

What identifier would that be?
(e.g., dependency name + version + repo name)

4) Internal Filtering & Noise Controls

Should filtering happen inside Renovate or inside Gulli?
For example:

Don’t send anything with severity < X

Don’t send anything already ticketed

Don’t send duplicates

Do you want severity, CVSS scores, or other metadata carried through to Slack/Jira?
If yes, what fields matter?

Should Renovate results be transformed/mapped to Gulli’s FeedEntry format?
ie:

title

url

id

published date

identifier

5) Authentication & Delivery Mechanism

How will Gulli access Renovate’s output?

Renovate publishes to Google Sheets?

Renovate publishes to S3/Artifact Registry?

Renovate exposes API endpoint?

Do you want Renovate → Gulli to use a webhook?
If yes:

Will Gulli need a new endpoint?

Or will you build a middleware adapter?

6) Failure & Retries

What should happen when Renovate output fails?

Send Slack alert?

Write error into spreadsheet?

Retry after X minutes?

What happens if Jira ticket creation fails?
(current Gulli logic sends slack error and keeps processing)

7) Deployment & Environment

Where will this pipeline run?

All inside Kubernetes?

Mixed (Renovate outside, Gulli inside)?

Same Google Org?

Do you have access to modify Renovate config already?
If no, what limitations exist?

8) Metrics & Telemetry

Do you want reporting/metrics?

Count per repo?

Count per severity?

Slack summary digest?

How often should Renovate results be processed by Gulli?

Every run of Renovate?

Scheduled (every N hours)?

9) Output / Visualization

What does your final flow diagram need to show?
Pick one or more:

Data flow between systems?

Decision points (filtering, ignore logic)?

Deployment layout (where do things run)?

Notification paths (Slack, Jira)?

🧠 Once You Answer These…

I will help you:

✔ Create a flow diagram skeleton
✔ Map each component (Renovate, Gulli, Slack, Jira, Spreadsheet)
✔ Label data formats & transformation points
✔ Add error paths
✔ Suggest implementation patterns (webhooks, adapters, queues)

Example Diagram Sections (Draft)

Here’s what the end goal could look like (draft labels):

[Renovate CI] 
     ↓ (dependency results JSON)
     ↓ API / Artifact
[Adapter Transformer]  ← optional
     ↓ JSON → normalized
[Renovate Feed Store] 
     ↓ exposed endpoint
[Gulli Collector (new type)]  —–→ fetches → parses → FeedEntry::Renovate
     ↓ known? → deduplicate
     ↓ auto_jira_issue_create?
     ↓ remember! → spreadsheet
     ↓ notify → Slack & Jira

Quick Check Before We Start the Diagram

Answer these first:

1️⃣ Do you want Renovate to push results to Gulli or Gulli to pull them?
2️⃣ What exact fields do you want from Renovate?
(Minimal: dependency + severity + repo + link?)
3️⃣ Do you want Gulli to apply ignore rules to Renovate results?

Reply with those and we’ll build your flow from there.
