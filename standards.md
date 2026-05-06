# Standards & API Reference

> Project: Influencer Marketing Platform · Generated: 2026-05-06

## Industry Standards & Specifications

### Regulatory & Disclosure Standards

**FTC Endorsement Guidelines (16 CFR Part 255)**
- Official URL: https://www.ftc.gov/business-guidance/advertising-marketing/endorsements-influencers-reviews
- Requires influencers to clearly and conspicuously disclose material connections (payment, free products, family relationships) to brands. As of 2025–2026, the FTC mandates disclosure must appear above the fold in video captions, be in readable font, and use unambiguous language ("Ad", "Paid Partnership") — platform tagging tools alone are insufficient. Maximum civil penalty: $53,088 per violation (adjusted annually for inflation). Any influencer platform must integrate compliance checks that verify FTC-compliant disclosures are present in published content.

**FTC Disclosures 101 for Social Media Influencers**
- Official URL: https://www.ftc.gov/business-guidance/resources/disclosures-101-social-media-influencers
- Companion guidance document detailing format requirements for specific content types including Reels, TikTok short-form video, and livestreams. Influencer platforms must account for format-specific disclosure placement rules when running automated compliance checks.

**ASA CAP Code (UK) — CAP Code Section 2**
- Official URL: https://www.asa.org.uk/codes-and-rulings/advertising-codes/non-broadcast-code.html
- The UK Advertising Standards Authority's Committee of Advertising Practice Code requires all commercial communications to be identifiable as advertising. Applies to UK-based creators and UK-targeted campaigns. Influencer platforms operating in the UK must enforce labelling of paid content, equivalent to the FTC's requirements but within the ASA's regulatory framework.

**GDPR — General Data Protection Regulation (EU) 2016/679**
- Official URL: https://gdpr-info.eu/
- Governs collection, storage, and processing of personal data for EU residents. Directly relevant to influencer platforms that build audience demographic profiles, collect creator contact data during outreach, and store campaign performance data. Platforms must implement explicit consent flows, data residency controls (EU data must remain in EU), the right to erasure, and data minimization principles. Any audience analytics feature must be designed to use only lawfully obtained, consented data.

**CCPA — California Consumer Privacy Act (Cal. Civ. Code §1798.100)**
- Official URL: https://oag.ca.gov/privacy/ccpa
- Grants California residents rights to know what personal data is collected, opt out of its sale, and request deletion. Relevant when collecting creator data (email addresses, social profile data, earnings data) from California-based influencers, and when building audience demographic profiles for California audiences. Platforms must provide a "Do Not Sell My Personal Information" mechanism and honor deletion requests within 45 days.

### Advertising & Measurement Standards

**IAB Creator Economy Definitions and Taxonomy (April 2025)**
- Official URL: https://www.iab.com/guidelines/creator-economy-definitions-and-taxonomy/
- Full document: https://www.iab.com/wp-content/uploads/2025/04/IAB_Creator_Economy_Definitions_Taxonomy_April-2025.pdf
- IAB's canonical shared vocabulary for the creator economy, establishing standardized definitions for terms including influencer marketing hub, performance-based pricing, platform payout, and creator categories. Any influencer platform should align its internal taxonomy (creator tiers, campaign types, pricing models) to this standard to enable cross-platform comparability and interoperability with ad tech partners.

**IAB/MRC Attention Measurement Guidelines (November 2025)**
- Official URL: https://www.iab.com/wp-content/uploads/2025/11/IAB_MRC_Attention_Measurement_Guidelines_November_2025.pdf
- Joint IAB and Media Rating Council framework for measuring attention across digital and cross-media environments, covering data signals, visual tracking, physiological measurement, and panel-based methodologies. Relevant to influencer platforms building post-campaign ROI attribution and engagement quality measurement features.

**MRC Minimum Standards for Media Rating Research**
- Official URL: https://www.mediaratingcouncil.org/standards-and-guidelines
- The Media Rating Council establishes accreditation standards for digital advertising measurement products. Influencer platforms offering reach, engagement, and conversion metrics should align measurement methodology with MRC standards to enable agency and enterprise customers to include influencer data in cross-media planning tools.

**IAB Guidelines for Incremental Measurement in Commerce Media**
- Official URL: https://www.iab.com/guidelines/guidelines-for-incremental-measurement-in-commerce-media/
- Relevant for influencer platforms offering sales attribution and e-commerce conversion tracking. Defines how incremental impact should be measured and reported in commerce media campaigns, directly applicable to influencer-driven Shopify and Amazon Attribution integrations.

### Platform Terms of Service (De-Facto Standards)

