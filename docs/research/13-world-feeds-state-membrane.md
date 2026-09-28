# 13 — World Feeds and the Internal/External State Membrane

Scope: this report designs the membrane between Year96's authoritative internal organization state and observed external world state. It assumes #01 provides CloudEvents-style StateEvent fabric and `y96://` addressing, #02 owns scope-effect prediction and watchlists, #04 owns Ext Comm gateways, #06 owns identity/policy gates, and #09 owns proof predicates. The core problem is not "add RSS"; it is to ingest world claims with provenance, rights, trust, freshness, and adversarial screening, then route only relevant, allowed, explained deltas into internal ownership/duty/thread state.

## TL;DR for the Year96 architect

- Split state into two authority domains: `y96://org/<org>/...` is internal source of truth, writable only by gated commands; `y96://world/<source>/...` is observed claim state, read-only except through feed providers, Ext Comm gateways, and verifier tools.
- Extend #01's `StateEvent` with `domain`, `authority`, `observedAt`, `sourceTrust`, `freshnessSla`, `usageRights`, `tosPolicy`, `fetchPolicy`, `worldClaimId`, and `internalization` refs. Do not let external claims overwrite internal objects.
- Treat external content as hostile input: normalize in a sandbox; preserve raw bytes/hash; strip active content; scan for PII, secrets, prompt injection, malware links, and rights/ToS conflicts before indexing or routing.
- Adopt boring syndication first: RSS 2.0, Atom, JSON Feed, WebSub, sitemaps/lastmod, HTTP conditional GET, Standard Webhooks, CloudEvents, AsyncAPI, SSE/WebSockets. These cover far more "world as feed" than exotic firehoses.
- Use ActivityPub/ActivityStreams, AT Protocol Jetstream, Wikimedia EventStreams, GitHub Events, Hacker News Firebase, SEC/arXiv/OpenAlex/GDELT/patents as provider adapters, not special cases.
- Default permissive stack: Miniflux/feedparser-like parser pattern, Scrapy, trafilatura, Crawl4AI, changedetection.io, dlt, Debezium, Apache NiFi, Redpanda Connect free API, datasketch, OPA, json-rules-engine, NATS. Exclude RSSHub/FreshRSS/Firecrawl/Airbyte core because AGPL/ELv2; they can be external integrations.
- Build a shared multi-tenant **World Mirror** only for public/common feeds; keep each org's subscriptions, relevance graph, and learned watchlists private because "what we monitor" leaks strategy.
- The membrane output is an `InternalizationEvent`, not a direct state mutation: it links external facts to internal subjects, owners, duties, threads, and watchlists with score, explanation, expiry, rights constraints, and confidence.
- Feed learning is a contextual bandit over cost and later usefulness: sources earn budget when their events become useful internalizations, proof predicates, or watchlist refreshes; they lose budget for noise, rights failures, stale data, and duplicates.
- Egress closes the proof loop: every outbound Ext Comm/action creates internal intent, external delivery attempt, observe-back subscription, platform-state observation, and #09 proof predicate.
- Biggest innovation: reliable trust/rights/relevance under adversarial and changing web conditions. Current tools crawl and parse; they do not know which claims an organization is allowed to use or should care about in five years.

## Landscape

### Standards and protocols

**RSS 2.0 and Atom.** RSS 2.0 remains the simplest widely deployed web-content syndication format; the RSS Advisory Board page says the current spec is 2.0.11 from March 2009 [1]. Atom RFC 4287 is an IETF Standards Track XML syndication format [2], and AtomPub RFC 5023 (not re-opened here) remains relevant for publishing. Year96 should normalize RSS/Atom to one `WorldItem` shape and preserve original fields for provenance.

**JSON Feed 1.1.** JSON Feed is explicitly a pragmatic JSON alternative to RSS/Atom and supports microblogs, attachments, feed icons, and extension fields [3]. It is ideal for agent-generated feeds because JSON parsing, schema validation, and typed extensions are cheaper than XML.

**WebSub.** W3C WebSub is a Recommendation for publisher-to-subscriber webhooks via hubs; hubs verify subscription requests and distribute updates [4]. It matters because polling everything is wasteful. Year96 should support WebSub for instant low-cost feeds but still verify signatures, freshness, replay protection, and source identity.

