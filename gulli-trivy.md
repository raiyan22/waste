
[Kubernetes CronJob প্রতিদিন/প্রতি ঘণ্টায় চালায়]
        │
        ▼
    bin/gulli
        │  শুধু একটাই কাজ — Gulli.run কে call করে
        │  এরপর থেকে সব কিছু Gulli.run এর ভেতরে ঘটে
        ▼
╔═══════════════════════════════════════════════════════════════════════╗
║  lib/gulli.rb — Gulli.run                                             ║
║  পুরো system এর main orchestrator                                     ║
╚═══════════════════════════════════════════════════════════════════════╝
        │
        ├── ধাপ ১: clear_cache!
        │         Gulli.cache = {}
        │         আগের run এ যা কিছু memory তে ছিল সব মুছে দাও
        │         কারণ: পুরনো cached data এই run কে প্রভাবিত করতে পারবে না
        │
        ├── ধাপ ২: feeds (FetchFeeds.call)
        │         ┌──────────────────────────────────────────────────────┐
        │         │ lib/gulli/fetch_feeds.rb                             │
        │         │                                                      │
        │         │ Google Spreadsheet এর Feeds!A:F sheet পড়ো           │
        │         │ প্রতিটা row একটা feed config hash হয়ে আসে:          │
        │         │ {                                                    │
        │         │   name:                   "my-service",             │
        │         │   url:                    "https://...",            │
        │         │   type:                   "trivy",  ← key field     │
        │         │   ignore_updates:          false,                   │
        │         │   auto_jira_issue_create:  true,                    │
        │         │   jira_ticket_assignee:    nil                      │
        │         │ }                                                    │
        │         │                                                      │
        │         │ type field দেখে Gulli.run ঠিক করে                   │
        │         │ কোন path এ যাবে                                      │
        │         └──────────────────────────────────────────────────────┘
        │
        │
        ╔═══════════════════════════════════════════════════════════════╗
        ║  PATH A — rss, dependabot, google-artifact-registry           ║
        ║  announcements method → প্রতিটা entry আলাদাভাবে process হয়   ║
        ╚═══════════════════════════════════════════════════════════════╝
        │
        │  FEED_ENTRY_COLLECTOR = {
        │    'rss'                      => CollectRSSArticles,
        │    'dependabot'               => CollectDependabotAlerts,
        │    'google-artifact-registry' => CollectGoogleArtifactRegistryAlerts,
        │  }
        │  এই Hash দিয়ে type → collector class map হয়
        │  'trivy' এখানে নেই — সে আলাদা path এ যায়
        │
        │  ┌────────────────────────────────────────────────────────────┐
        │  │ type = 'rss'                                               │
        │  │ CollectRSSArticles.call(feed)                              │
        │  │   httpclient দিয়ে RSS URL এ GET request                   │
        │  │   Feedjira দিয়ে XML parse করো                             │
        │  │   প্রতিটা item → FeedEntry::RSSArticle                    │
        │  │   → [FeedEntry::RSSArticle, ...]                          │
        │  └────────────────────────────────────────────────────────────┘
        │
        │  ┌────────────────────────────────────────────────────────────┐
        │  │ type = 'dependabot'                                        │
        │  │ CollectDependabotAlerts.call(feed)                         │
        │  │   GitHub REST API তে authenticated GET request             │
        │  │   JSON parse করো                                          │
        │  │   প্রতিটা alert → FeedEntry::Dependabot                   │
        │  │   → [FeedEntry::Dependabot, ...]                          │
        │  └────────────────────────────────────────────────────────────┘
        │
        │  ┌────────────────────────────────────────────────────────────┐
        │  │ type = 'google-artifact-registry'                          │
        │  │ CollectGoogleArtifactRegistryAlerts.call(feed)             │
        │  │   GCP Container Analysis API তে authenticated GET request  │
        │  │   JSON parse করো                                          │
        │  │   প্রতিটা occurrence → FeedEntry::GoogleArtifactRegistry   │
        │  │   → [FeedEntry::GoogleArtifactRegistry, ...]              │
        │  └────────────────────────────────────────────────────────────┘
        │
        │  উপরের collector গুলো থেকে পাওয়া সব entries এর জন্য
        │  একটা একটা করে handle_announcement(entry) call হয়:
        │
        │  handle_announcement(entry)         ← lib/gulli.rb
        │    │
        │    ├── already_known_or_old?(announcement)
        │    │     │
        │    │     ├── announcement.known?
        │    │     │     Spreadsheet এর Data sheet পড়ো
        │    │     │     এই entry র id আগে লেখা আছে কিনা দেখো
        │    │     │     থাকলে → true, পুরো handle_announcement থেকে return
        │    │     │
        │    │     └── announcement.published_date_before_6_months?
        │    │           entry র date ৬ মাসের বেশি পুরনো কিনা দেখো
        │    │           পুরনো হলে → true, পুরো handle_announcement থেকে return
        │    │
        │    ├── auto_jira_issue_create? && !ignore? → true হলে:
        │    │     │
        │    │     └── announcement.create_jira_ticket!(article: announcement)
        │    │           │
        │    │           ▼
        │    │         ┌─────────────────────────────────────────────────┐
        │    │         │ lib/gulli/jira_ticket_creator.rb                │
        │    │         │ JiraTicketCreator.call(article:)                │
        │    │         │                                                 │
        │    │         │ payload তৈরি করো:                              │
        │    │         │   summary    → article.title                   │
        │    │         │   feed       → article.feed                    │
        │    │         │   url        → article.url                     │
        │    │         │   due_date   → AssessmentDueDateGenerator.call │
        │    │         │   description→ build_description(article)      │
        │    │         │                                                 │
        │    │         │ build_description:                              │
        │    │         │   article.respond_to?(:description)?           │
        │    │         │   Path A entries implement করে না              │
        │    │         │   তাই nil return → payload থেকে বাদ           │
        │    │         │                                                 │
        │    │         │ Jira REST API POST → ticket তৈরি              │
        │    │         │ → ticket URL return করে বা nil                 │
        │    │         └─────────────────────────────────────────────────┘
        │    │
        │    ├── announcement.remember!
        │    │     Data sheet এ একটা row append করো
        │    │     পরেরবার known? এই entry কে চিনবে
        │    │     (প্রতিটা entry তে আলাদা API call)
        │    │
        │    └── Gulli::Notify.article(announcement)
        │          Slack এ message পাঠাও
        │          feed নাম + URL + spreadsheet link + jira link
        │
        │
        ╔═══════════════════════════════════════════════════════════════╗
        ║  PATH B — bucket (Renovate)                                   ║
        ║  process_buckets → Bulk dedup + Bulk write                    ║
        ╚═══════════════════════════════════════════════════════════════╝
        │
        │  process_buckets                    ← lib/gulli.rb
        │    │
        │    └── handle_bucket_findings_batch
        │          │
        │          ├── find_new_bucket_findings
        │          │     │
        │          │     ├── CollectBucketReports.call
        │          │     │     GCS_KEYFILE দিয়ে authenticate
        │          │     │     GCS_BUCKET_NAME + GCS_BUCKET_PREFIX থেকে
        │          │     │     Renovate JSON download করো
        │          │     │     → [FeedEntry::BucketFinding, ...]
        │          │     │
        │          │     ├── load_known_ids          ← lib/gulli.rb
        │          │     │     Spreadsheet.get_values(Data range) call করো
        │          │     │     সব row এর 'id' column বের করো → Set বানাও
        │          │     │     এই Set দিয়ে O(1) lookup সম্ভব
        │          │     │
        │          │     └── findings.reject { known_ids.include?(finding.id) }
        │          │           Set এ আছে এমন id গুলো বাদ দাও
        │          │           কোনো API call নেই — শুধু memory lookup
        │          │           → নতুন findings শুধু
        │          │
        │          ├── প্রতিটা new finding এর জন্য:
        │          │   process_bucket_finding(finding)   ← lib/gulli.rb
        │          │     ├── JiraTicketCreator.call (enable থাকলে)
        │          │     └── Notify.article → Slack
        │          │
        │          └── batch_remember!(new_findings)     ← lib/gulli.rb
        │                findings গুলো থেকে row array বানাও:
        │                [feed, id, identifier, url, '', date, '']
        │                Spreadsheet.batch_append → একটাই API call এ সব লেখো
        │
        │
        ╔═══════════════════════════════════════════════════════════════╗
        ║  PATH B — trivy (নতুন)                                        ║
        ║  process_trivy → Bulk dedup + Bulk write                      ║
        ╚═══════════════════════════════════════════════════════════════╝
        │
        └── process_trivy                     ← lib/gulli.rb
              │
              ├── trivy_feeds                 ← lib/gulli.rb
              │     feeds.select { |f| f[:type] == 'trivy' }
              │     Feeds sheet এ type = 'trivy' এমন row গুলো filter করো
              │     প্রতিটা row একটা আলাদা trivy feed config
              │
              └── প্রতিটা trivy feed এর জন্য:
                  handle_trivy_findings_batch(feed)   ← lib/gulli.rb
                    │
                    ├── CollectTrivyAlerts.call(feed)
                    │     │
                    │     ▼
                    │   ┌──────────────────────────────────────────────────┐
                    │   │ lib/gulli/collect_trivy_alerts.rb                │
                    │   │ CollectTrivyAlerts#call                          │
                    │   │                                                  │
                    │   │ configured? চেক করো:                            │
                    │   │   GCS_BUCKET_NAME আছে?                          │
                    │   │   GCS_BUCKET_PREFIX আছে?                        │
                    │   │   GOOGLE_KEYFILE আছে?                           │
                    │   │   না থাকলে → [] return করো                      │
                    │   │                                                  │
                    │   │ gcs_object_path বানাও:                          │
                    │   │   prefix = GCS_BUCKET_PREFIX                    │
                    │   │   "#{prefix}/trivy-report.json"                 │
                    │   │                                                  │
                    │   │ gcs_service তৈরি করো:                           │
                    │   │   Google::Apis::StorageV1::StorageService        │
                    │   │   GOOGLE_KEYFILE দিয়ে authenticate               │
                    │   │   (Sheets এর মতো একই keyfile — ইচ্ছাকৃত)       │
                    │   │   scope: devstorage.read_only                   │
                    │   │                                                  │
                    │   │ read_report:                                     │
                    │   │   StringIO buffer এ file download করো           │
                    │   │   error হলে → nil return, [] দিয়ে বের হও       │
                    │   │                                                  │
                    │   │ parse_report(content):                           │
                    │   │   JSON.parse করো                                │
                    │   │   data['Results'] iterate করো                   │
                    │   │   প্রতিটা Result এর জন্য:                       │
                    │   │     target = result['Target']                   │
                    │   │     ← "python3.11 (debian 12.10)" এরকম         │
                    │   │     result['Vulnerabilities'].each:              │
                    │   │       build_entry(vuln, target) call করো        │
                    │   │                                                  │
                    │   │ build_entry(vuln, target):                       │
                    │   │   vuln.merge('__target__' => target)            │
                    │   │   ← target টা vuln hash এ inject করো            │
                    │   │   কারণ: raw vuln hash এ target নেই             │
                    │   │   কিন্তু FeedEntry::Trivy এর দরকার             │
                    │   │   FeedEntry::Trivy.new(                         │
                    │   │     feed:  @name,                               │
                    │   │     entry: merged_vuln_hash                     │
                    │   │   )                                             │
                    │   │                                                  │
                    │   │ → [FeedEntry::Trivy, ...] return করো            │
                    │   └──────────────────────────────────────────────────┘
                    │     │
                    │     ▼
                    │   প্রতিটা FeedEntry::Trivy object এ:
                    │   ┌──────────────────────────────────────────────────┐
                    │   │ lib/gulli/feed_entry/trivy.rb                    │
                    │   │ FeedEntry::Trivy                                 │
                    │   │                                                  │
                    │   │ @feed  = feed[:name]  e.g. "my-service"         │
                    │   │ @entry = vuln hash with __target__ injected      │
                    │   │                                                  │
                    │   │ id:                                              │
                    │   │   "trivy-{feed}-{CVE_ID}-{pkg}-{version}"       │
                    │   │   e.g. "trivy-my-service-CVE-2023-1234-         │
                    │   │         openssl-1.1.1"                          │
                    │   │   sanitized_pkg_name: @, /, : replace করে      │
                    │   │   এই id stable — same vuln = same id সবসময়    │
                    │   │                                                  │
                    │   │ title:                                           │
                    │   │   "[my-service] CVE-2023-1234 -                 │
                    │   │    openssl 1.1.1 → 1.1.1t (HIGH)"              │
                    │   │   Jira ticket এর summary হিসেবে যায়             │
                    │   │                                                  │
                    │   │ url:                                             │
                    │   │   entry['PrimaryURL']                           │
                    │   │   fallback: avd.aquasec.com/nvd/{cve_id}       │
                    │   │                                                  │
                    │   │ identifier:                                      │
                    │   │   entry['VulnerabilityID'] → "CVE-2023-1234"   │
                    │   │                                                  │
                    │   │ date:                                            │
                    │   │   entry['LastModifiedDate']                     │
                    │   │   fallback: PublishedDate → Time.now            │
                    │   │                                                  │
                    │   │ ignore?       → সবসময় false                    │
                    │   │ published_date_before_6_months? → সবসময় false  │
                    │   │ auto_jira_issue_create? → সবসময় true           │
                    │   │                                                  │
                    │   │ description:  ← এটাই Trivy কে special করে      │
                    │   │   {                                             │
                    │   │     type: 'doc', version: 1,                   │
                    │   │     content: description_rows                   │
                    │   │   }                                             │
                    │   │   description_rows → adf_row() দিয়ে বানানো:   │
                    │   │   • Vulnerability: CVE-2023-1234               │
                    │   │   • Package:       openssl                      │
                    │   │   • Installed:     1.1.1                        │
                    │   │   • Fixed:         1.1.1t (না থাকলে            │
                    │   │                    "No fix available")          │
                    │   │   • Severity:      HIGH                         │
                    │   │   • Target:        entry['__target__']          │
                    │   │                    ← collector inject করেছিল   │
                    │   │   • CVSS Score:    entry.dig('CVSS','nvd',     │
                    │   │                    'V3Score') বা N/A            │
                    │   │   • URL:           PrimaryURL                   │
                    │   │                                                  │
                    │   │   প্রতিটা row ADF paragraph format এ:           │
                    │   │   { type: 'paragraph', content: [              │
                    │   │       { text: "Label: ", marks: [strong] },    │
                    │   │       { text: "value" }                         │
                    │   │   ]}                                            │
                    │   └──────────────────────────────────────────────────┘
                    │
                    ├── load_known_ids              ← lib/gulli.rb
                    │     Spreadsheet.get_values(Data range)
                    │     সব row এর 'id' field বের করো → Set বানাও
                    │     Bucket আগে run হলে এই Set cache এ আছে → instant
                    │     না হলে fresh API call → cache এ রাখো
                    │
                    ├── findings.reject { known_ids.include?(finding.id) }
                    │     Set lookup — কোনো API call নেই
                    │     id format: "trivy-{feed}-{CVE}-{pkg}-{version}"
                    │     আগে দেখা vulnerability গুলো বাদ পড়ে যায়
                    │     → শুধু নতুন findings থাকে
                    │
                    │     new_findings empty? → return, আর কিছু করো না
                    │
                    ├── new_findings.each
                    │   └── process_trivy_finding(finding)  ← lib/gulli.rb
                    │         │
                    │         ├── finding.auto_jira_issue_create? → true
                    │         │   && !finding.ignore? → true (সবসময়)
                    │         │   তাই সবসময় Jira ticket তৈরি হয়
                    │         │   │
                    │         │   finding.create_jira_ticket!(article: finding)
                    │         │     │
                    │         │     ▼
                    │         │   ┌────────────────────────────────────────┐
                    │         │   │ lib/gulli/jira_ticket_creator.rb       │
                    │         │   │ JiraTicketCreator.call(article:)       │
                    │         │   │                                        │
                    │         │   │ payload তৈরি করো:                     │
                    │         │   │   summary → finding.title             │
                    │         │   │     "[my-service] CVE-2023-1234 -     │
                    │         │   │      openssl 1.1.1 → 1.1.1t (HIGH)"  │
                    │         │   │   feed    → finding.feed             │
                    │         │   │   url     → finding.url              │
                    │         │   │   due_date→ AssessmentDueDateGenerator│
                    │         │   │             .call                     │
                    │         │   │             (Dhaka time অনুযায়ী)     │
                    │         │   │                                        │
                    │         │   │ build_description(finding):           │
                    │         │   │   finding.respond_to?(:description)   │
                    │         │   │   → TRUE! Trivy implement করে         │
                    │         │   │   finding.description call হয়         │
                    │         │   │   → ADF doc object return হয়          │
                    │         │   │   payload এ description field যোগ হয় │
                    │         │   │                                        │
                    │         │   │ Jira REST API POST:                   │
                    │         │   │   /rest/api/3/issue                   │
                    │         │   │   Basic auth header                   │
                    │         │   │   Content-Type: application/json      │
                    │         │   │   body: payload.to_json               │
                    │         │   │                                        │
                    │         │   │ response.status == 201?               │
                    │         │   │   → ticket URL return করো            │
                    │         │   │     finding.jira_ticket_link এ store  │
                    │         │   │   না হলে → handle_failure             │
                    │         │   │     Notify.fatal → Slack              │
                    │         │   │     nil return করো                    │
                    │         │   └────────────────────────────────────────┘
                    │         │
                    │         └── Gulli::Notify.article(finding)
                    │               Slack এ message পাঠাও:
                    │               feed নাম + CVE URL +
                    │               spreadsheet link + jira ticket link
                    │
                    └── batch_remember_trivy!(new_findings)  ← lib/gulli.rb
                          │
                          │ প্রতিটা finding থেকে row বানাও:
                          │ [
                          │   finding.feed,            ← "my-service"
                          │   finding.id,              ← "trivy-my-service-CVE-..."
                          │   finding.identifier,      ← "CVE-2023-1234"
                          │   finding.url,             ← PrimaryURL
                          │   finding.ignore?,         ← false
                          │   finding.date,            ← LastModifiedDate
                          │   finding.jira_ticket_link ← Jira URL বা nil
                          │ ]
                          │
                          └── Spreadsheet.batch_append(data_range, rows)
                                সব rows একটাই Google Sheets API call এ লেখো
                                পরের run এ load_known_ids এই id গুলো পাবে
                                তাই same vulnerability আর process হবে না


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Supporting Layer — উপরের সব কিছু এদের call করে, এরা একে অপরকে না
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  lib/gulli/spreadsheet.rb — Gulli::Spreadsheet
  ┌────────────────────────────────────────────────────────────────────┐
  │ .get_values(range)   → Data sheet পড়ো, row array return করো      │
  │ .known?(id)          → Path A dedup, id আছে কিনা দেখো            │
  │ .remember!(entry)    → Path A, একটা row append করো               │
  │ .batch_append(range, rows) → Path B, একটা API call এ সব লেখো    │
  └────────────────────────────────────────────────────────────────────┘

  lib/gulli/jira_ticket_creator.rb — Gulli::JiraTicketCreator
  ┌────────────────────────────────────────────────────────────────────┐
  │ .call(article:)      → Jira ticket তৈরি করো, URL বা nil return   │
  │ build_description    → respond_to?(:description) guard দিয়ে      │
  │                        ADF object অথবা nil return করে            │
  │                        Trivy → ADF পায়                            │
  │                        অন্যরা → nil, description field বাদ যায়  │
  └────────────────────────────────────────────────────────────────────┘

  lib/gulli/notify.rb — Gulli::Notify
  ┌────────────────────────────────────────────────────────────────────┐
  │ .article(article)    → নতুন finding এর Slack message              │
  │ .fatal(msg)          → Jira failure বা crash এর Slack message     │
  └────────────────────────────────────────────────────────────────────┘

  lib/utility/assessment_due_date_generator.rb
  ┌────────────────────────────────────────────────────────────────────┐
  │ .call → Asia/Dhaka timezone এ এখন কটা বাজে দেখো                 │
  │         ১০টার আগে → আজকের date                                    │
  │         ১০টার পরে → কালকের date                                   │
  │         শনিবার পড়লে → রবিবার                                      │
  │         "YYYY-MM-DD" string return করো → Jira due date হয়        │
  └────────────────────────────────────────────────────────────────────┘


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Path A বনাম Path B — কেন দুটো আলাদা রাখা হয়েছে
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Path A (rss, dependabot, gar):
    প্রতিটা entry → Spreadsheet read (known?) → Spreadsheet write (remember!)
    ৫০টা entry = ১০০টা Google API call
    ছোট result set এর জন্য acceptable

  Path B (bucket, trivy):
    একবার পুরো sheet পড়ো → Set বানাও → সব new row একটা call এ লেখো
    ৫০০টা entry = মাত্র ২টা Google API call
    Trivy scan এ ১০০+ vulnerability থাকা স্বাভাবিক
    Path A ব্যবহার করলে Google Sheets API rate limit এ পড়ে যেত

  Trivy তে __target__ injection কেন দরকার:
    Trivy JSON structure:
      Results[]
        └── Result
              ├── Target: "python3.11 (debian 12.10)"  ← parent এ
              └── Vulnerabilities[]
                    └── { VulnerabilityID, PkgName, ... }  ← child এ Target নেই

    FeedEntry::Trivy কে জানতে হবে কোন image/OS এর vulnerability
    কিন্তু raw vuln hash এ সেটা নেই
    তাই CollectTrivyAlerts → build_entry এ:
      vuln.merge('__target__' => target)
    এরপর FeedEntry::Trivy → entry['__target__'] দিয়ে access করে
    Jira description এর Target field এ দেখায়

  description field কেন শুধু Trivy তে:
    JiraTicketCreator → build_description:
      article.respond_to?(:description) → check করো
      FeedEntry::RSSArticle  → implement করেনি → nil → field বাদ
      FeedEntry::Dependabot  → implement করেনি → nil → field বাদ
      FeedEntry::Trivy       → implement করেছে → ADF object → field যোগ
    এই guard এর কারণে পুরনো FeedEntry গুলো কোনো change ছাড়াই কাজ করে