**Meta Branded Content Policies (Instagram)**
- Official URL: https://www.facebook.com/policies/brandedcontent/
- Instagram requires that branded content be tagged using its native "Paid Partnership" label tool when posts involve material connections between creators and brands. Influencer platforms accessing Instagram data must comply with the Meta Platform Terms and Instagram API Terms, which prohibit scraping, require use of official APIs, and restrict storage of personal data obtained via Graph API.

**TikTok Branded Content Policy**
- Official URL: https://www.tiktok.com/community-guidelines/en/branded-content/
- TikTok requires creators to enable its native branded content toggle when posting sponsored content and prohibits misleading or undisclosed advertising. Influencer platforms must use TikTok's Creator Marketplace API or Business API for data access rather than unauthorized scraping.

**YouTube Paid Product Promotion Policy**
- Official URL: https://support.google.com/youtube/answer/154235
- YouTube creators must disclose paid product promotion using YouTube's native disclosure feature for any video containing paid endorsements. Applies to influencer platform compliance monitoring for YouTube-based campaigns.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)**
- Official URL: https://www.rfc-editor.org/rfc/rfc6749
- The industry-standard authorization protocol for granting third-party applications access to user accounts on social platforms (Instagram, TikTok, YouTube, LinkedIn) without exposing credentials. All major platform APIs (Instagram Graph API, YouTube Data API, LinkedIn Marketing API, TikTok for Developers) use OAuth 2.0 flows. Influencer platforms must implement OAuth 2.0 authorization code flow for creator authentication and token management.

**OpenID Connect 1.0**
- Official URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0, enabling platforms to verify creator identity and obtain basic profile information. Used in conjunction with OAuth 2.0 for user authentication in platform integrations that require verified creator identity (e.g., when onboarding creators via LinkedIn or Google accounts). Supports GDPR-compliant consent flows via the `prompt=consent` parameter.

**OWASP API Security Top 10**
- Official URL: https://owasp.org/www-project-api-security/
- The Open Web Application Security Project's definitive guide to API security risks, including broken authentication, excessive data exposure, lack of resource and rate limiting, and injection attacks. Directly applicable to any influencer platform exposing public APIs or handling creator authentication tokens and campaign data.

### Data Model & API Specifications

**OpenAPI Specification 3.1.1**
- Official URL: https://spec.openapis.org/oas/v3.1.1.html
- The industry standard for describing RESTful APIs in machine-readable YAML or JSON format, achieving full compatibility with JSON Schema Draft 2020-12. All major influencer data API providers (Modash, CreatorIQ ExchangeIQ, Upfluence) expose OpenAPI-compatible documentation. Any influencer platform API should be documented using OAS 3.1 to enable SDK auto-generation, interactive documentation, and contract testing.

**JSON Schema Draft 2020-12**
- Official URL: https://json-schema.org/specification
- The standard for validating the structure of JSON data, used in conjunction with OpenAPI 3.1 to define request and response data models for creator profiles, campaign data, audience demographics, and analytics payloads.

**Webhooks (HTTP Callbacks — RFC 2616 / RFC 7230)**
- No single formal standard; industry convention for real-time event notifications. Influencer platforms should support webhook subscriptions for events such as: new creator content published, campaign status changes, compliance flag triggered, payment processed. TikTok Creator Marketplace API, Upfluence API, and others use HTTP POST webhooks for real-time updates.

---

## Similar Products — Developer Documentation & APIs

### Modash Influencer Marketing API

- **Description:** REST API providing access to 380M+ creator profiles across Instagram, TikTok, and YouTube, with two distinct products: Discovery API (aggregated, searchable creator data) and Raw API (live, per-request data from any profile). Includes a Collaborations API mapping influencer and brand partnerships.
- **API Documentation:** https://docs.modash.io/
- **Discovery API:** https://www.modash.io/influencer-marketing-api/discovery
- **Raw API:** https://www.modash.io/influencer-marketing-api/raw
- **Collaborations API:** https://www.modash.io/influencer-marketing-api/collaborations-api
- **Postman Collection:** Available via official docs
- **Standards:** REST/JSON; resource-oriented endpoints; stateless communication
- **Authentication:** API token obtained from marketer.modash.io/developer
- **Pricing:** Discovery API from $16,200/year (3,000 credits/month); Raw API from $10,000/year (40,000 requests/month)
- **Notable:** AI Search endpoint enables natural language queries across creator content; 30+ data points in Collaborations API

### CreatorIQ API (ExchangeIQ)