**ActivityPub and ActivityStreams 2.0.** ActivityPub is a W3C Recommendation for decentralized social networking, with client-to-server and server-to-server APIs based on ActivityStreams [5]. It matters as a rich external-state protocol: actors, activities, objects, inboxes, and federation become world claims, not trusted internal truth.

**CloudEvents, Standard Webhooks, and AsyncAPI.** CloudEvents graduated at CNCF in January 2024, and CloudEvents SQL reached v1 in June 2024 [6]. Standard Webhooks documents HTTP callbacks as service-to-service event notifications [7]. AsyncAPI 3.1.0 describes message-driven APIs in a protocol-agnostic Apache-2.0 specification [8]. Year96 should use CloudEvents as the outer event envelope, Standard Webhooks for signature/replay conventions, and AsyncAPI to describe feed/provider contracts.

**SSE/WebSockets and Wikimedia EventStreams.** Wikimedia EventStreams exposes `text/event-stream` streams, supports comma-composed streams, and allows timestamp-based historical consumption with finite retention [9]. This is a strong reference for resumable external streams with cursors, replay windows, and source schemas.

**AT Protocol firehose / Jetstream.** Bluesky's Jetstream is a JSON-filterable, lower-bandwidth alternative to the full atproto firehose, filterable by collection or repo and self-hostable [10]. For Year96 it is a model for "public social stream, private filter graph": consume shared public firehose infrastructure, but keep per-org filters private.

**Robots, sitemaps, conditional GET, and AI preferences.** RFC 9309 standardizes robots.txt path matching and crawl allow/deny behavior [11]. Sitemaps tell crawlers which pages matter and include `lastmod` and media hints [12]. HTTP conditional requests (`ETag`, `If-Modified-Since`) are mandatory for polite budgets [13]. The IETF AIPref draft distinguishes acquisition from usage and adds `Content-Usage` preferences to robots-style policy [14]. Year96 must store these signals as rights policy, not just crawler hints.

**NLWeb.** NLWeb proposes natural-language endpoints for websites, returning Schema.org JSON and acting as MCP servers, with A2A planned [15]. Treat it as a future "site-as-feed" provider: a vendor website can expose askable, structured world state without scraping.

### OSS ingestion and normalization

**Miniflux.** Miniflux supports Atom, RSS, and JSON Feed, OPML, tracking-param stripping, pixel-tracker removal, CSP/Trusted Types, media proxying, and content sanitization [16]. Its LICENSE is Apache-2.0 [17]. Year96 should adopt its design patterns and possibly components for feed hygiene, though not necessarily its whole reader UX.

**RSSHub and FreshRSS.** RSSHub's README says "Everything is RSSible" [18], but its LICENSE is AGPL-3.0 [19]. FreshRSS supports WebSub, XPath scraping, JSON documents, and reshare APIs [20], but its LICENSE is AGPL-3.0 [21]. Both are useful external integrations or reference catalogs, excluded from core.

**changedetection.io and Huginn.** changedetection.io monitors web pages, ships alerts to many channels, and now advertises AI change-detection rules [22]; its LICENSE is Apache-2.0 [23]. Huginn is a hackable self-hosted IFTTT/Zapier-like agent graph that reads the web, watches events, and takes actions [24]; its LICENSE is MIT [25]. Year96 should trial both patterns: changedetection for page-watch sensors, Huginn for graph-of-agents mental model.

**Scrapy, trafilatura, Crawl4AI, Firecrawl.** Scrapy is a mature Python web scraping framework, maintained by Zyte and contributors [26], under BSD-style license [27]. Trafilatura extracts main text and metadata and is Apache-2.0 [28][29]. Crawl4AI turns pages into LLM-ready Markdown and has self-hosted and cloud/MCP modes [30], Apache-2.0 [31]. Firecrawl is operationally attractive for search/scrape/interact APIs [32], but its LICENSE is AGPL-3.0 [33], so exclude from core and allow as paid external API if terms pass.