- **Description:** Enterprise-grade REST API providing access to the CreatorIQ platform's creator data, campaign management, and analytics. Designed for integration with CRM (Salesforce), BI tools (Tableau), data lakes, and custom enterprise workflows. Used by Fortune 500 brands including Disney and Unilever.
- **API Documentation:** https://apidocs.creatoriq.com/
- **Overview:** https://apidocs.creatoriq.com/docs/ciq-api-documentation/o5yqwvpp1lbnb-overview
- **Standards:** REST/JSON; SSL/TLS encryption; JSON-based communications
- **Authentication:** Organization Name + API Key; contact support@creatoriq.com to obtain credentials
- **Notable:** ExchangeIQ brand name for the custom API; supports campaign workflows, CRM data feeds, content-first search, and Salesforce/Shopify integrations; access gated behind enterprise contract

### Upfluence API

- **Description:** REST API providing access to Upfluence's 12M+ creator database with AI-powered matching, Shopify conversion tracking, affiliate link management, and campaign analytics. Designed for e-commerce and D2C brands. Supports real-time webhooks for engagement notifications.
- **API Documentation:** https://www.upfluence.com/influencer-marketing-api
- **Developer Tracker:** https://apitracker.io/a/upfluence
- **GitHub:** https://github.com/upfluence
- **Standards:** REST/JSON; OAuth 2.0 authentication; webhook notifications for real-time events
- **Authentication:** OAuth 2.0; secure API key issuance
- **Notable:** Native Shopify and Amazon Attribution integrations; Zapier connector for third-party automation; webhook push for new posts and engagement spikes

### HypeAuditor API

- **Description:** Influencer analytics API providing creator profile data, audience quality scoring, fraud detection, and social listening. Includes a Recruitment API for discovery at scale and a My Network API for programmatic access to tracked creator lists.
- **API Documentation:** https://hypeauditor.readme.io/reference/basic
- **API Overview:** https://hypeauditor.com/api-integration/
- **Standards:** REST; POST method to `https://hypeauditor.com/api/method/%endpoint%`; JSON responses with `Content-Type: application/json`
- **Authentication:** Organization Name + Client ID + API Token; managed via admin panel; version v=2 current (v=1 deprecated August 2024)
- **Notable:** Fraud detection and audience authenticity scoring available via API; social listening endpoints for keyword/hashtag monitoring; My Network API added 2025 for programmatic network management

### IMAI (InfluencerMarketing.ai) API

- **Description:** RESTful API with access to a large creator database across Instagram, TikTok, and YouTube. Provides search, AI-powered matching, influencer reports, audience reports, and email lookup endpoints. Designed for agencies building custom influencer tools.
- **API Documentation:** https://docs.influencermarketing.ai
- **Product Page:** https://influencermarketing.ai/api/
- **Standards:** REST/JSON; resource-based, stateless endpoints; clean JSON request/response structure
- **Authentication:** API key; credit-based usage tracking (Discovery credits + Raw API requests tracked separately)
- **Pricing:** Platform from $99–$1,800/month; 7-day free trial; API usage billed in credits per endpoint
- **Notable:** AI-powered search for better creator-brand matching; email lookup endpoint for outreach; separate billing for Raw API vs. Discovery API

### Phyllo Social Data API

- **Description:** Unified API providing authenticated access to creator data across 20+ social platforms including Instagram, YouTube, TikTok, Twitch, and Patreon. Specializes in first-party, creator-consented data (identity, audience, engagement, income). Provides a Connect SDK for creator onboarding.
- **API Documentation:** https://www.getphyllo.com/influencer-marketing/api
- **Influencer Marketing API:** https://influencer-marketing.getphyllo.com/
- **Social Data API:** https://www.getphyllo.com/social-data-api
- **Standards:** REST/JSON; OpenAPI-compatible; Postman collections available
- **Authentication:** API key for server-side; creator-side authentication handled via Connect SDK (OAuth flows per platform)
- **Notable:** Only major provider offering consented, first-party creator data including income/earnings; handles complex per-platform OAuth flows; strong GDPR compliance posture due to creator-consent model; SDK components: server-side APIs + client-side Connect SDK

### Instagram Graph API (Meta)

- **Description:** Meta's official API for business and creator Instagram accounts. Provides access to content publishing, insights, comment moderation, audience demographics, and mentions. Required for any platform building Instagram integrations with data beyond public post metadata.
- **API Documentation:** https://developers.facebook.com/docs/instagram-platform/overview/
- **Content Publishing:** https://developers.facebook.com/docs/instagram-platform/content-publishing/
- **Permissions Reference:** https://developers.facebook.com/docs/permissions/
- **Standards:** REST/JSON; OAuth 2.0 (Business Login generating Instagram User access tokens); HTTPS required
- **Authentication:** OAuth 2.0; Business Login for business/creator accounts; token scopes control data access
- **Rate Limits:** 50 published posts per 24-hour period; per-endpoint rate limits apply
- **Notable:** API changes in 2025 introduced stricter permission scoping; public data access significantly restricted compared to legacy API; Business Login mandatory for professional account data