**Apache NiFi, Redpanda Connect, Debezium, dlt, Airbyte.** NiFi is Apache-2.0 and mature for flow-based ingestion [34][35]. Redpanda Connect describes declarative YAML stream processing, CDC connectors, Bloblang, at-least-once delivery, and observability; README exposes Apache v2 free APIs plus enterprise API surfaces [36]. Debezium provides low-latency CDC under Apache-2.0 [37][38]. dlt is an Apache-2.0 Python data loading library that agents can embed in notebooks, Lambda, Airflow, or local laptops [39][40]. Airbyte has broad connectors but the root LICENSE is Elastic License 2.0 and the README shows mixed MIT/ELv2 badges [41][42], so use only as external integration after connector-level review.

**datasketch.** datasketch provides MinHash/LSH and other probabilistic sketches for large-scale similarity, MIT licensed [43][44]. It is a good default for near-duplicate detection before expensive embeddings/LLM clustering.

### World sources and product patterns

**World data APIs.** GDELT monitors global news media and updates streams every 15 minutes [45]. arXiv offers public APIs with explicit API terms, acknowledgement and brand-use constraints [46]. OpenAlex provides an API/help center for scholarly metadata [47]. GitHub Events are optimized for polling with ETags and have rate/latency caveats: public events may lag 30 seconds to 6 hours and only include recent events [48]. Hacker News Firebase provides near-real-time public data and says there is currently no rate limit [49]. USPTO provides public patent data portals [50]. SEC EDGAR API fetch was blocked by SEC 403 in this environment; use official SEC docs and strict user-agent/rate policy in implementation.

**Commercial analogues.** Dataminr positions itself as real-time, client-tailored intelligence connecting emerging events, exposure, and business impact [51]. Feedly AI Feeds claim up to 9x more relevant market intelligence results through 1000+ market-intelligence concepts [52]. Recorded Future, AlphaSense, Klue, and Crayon show the same product pattern: fuse outside sources into organization-specific intelligence [53][54][55][56]. Year96 should not copy their closed data, but should copy the pipeline: sources -> extraction -> entity/topic graph -> relevance -> workflow delivery.

**Consumer proactive monitoring.** ChatGPT Pulse surfaces proactive updates based on prior context [57]. Gemini Scheduled Actions let users schedule recurring or one-off personalized updates [58]. Perplexity Tasks appeared in help/search results but official help was 403 here; mark product details unverified. These products validate that "watch the world for me" is becoming mainstream, but Year96's differentiator is proofs, rights policy, private relevance learning, and bidirectional observe-back.

## Component scorecard