### TikTok for Developers API

- **Description:** TikTok's official developer platform providing access to creator data, content, and the TikTok Creator Marketplace. The Creator Search Insights API (introduced 2026) returns creator-level follower counts, engagement rates, audience demographics, and growth trends without requiring creator OAuth. TikTok Business API supports campaign management and Creator Marketplace workflows.
- **Developer Portal:** https://developers.tiktok.com/
- **Business API:** https://business-api.tiktok.com/portal/docs
- **Creator Marketplace Announcement:** https://www.socialmediatoday.com/news/tiktok-launches-creator-marketplace-api-to-facilitate-more-brand-collaborat/605900/
- **Standards:** REST/JSON; OAuth 2.0 for creator authentication; webhook support for Creator Marketplace order events
- **Authentication:** OAuth 2.0 for creator-consented data; API key for business-level data; application approval required
- **Notable:** Creator Search Insights API (2026) is the first TikTok endpoint providing demographic data without creator OAuth — significant for discovery workflows; TikTok Shop APIs expanded in 2025–2026

### YouTube Data API v3 (Google)

- **Description:** Google's primary API for programmatic access to YouTube channels, videos, playlists, and analytics. Provides channel statistics, video engagement data, audience demographics (age, gender, geography, device type), and upload history. Income and AdSense data accessible via YouTube Analytics API for creators who grant permission.
- **API Reference:** https://developers.google.com/youtube/v3/docs
- **OAuth Implementation Guide:** https://developers.google.com/youtube/v3/guides/authentication
- **Getting Started:** https://developers.google.com/youtube/v3/getting-started
- **Standards:** REST/JSON; OAuth 2.0; requires Google Cloud project and API enablement
- **Authentication:** OAuth 2.0; key scopes: `youtube.readonly`, `yt-analytics.readonly`, `youtubepartner` (partner-only); tokens require explicit refresh
- **Rate Limits:** Default 10,000 units/day per Google Cloud project; quota increase approval tightened since 2025
- **Notable:** YouTube Analytics API now supports granular demographic breakdowns; quota constraints make high-volume discovery workflows difficult without approved quota increases

### LinkedIn Marketing API

- **Description:** LinkedIn's official API for marketing integrations, providing access to LinkedIn ad campaigns, company pages, and content. Partner Program provides broader API access (higher rate limits, additional endpoints) for certified marketing technology platforms. Direct creator/influencer discovery API not yet publicly available; influencer data access requires Marketing Developer Platform partnership.
- **API Overview:** https://learn.microsoft.com/en-us/linkedin/marketing/overview
- **Marketing API Documentation:** https://learn.microsoft.com/en-us/linkedin/marketing/
- **Developer Portal:** https://developer.linkedin.com/
- **Standards:** REST/JSON; OAuth 2.0; requires LinkedIn App registration and Partner Program certification for advanced access
- **Authentication:** OAuth 2.0; application must be registered and approved; Marketing Developer Platform partner status required for broader access
- **Notable:** No native LinkedIn influencer discovery API comparable to Instagram Graph API; B2B influencer data access requires partnerships with LinkedIn data resellers (Favikon, Phyllo) or direct Marketing Partner Program membership; Substack creator data not available via official API

---

## Notes

**Platform API access is a critical build-vs-buy decision.** Direct social platform APIs (Instagram Graph API, TikTok for Developers, YouTube Data API) impose strict rate limits, require platform approval, and are increasingly restrictive about bulk data access. Most production influencer platforms use third-party data aggregators (Modash, Phyllo, HypeAuditor, IMAI) as their data layer, purchasing API access to pre-aggregated creator databases rather than building direct platform integrations.

**GDPR compliance is incompatible with scraped data.** The creator-consent model pioneered by Phyllo (first-party, creator-authenticated data) provides the strongest GDPR compliance posture. Platforms relying on scraping or third-party data aggregators must carefully review their data provenance and consent chains.

**TikTok's Creator Search Insights API (2026)** represents a significant shift — providing demographic-level data without creator OAuth authentication for the first time. This opens new possibilities for discovery workflows that previously required creators to authenticate their accounts.

**LinkedIn B2B influencer data** remains the largest gap in the official API landscape. No native LinkedIn influencer discovery API exists; platforms serving B2B verticals must partner with data resellers or the LinkedIn Marketing Partner Program.

**No open standards currently exist** for cross-platform influencer data interchange. The IAB Creator Economy Taxonomy (April 2025) provides vocabulary standardization but no data model or API interoperability standards. This is an area of active development and potential first-mover advantage for open-source tooling.