| Name | Kind (OSS/paper/product/standard) | What it gives Year96 | License (SPDX, verified) | Maturity (stars, last release/commit, backer) | Verdict |
|---|---|---|---|---|---|
| RSS 2.0 | Standard | Simple ubiquitous feed format | Spec/public web | Current spec 2.0.11, 2009 [1] | Adopt |
| Atom RFC 4287/5023 | Standard | XML feed/publishing semantics | IETF RFC | Standards Track, mature [2] | Adopt |
| JSON Feed 1.1 | Standard | JSON-native feeds for agents | Spec/site terms (unverified) | Active open web format [3] | Adopt |
| WebSub | Standard | Push updates from feed hubs | W3C RF Recommendation | W3C Recommendation [4] | Adopt |
| ActivityPub | Standard | Federated social events/actors | W3C RF Recommendation | W3C Recommendation [5] | Trial |
| CloudEvents | Standard/OSS | Outer event envelope/filtering | Apache-2.0/CNCF | CNCF graduated Jan 2024 [6] | Adopt |
| Standard Webhooks | Standard/OSS | Webhook signature/replay conventions | Apache-2.0 verified [7][59] | Svix-origin community standard | Adopt |
| AsyncAPI 3.1 | Standard | Machine-readable event API contracts | Apache-2.0 [8] | Mature spec ecosystem | Adopt |
| robots.txt RFC 9309 | Standard | Crawl allow/deny | IETF RFC | Mature [11] | Adopt |
| IETF AIPref | Draft standard | AI content-usage preferences | IETF draft | Draft, 2025 [14] | Watch/Trial |
| ATProto Jetstream | OSS/protocol | Public social JSON firehose | MIT likely via atproto repo (not LICENSE-checked here) | Team-maintained, active [10] | Trial |
| Wikimedia EventStreams | Product/API | SSE stream/replay pattern | Site/API terms | Production public stream [9] | Trial |
| NLWeb | OSS/protocol | Website natural-language feed endpoint | MIT verified by README in #04; not rechecked here | Microsoft-origin, MCP now/A2A planned [15] | Watch |
| Miniflux | OSS | Secure feed reader/parser patterns | Apache-2.0 verified [17] | Active; stars not rechecked due GitHub API rate limit | Adopt patterns / Trial |
| RSSHub | OSS | Huge RSS route catalog | AGPL-3.0 verified [19] | Popular; activity not fully verified | Excluded-license |
| FreshRSS | OSS | Self-hosted reader, WebSub, scraping | AGPL-3.0 verified [21] | Mature; activity not fully verified | Excluded-license |
| changedetection.io | OSS/product | Page change sensor + alerts | Apache-2.0 verified [23] | Active README, AI rules [22] | Trial |
| Huginn | OSS | Event-agent graph for web automation | MIT verified [25] | Mature but older UX; activity not fully verified | Trial |
| Scrapy | OSS | Robust crawler framework | BSD-3-Clause-style verified [27] | Mature/Zyte-backed [26] | Adopt |
| trafilatura | OSS | Main text/metadata extraction | Apache-2.0 verified [29] | ACL-demo lineage; active README [28] | Adopt |
| Crawl4AI | OSS/product | LLM-ready Markdown crawler/MCP | Apache-2.0 verified [31] | Fast-growing; stars not rechecked | Trial |
| Firecrawl | OSS/product | Hosted scrape/search/interact API | AGPL-3.0 verified [33] | Active product [32] | Excluded-license core; external API |
| Apache NiFi | OSS | Visual/dataflow ingestion | Apache-2.0 verified [35] | Apache mature [34] | Trial |
| Redpanda Connect | OSS/product | YAML stream pipelines/connectors | Apache-2.0 free API + enterprise surfaces [36] | Active README; license split needs procurement review | Trial |
| Debezium | OSS | CDC from internal DBs/SaaS mirrors | Apache-2.0 verified [38] | Mature [37] | Adopt |
| dlt | OSS | Agent-friendly ELT loading | Apache-2.0 verified [40] | Active Python ecosystem [39] | Adopt |
| Airbyte | OSS/product | Broad connector catalog | ELv2 root + mixed docs [41][42] | Popular but license restricted | Excluded-license core |
| datasketch | OSS | MinHash/LSH duplicate detection | MIT verified [44] | Active enough; v2.0 note [43] | Adopt |
| OPA | OSS | Rights/routing policy checks | Apache-2.0 verified [60] | CNCF graduated [61] | Adopt |
| json-rules-engine | OSS | Simple declarative rules | ISC verified [62] | Useful lightweight TS rules [63] | Trial |
| GDELT | Product/API | Global news/event stream | Terms unverified | Updates every 15 min [45] | Trial |
| Feedly AI Feeds | Product | Learned market-intel feeds | Commercial terms | 1000+ AI models, 9x relevance claim [52] | Watch pattern |

## How I would build this part of Year96

### 1. Two-domain state model

Year96 should make domain authority explicit at the URI and event layers:

- Internal: `y96://org/acme/thread/facebook-ads-2026`, `y96://org/acme/ownership/ingredient-strategy`, `y96://org/acme/ad-intent/meta/campaign-123`. This is authoritative org-owned state. Writes require commands, identity gates (#06), communication routing (#04), and proof policy (#09).
- External: `y96://world/rss/example.com/feed/item/<hash>`, `y96://world/atproto/app.bsky.feed.post/<cid>`, `y96://world/sec/company/<cik>/filing/<accession>`, `y96://world/meta-ads/account/<id>/campaign/<remote-id>`. This is observed claim state with provenance and confidence.

Proposed #01 `StateEvent` deltas:

```ts
type StateDomain = 'internal' | 'external' | 'membrane';
type Authority = 'source-of-truth' | 'observed-claim' | 'derived-claim' | 'proof-observation';

interface StateEventY96WorldExt {
  domain: StateDomain;
  authority: Authority;
  observedAt?: string;
  firstSeenAt?: string;
  lastSeenAt?: string;
  sourceTrust?: { source: string; score: number; basis: string[]; updatedAt: string };
  freshnessSla?: { maxAgeMs: number; staleAfter: string; nextFetchDue?: string };
  usageRights?: {
    acquisition: 'allowed' | 'disallowed' | 'unknown';
    usage: ('index' | 'summarize' | 'quote' | 'train' | 'route' | 'store-raw')[];
    robots?: string;
    aiPreferences?: string;
    tosRef?: string;
    attribution?: string;
  };
  fetchPolicy?: { method: 'poll' | 'push' | 'stream' | 'manual'; etag?: string; lastModified?: string; rateLimit?: string };
  worldClaimId?: string;
  clusterId?: string;
  injectionRisk?: 'low' | 'medium' | 'high' | 'blocked';
}
```

Authority rule: internal objects are never overwritten by external claims. If Meta Ads API says a campaign is paused, Year96 records an external platform-state claim and routes an internalization candidate to the Ads Duty; only a gated internal command updates internal intent/status. This creates dual representation: internal desired campaign (`budget`, `audience`, `creative`, `why`) plus external platform state (`remote status`, `review`, `delivery`, `spend`, `errors`).

### 2. State Membrane pipeline

Ingress sequence:

1. `FeedSubscription` is declared by a sensor (#07), watchlist (#02), hanger/listener (#04), ownership duty, proof requirement (#09), or learned discovery policy.
2. `SubscriptionManager` resolves provider, credentials, legal basis, budget, and privacy class. Subscriptions are confidential internal state.
3. `FetchScheduler` chooses poll/push/stream cadence using freshness SLA, ETag/Last-Modified, WebSub/webhook availability, source cost, error rates, and expected utility.
4. `FeedSourceProvider` fetches or receives raw data into quarantine. It records bytes, headers, cursor, source IP, TLS, signature, robots/AIPref/sitemap signals, ToS ref, and cost.
5. `Normalizer` parses into `WorldItem` with canonical URL, authors, timestamps, language, entities, claims, attachments, and raw-hash pointer.
6. `DedupClusterer` uses canonical URL, content hash, SimHash/MinHash, embeddings, entity/time overlap, and source syndication hints to avoid repeated alerts and form story clusters.
7. `TrustScorer` scores source and item: source reputation, authenticated push signature, historical correction rate, cross-source corroboration, temporal proximity, bot likelihood, and adversarial flags.
8. `RightsPolicyProvider` and safety screeners decide whether Year96 may store raw, extract text, index, summarize, route, quote, train, or must drop/redact. PII and prompt-injection scanners label the item before any LLM sees it.
9. `WorldMirrorStore` writes external claim events and projections. Public/common items may go to a shared mirror; org-specific private sources and all subscription metadata stay in the org partition.
10. `RelevanceRouter` calls #02's cascade: standing rules, graph blast radius, watchlists, embeddings, learned ranker, and expensive adjudicator only for high-value candidates.
11. `InternalizationPolicy` emits `y96.membrane.internalization.proposed` or `accepted`, linking external claim cluster to internal owners/duties/threads with relevance score, why path, confidence, expiry, allowed-use constraints, and proof obligations.

Egress sequence:

1. Internal intent is created: "launch Meta ads campaign X".
2. #04 Ext Comm gateway and #06 policy gate approve outbound API/message.
3. External action provider executes and records delivery attempt as external observation.
4. `ObserveBackVerifier` subscribes/polls platform state, receipts, webhooks, spend/impression stats, and screenshots.
5. #09 proof predicates evaluate observed external state against expected internal end state. Failures route back as internal events, not silent retries.

### 3. Core types and provider interfaces

```ts
type Y96Uri = `y96://${string}`;

interface FeedSubscription {
  id: Y96Uri;
  org: Y96Uri;
  source: string;
  target: string; // feed URL, API endpoint, stream subject, webhook route
  declaredBy: Y96Uri;
  purpose: 'proof' | 'watchlist' | 'ownership-sensor' | 'thread-hanger' | 'learned' | 'manual';
  confidentiality: 'public' | 'org-private' | 'secret-strategy';
  freshnessSlaMs: number;
  budget: { maxFetchesPerDay: number; maxCostUsdPerDay: number; maxItemsPerDay: number };
  rightsRequired: ('store-raw' | 'index' | 'summarize' | 'route' | 'quote')[];
  routingHints: { subjects: Y96Uri[]; rules: string[]; watchlists: Y96Uri[] };
}

interface WorldItem {
  uri: Y96Uri;
  source: string;
  canonicalUrl?: string;
  title?: string;
  text?: string;
  entities: { type: string; name: string; uri?: Y96Uri; confidence: number }[];
  claims: { predicate: string; object: unknown; confidence: number; evidence: string }[];
  publishedAt?: string;
  observedAt: string;
  rawBlob: { uri: Y96Uri; sha256: string; mediaType: string };
  rights: StateEventY96WorldExt['usageRights'];
  trust: StateEventY96WorldExt['sourceTrust'];
  injectionRisk: StateEventY96WorldExt['injectionRisk'];
}

interface FeedSourceProvider {
  id: string;
  supports(target: string): boolean;
  discover(seed: DiscoverySeed): Promise<FeedCandidate[]>;
  fetch(sub: FeedSubscription, cursor?: unknown): Promise<FetchBatch>;
  receivePush(req: IncomingWebhook): Promise<FetchBatch>;
  health(): Promise<ProviderHealth>;
}

interface FeedRegistry {
  registerProvider(p: FeedSourceProvider): void;
  upsertSubscription(sub: FeedSubscription): Promise<void>;
  candidates(q: FeedDiscoveryQuery): Promise<FeedCandidate[]>;
}

interface SubscriptionManager {
  declare(request: SubscriptionRequest, actor: Y96Uri): Promise<FeedSubscription>;
  revoke(id: Y96Uri, reason: string): Promise<void>;
  listDue(now: string): Promise<FeedSubscription[]>;
}

interface FetchScheduler {
  next(sub: FeedSubscription, stats: SourceStats, budget: BudgetState): Promise<FetchPlan>;
  record(result: FetchResult): Promise<void>;
}

interface Normalizer { normalize(batch: FetchBatch): Promise<WorldItem[]>; }
interface DedupClusterer { assign(items: WorldItem[]): Promise<{ item: WorldItem; clusterId: Y96Uri; duplicateOf?: Y96Uri }[]>; }
interface TrustScorer { score(item: WorldItem, cluster: ClusterContext): Promise<WorldItem['trust']>; }
interface RightsPolicyProvider { evaluate(item: WorldItem, sub: FeedSubscription): Promise<RightsDecision>; }
interface WorldMirrorStore { put(item: WorldItem): Promise<void>; get(uri: Y96Uri): Promise<WorldItem>; query(q: WorldQuery): AsyncIterable<WorldItem>; }
interface RelevanceRouter { route(item: WorldItem, ctx: OrgContext): Promise<RoutingDecision[]>; }
interface InternalizationPolicy { decide(r: RoutingDecision): Promise<InternalizationEvent | null>; }
interface ObserveBackVerifier { observe(intent: Y96Uri, externalRef: Y96Uri, predicates: ProofPredicate[]): Promise<ObserveBackResult>; }
```

### 4. Routing, feed learning, and budgets

Use three routing layers:

- Declarative rules: OPA for rights/security and json-rules-engine for business-readable "if source is SEC filing and entity matches portfolio company, route to finance duty". Drools/KIE is powerful but heavier.
- Subject hierarchy: NATS subjects like `world.news.company.spacex.patent`, `org.acme.watchlist.food-ingredients.*`, and `proof.meta-ads.campaign.*` map cheap filters to #02 candidate generation.
- Learned routing: contextual bandit/ranker learns which feed-source/event features predicted "mattered later" outcomes: accepted internalizations, useful watchlist refreshes, proof failures caught early, strategy changes, or human useful/noisy feedback.

Budget score:

`expectedUtility = pUseful * impact * freshnessValue * learningValue - fetchCost - attentionCost - rightsRisk - privacyLeakRisk`

The scheduler allocates budget by source and by purpose. Proof feeds get hard freshness SLAs. Strategic weak-signal feeds get low-frequency option budgets. Low-trust high-noise feeds decay unless they periodically produce useful corroborated clusters. Discovery expands from internal subjects: owners, duties, hangers, competitors, suppliers, technologies, regulations, and weakly connected graph neighbors suggested by #02.

### 5. Worked examples

**Rocket-company patent -> food-ingredient watchlist.** A patent provider ingests a rocket-company patent about cryogenic microencapsulation. Normalization extracts assignee, inventors, CPC classes, abstract, and entities. Dedupe clusters it with a trade-news story and arXiv materials paper. Trust is moderate: patent source high, relevance speculative. #02 finds a weak path from cryogenic systems -> protein stability -> shelf-stable ingredients -> the org's food-ingredient Ownership. The membrane emits `internalization.proposed` with `pMatters=0.07`, `horizon=5y_plus`, rights "summarize/route only", and a watchlist refresh, not an interruption. If later hiring/news corroborates food applications, the watchlist crosses a notify threshold.

**Facebook/Meta ads campaign.** Internal intent says: "launch campaign for product A, budget $500/day, target segment B, approved creative C, why = validate demand." Ext Comm gateway calls Meta APIs after #06 approval. External state says campaign remote ID `123`, review pending, then active, then spend/impressions. Observe-back verifier polls API/webhooks and captures screenshots. Internal state remains: desired campaign, owner, duty, proof predicates. External state remains: platform claims. A mismatch, e.g. remote budget $5,000/day, routes an urgent internalization to Ads Duty and #09 fails the proof bundle.

**Internal PR merged -> scope effect on dependent threads.** A PR merge is internal authoritative state from GitHub integration or repo event. It is not a world claim if the repo is org-owned. #01 emits internal `code.change.merged`; #02 affected-graph and standing queries wake threads depending on that package. If the repo is public, the same event may also appear in the shared world mirror for external observers, but Year96's org uses the internal event as source of truth.

### 6. Permissive-only default stack

Laptop: NATS JetStream from #01, SQLite/Postgres, Miniflux/feedparser-style parser, Scrapy + trafilatura + Crawl4AI for allowed pages, changedetection.io for page diffs, dlt for loading APIs into DuckDB/Postgres, datasketch for MinHash, OPA + json-rules-engine for rights/routing, Playwright screenshots for observe-back, local prompt-injection/PII classifiers.

Cluster: Kafka/Pulsar/NATS provider from #01, object storage raw blobs, Apache NiFi or Redpanda Connect for managed pipelines, Debezium for CDC, dlt jobs for long-tail APIs, Vespa/OpenSearch/Qdrant projections from #01, Feldera/RisingWave standing queries from #02, OPA distributed policy, shared public World Mirror partitions plus per-org private relevance graph. Excluded-license tools may run only as external integrations with legal review and no core dependency.

### 7. Proof and test plan: the 70% rule

- Unit: URI/domain authority checks; parser golden files; robots/AIPref policy; ETag scheduling; dedupe hash stability; rights decisions.
- Integration: RSS/Atom/JSON Feed/WebSub/Standard Webhook/Jetstream/EventStreams providers with replayed fixtures and cursor failures.
- Mocked E2E: noisy duplicate news cluster -> one internalization; rights-blocked page -> no LLM/index; prompt-injection feed item -> quarantine; proof feed stale -> #09 predicate fails.
- E2E: run a seed org with real public feeds under budgets, verify accepted internalizations, watchlist refreshes, and no forbidden writes to internal state.
- Agentic verifiers: adversarial source tries prompt injection, PII leakage, fake webhook replay, source poisoning, ToS conflict, and strategy-leaking subscription query.
- Observability: every fetch has trace, cost, headers, cursor, raw hash, policy decisions, model versions, cluster IDs, route explanations, and outcome labels.
- Delayed proof: observe-back verifiers recheck external actions after SLA windows and reopen threads if world state diverges.

## What is still unsolved (late 2026)

- **Rights semantics are not settled.** robots.txt is about crawling, AIPref is draft, ToS are prose, and "summarize vs train vs route" boundaries are legally fuzzy.
- **Trust is contextual.** A source can be reliable for timestamps but unreliable for causality; a social post can be false yet strategically important because others believe it.
- **Subscription privacy leaks intent.** A per-org feed graph reveals competitors, deals, product strategy, and fears. Shared mirrors must not expose private filters.
- **Prompt injection through feeds is a first-class attack.** Feed text will tell agents to ignore policies. #06 must ensure external content is data, never authority.
- **Adaptive recrawl is harder than it looks.** Change-rate estimators handle pages; they do not handle adversarial sites, bot walls, API policy shifts, or "rare but critical" sources.
- **Near-duplicate clustering is not story truth.** Similar articles may share wire copy but differ in corrections, jurisdiction, or one crucial fact.
- **World-source ToS/rate limits change.** Year96 needs provider-health and legal-policy updates as live state, not static config.
- **Long-horizon feed learning lacks labels.** Most weak signals never resolve cleanly. Year96 must learn from "mattered later" labels via #02/#09 replay and human adjudication.
- **External action proof remains probabilistic.** SaaS APIs hide review queues, ad auctions, moderation, and fraud systems. Observe-back can prove observed state, not omniscience.

## Sources

1. https://www.rssboard.org/rss-specification
2. https://datatracker.ietf.org/doc/html/rfc4287
3. https://www.jsonfeed.org/version/1.1/
4. https://www.w3.org/TR/websub/
5. https://www.w3.org/TR/activitypub/
6. https://cloudevents.io/
7. https://www.standardwebhooks.com/
8. https://www.asyncapi.com/docs/reference/specification/latest
9. https://wikitech.wikimedia.org/wiki/Event_Platform/EventStreams
10. https://atproto.com/blog/jetstream
11. https://www.rfc-editor.org/rfc/rfc9309.html
12. https://developers.google.com/search/docs/crawling-indexing/sitemaps/overview
13. https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Conditional_requests
14. https://www.ietf.org/archive/id/draft-ietf-aipref-attach-02.html
15. https://github.com/nlweb-ai/NLWeb
16. https://github.com/miniflux/v2
17. https://raw.githubusercontent.com/miniflux/v2/main/LICENSE
18. https://github.com/RSSHub/RSSHub
19. https://raw.githubusercontent.com/DIYgod/RSSHub/master/LICENSE
20. https://github.com/FreshRSS/FreshRSS
21. https://raw.githubusercontent.com/FreshRSS/FreshRSS/edge/LICENSE.txt
22. https://github.com/dgtlmoon/changedetection.io
23. https://raw.githubusercontent.com/dgtlmoon/changedetection.io/master/LICENSE
24. https://github.com/huginn/huginn
25. https://raw.githubusercontent.com/huginn/huginn/master/LICENSE
26. https://github.com/scrapy/scrapy
27. https://raw.githubusercontent.com/scrapy/scrapy/master/LICENSE
28. https://github.com/adbar/trafilatura
29. https://raw.githubusercontent.com/adbar/trafilatura/master/LICENSE
30. https://github.com/unclecode/crawl4ai
31. https://raw.githubusercontent.com/unclecode/crawl4ai/main/LICENSE
32. https://github.com/firecrawl/firecrawl
33. https://raw.githubusercontent.com/firecrawl/firecrawl/main/LICENSE
34. https://github.com/apache/nifi
35. https://raw.githubusercontent.com/apache/nifi/main/LICENSE
36. https://github.com/redpanda-data/connect
37. https://github.com/debezium/debezium
38. https://raw.githubusercontent.com/debezium/debezium/main/LICENSE.txt
39. https://github.com/dlt-hub/dlt
40. https://raw.githubusercontent.com/dlt-hub/dlt/devel/LICENSE.txt
41. https://github.com/airbytehq/airbyte
42. https://raw.githubusercontent.com/airbytehq/airbyte/master/LICENSE
43. https://github.com/ekzhu/datasketch
44. https://raw.githubusercontent.com/ekzhu/datasketch/master/LICENSE
45. https://gdeltproject.org/
46. https://info.arxiv.org/help/api/index.html
47. https://docs.openalex.org/
48. https://docs.github.com/en/rest/activity/events?apiVersion=2022-11-28
49. https://github.com/HackerNews/API
50. https://www.uspto.gov/learning-and-resources/open-data-and-mobility
51. https://www.dataminr.com/
52. https://docs.feedly.com/article/699-guide-to-ai-feeds-market-intel
53. https://www.recordedfuture.com/platform
54. https://www.alpha-sense.com/
55. https://www.klue.com/
56. https://www.crayon.co/
57. https://openai.com/index/introducing-chatgpt-pulse/
58. https://blog.google/products-and-platforms/products/gemini/scheduled-actions-gemini-app/
59. https://raw.githubusercontent.com/standard-webhooks/standard-webhooks/main/LICENSE
60. https://raw.githubusercontent.com/open-policy-agent/opa/main/LICENSE
61. https://github.com/open-policy-agent/opa
62. https://raw.githubusercontent.com/CacheControl/json-rules-engine/master/LICENSE
63. https://github.com/CacheControl/json-rules-engine
