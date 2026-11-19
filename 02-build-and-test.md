# 2. Build and test

<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. SPDX-License-Identifier: MIT-0 -->

[← Start](README.md) · [Next: 3. Migrate →](03-migrate.md)

You arrive with the Phase 2 decisions. This file builds the store's AWS edge stack beside the current edge, restores every control the current edge enforces today, and prepares the DNS records that [3. Migrate](03-migrate.md) uses to move users. Nothing here touches production traffic; the current edge keeps serving. You leave with the Phase 3 gate: every test passed, fail back included.

## Phase 3: Stage and test the configuration

### Discover and design

The inventory is the source of truth for everything you build below, and the parity list that comes out of it gates the first public cohort.

- [ ] **Inventory hostnames, certificates and DNS.**  
  Why: Every apex, wildcard, CAA record, certificate-validation record, TTL and DNSSEC setting must exist on the AWS side before anything moves. Check long values and large record sets against Route 53 quotas.  
  Read: [CloudFront certificate requirements (Doc, R203)](#r203) — certificate names and Region; [Quotas for Amazon Route 53 (Doc, R121)](#r121) — record and value limits.  
  Pillar: Reliability

- [ ] **Inventory cache, origin and edge behavior.**  
  Why: Match conditions, evaluation order, TTLs, cache keys, origin headers, redirects, edge functions, purge methods and log consumers each map to a CloudFront setting or to an item below.  
  Read: [CloudFront cache behavior settings (Doc, R245)](#r245) — order and methods; [CloudFront cache-tag invalidation (Doc, R215)](#r215) — tag purge.  
  Pillar: Operational Excellence

- [ ] **Inventory security controls and classify every rule as parity or new.**  
  Why: Each rule's action, exception, rate key, client integration and owner decide how it maps to AWS WAF. Parity rules must enforce before any public cohort moves.  
  Read: [Testing and tuning AWS WAF protections (Doc, R401)](#r401) — safe rule rollout.  
  Pillar: Security

- [ ] **Export the observed positive model from current logs.**  
  Why: The hosts, methods, paths, content types and request sizes the application accepts today become the starting allowlist.  
  Read: [AWS WAF rule statements (Doc, R460)](#r460) — match and logical statements.  
  Pillar: Security

- [ ] **Map content types to ordered cache behaviors.**  
  Why: Each row needs a path pattern, allowed methods, cache key, CDN TTL, browser TTL, values forwarded to the origin, edge logic and origin. A personalized response must never share a cache key between users.  
  Read: [Managed cache policies (Doc, R207)](#r207) — cache-key choices; [Managed origin request policies (Doc, R208)](#r208) — values sent only to the origin.  
  Pillar: Security

Worked example for the store, with Amazon S3 and an Application Load Balancer (ALB) as origins:

| Content | Path | Cache key | CDN TTL | Browser TTL | Forwarded to origin | Edge logic | Origin |
|---|---|---|---|---|---|---|---|
| Images | `/images/*` | Path and query string | 1 year | 1 year | Query string | Image optimization | S3 |
| CSS, JavaScript, fonts | By extension | Path, normalized Accept-Encoding | 1 year | 1 year | None | Compression | S3 |
| HTML | No extension | Path, normalized Accept-Encoding, device type, signed-in or not | Origin TTL | Origin TTL | Device type (CloudFront headers) | Language and apex redirects, compression | ALB |
| API | `/api/*` | Caching off | — | — | Query string, cookies, selected headers | None | ALB |

### Build through IaC

The inventory becomes templates. One pipeline deploys everything, so one rollback undoes it.

- [ ] **Deploy the whole AWS edge stack through IaC.**  
  Why: The distribution, protection pack, Shield Advanced protection, certificates, DNS and API resources need one reviewed deployment and one rollback path.  
  Read: [CloudFormation change sets (Doc, R810)](#r810) — pre-deployment review; [CloudFormation Hooks (Doc, R812)](#r812) — enforced controls.  
  Pillar: Operational Excellence

- [ ] **Attach the protection pack when you create the distribution.**  
  Why: Changing the protection-pack association on a distribution that has a continuous deployment policy is restricted. Attaching at creation avoids the restriction.  
  Read: [Continuous deployment limits (Doc, R806)](#r806) — association restrictions.  
  Pillar: Reliability

### Keep both origin access paths working

The origin accepts both edges until the current edge is decommissioned in [3. Migrate](03-migrate.md). Lock it to CloudFront earlier and fail back has nowhere to go.

- [ ] **Add a CloudFront-only origin access path and leave the current edge's path in place until decommissioning.**  
  Why: CloudFront authenticates to the origin with OAC, a VPC origin, origin mTLS or a secret header; the current edge keeps its own credential or IP rule. Remove the current edge's path early and the weighted rollout and fail back stop working. A VPC origin is private, so the current edge needs a separate public path.  
  Read: [Origin access control (Doc, R205)](#r205) — supported AWS origins; [CloudFront VPC origins (Doc, R204)](#r204) — private origins and limits; [Restricting access to an Application Load Balancer (Doc, R251)](#r251) — secret header and rotation.  
  Pillar: Security

- [ ] **Match viewer and origin TLS behavior.**  
  Why: The viewer certificate comes from ACM in `us-east-1`; redirect HTTP to HTTPS and set HSTS with a response headers policy. The origin certificate must cover the origin domain name, or the Host header if you forward it, or CloudFront returns 502.  
  Read: [HTTPS to custom origins (Doc, R247)](#r247) — origin name matching; [Managed response headers policies (Doc, R209)](#r209) — HSTS.  
  Pillar: Security

### Configure AWS WAF and AWS Shield Advanced

Parity rules come first: they must enforce before the first public cohort moves. New rules start in Count and earn their action in [4. Operate](04-operate.md). AWS WAF evaluates rules in priority order and stops at the first Allow, Block, CAPTCHA or Challenge; Count does not stop evaluation.

- [ ] **Put AWS WAF on CloudFront, and nowhere else.**  
  Why: AWS WAF behind CloudFront sees CloudFront's addresses, not your users', so IP, geo, ASN and rate rules there depend on a forwarded header that a client can forge. Nothing in this checklist puts a protection pack on the origin or the API.  
  Read: [Resources that AWS WAF can protect (Doc, R462)](#r462) — CloudFront scope.  
  Pillar: Security

- [ ] **Order rules by threat type and cost.**  
  Why: Order the rules by threat: availability, then vulnerabilities, then abuse. That gives: narrow known-good Allow rules; the anti-DDoS rule group; positive-model Block rules; IP reputation and rate limits; the Core rule set and Known bad inputs groups; then the scoped Bot Control, ATP and ACFP groups. The paid groups go last so a request blocked earlier never reaches them. Use CAPTCHA instead of Block on human paths such as sign-up.  
  Read: [Using the anti-DDoS rule group (Doc, R453)](#r453) — priority; [Intelligent threat mitigation best practices (Doc, R428)](#r428) — paid-rule scope.  
  Pillar: Cost Optimization

- [ ] **Configure the anti-DDoS rule group for staging in Count.**  
  Why: The rule group learns from production traffic, so staging only confirms the configuration. The production Count period and the switch to Block happen in [3. Migrate](03-migrate.md).  
  Read: [Using the anti-DDoS rule group (Doc, R453)](#r453) — testing, sensitivity and exemptions.  
  Pillar: Security

- [ ] **Build the positive security model in Count.**  
  Why: These rules block every Host header, method, path, content type and size the application does not accept. Set body inspection size, oversize handling and the JSON invalid-body fallback explicitly.  
  Read: [AWS WAF body inspection limits (Doc, R411)](#r411) — inspection size; [Oversize request components (Doc, R412)](#r412) — Continue, Match or No match; [Request components in AWS WAF (Doc, R448)](#r448) — JSON body fallback.  
  Pillar: Security

- [ ] **Run Bot Control as a measured PoC.**  
  Why: Scope the paths, integrate tokens, observe in Count, analyze labels, enforce one rule at a time, and judge against the Phase 1 criteria. Token-dependent rules stay in Count until the weighted ramp reaches 100% in [3. Migrate](03-migrate.md).  
  Read: [Implementing a bot control strategy on AWS (Guide, R426)](#r426) — PoC lifecycle; [AWS WAF Bot Control (Doc, R404)](#r404) — rules and labels.  
  Pillar: Security

- [ ] **Set AWS WAF SDK integration explicitly in IaC.**  
  Why: Token-dependent rules need the AWS WAF client SDK on the page. Automatic injection by CloudFront has side effects and does not fit every architecture; integrating the SDK in the application avoids both. Choose in IaC, not by console default.  
  Read: [Automatic AWS WAF SDK injection (Doc, R409)](#r409) — side effects and unsupported configurations.  
  Pillar: Reliability

- [ ] **Turn on Origin Shield and test it functionally.**  
  Why: Origin Shield, a central cache layer in front of your origin, cuts origin load while edge caches fill during rollout. Enable it in the Region the documentation maps to your origin.  
  Read: [CloudFront Origin Shield (Doc, R210)](#r210) — Region choice and behavior.  
  Pillar: Performance Efficiency

- [ ] **Subscribe to Shield Advanced and protect the distribution, hosted zone and public load balancer.**  
  Why: Shield Advanced adds resource-specific DDoS detection, cost protection and SRT access. Application-layer mitigation comes from the anti-DDoS rule group in the protection pack; the blog explains how the two fit together.  
  Read: [AWS best practices for DDoS resiliency (Guide, R501)](#r501) — architecture; [Shield Advanced and the anti-DDoS managed rule group (Blog, R502)](#r502) — how Shield Advanced and the rule group work together.  
  Pillar: Reliability

- [ ] **Set up SRT access, proactive engagement and DDoS alarms.**  
  Why: With proactive engagement the SRT contacts you during a DDoS event; it needs a contact and a Route 53 health check on each protected resource (DNS section below). Alarm in `us-east-1` on Shield Advanced's `DDoSDetected` and the anti-DDoS rule group's `DDoSAttackRequests`, treating missing data as not breaching: both metrics are sparse outside an event.  
  Read: [Shield Advanced and the anti-DDoS managed rule group (Blog, R502)](#r502) — alarm guidance; [Health-based detection with Shield Advanced (Doc, R503)](#r503) — health check prerequisite.  
  Pillar: Reliability

### Design for low false positives

A false positive is a legitimate request that a rule blocks. The Phase 1 ceiling is the number these three items keep you under, during the rollout and after it.

- [ ] **Design rules so that false positives stay under the Phase 1 ceiling.**  
  Why: A rule scoped to the paths it protects fires on less legitimate traffic. Tune sensitivity and rate thresholds; Block only on high confidence, otherwise CAPTCHA, Challenge or Count. IP-only rules block everyone behind a shared address, so combine IP with the client's TLS fingerprint and HTTP header order, which AWS WAF exposes as match fields.  
  Read: [Managing false positives in AWS WAF (Builder Center, R464)](#r464) — design, detect and exclude; [Reduce false positives with client fingerprint and header order (Blog, R465)](#r465) — signals beyond IP.  
  Pillar: Security

- [ ] **Test the known Bot Control false-positive cases in Count before you enable Block.**  
  Why: Mobile apps and in-app browsers with non-browser user agents, your own monitors and tests, marketing tools, and verified bots behind a proxy all look like bots. Add exceptions in Count first.  
  Read: [Example false-positive scenarios with Bot Control (Doc, R419)](#r419) — the known cases and the user-agent exception example.  
  Pillar: Security

- [ ] **Write the false-positive runbook: find the rule in the log, then make the narrowest change.**  
  Why: The AWS WAF log's `terminatingRuleId` names the rule that blocked a request. Fix in order. First, set that rule to Count so it only labels, and add a rule that blocks on the label except for the legitimate case. Second, add a scope-down statement, which limits what the rule group inspects. Third, fix body parsing and size handling. Give users a way to report a 403. Do not fail back over a false positive.  
  Read: [How do I detect false positives caused by AWS Managed Rules? (AWS re:Post, R466)](#r466) — log fields and the label method, step by step; [Rule action overrides (Doc, R415)](#r415) — one-rule changes; [Label-match rules (Doc, R417)](#r417) — targeted enforcement.  
  Pillar: Operational Excellence

### Configure APIs

The store's API hostname gets the same shape as its pages: CloudFront in front, the protection pack on CloudFront, an origin that answers only CloudFront.

- [ ] **Create the API with the Regional endpoint type and put it behind CloudFront.**  
  Why: The edge-optimized type puts the API behind a CloudFront distribution that API Gateway controls, not you. With the Regional type your distribution is the only edge, and the DDoS and AWS WAF layer for the API. Cache only confirmed cacheable GET paths.  
  Read: [Protecting API endpoints (Guide, R601)](#r601) — CloudFront in front of Regional APIs.  
  Pillar: Security

- [ ] **Pick one API origin pattern and record it.**  
  Why: Pattern 1, the simpler: give the API a custom domain name that only CloudFront calls, with the managed origin request policy AllViewerExceptHostHeader so the API sees its own name. Pattern 2: point CloudFront at the hostname API Gateway assigns and forward Host exactly as the second resource describes.  
  Read: [API Gateway custom domains behind CloudFront (AWS re:Post, R617)](#r617) — custom-domain pattern; [API Gateway with a CloudFront distribution (AWS re:Post, R602)](#r602) — Regional-target pattern.  
  Pillar: Reliability

- [ ] **Block direct access to the API with a Lambda authorizer that checks CloudFront's secret header.**  
  Why: A Lambda authorizer is a function API Gateway runs before each request to admit or reject it. CloudFront adds a rotated secret header to every origin request; the authorizer rejects requests without it. Attach it to every route and turn off the public hostname API Gateway gives every API. This is origin access control, not a WAF.  
  Read: [Restricting an HTTP API with a Lambda authorizer (Blog, R618)](#r618) — header check and secret rotation.  
  Pillar: Security

- [ ] **Validate requests and throttle bursts at API Gateway.**  
  Why: Request validation rejects malformed bodies before they reach your backend. Throttling protects the backend from bursts that pass the edge.  
  Read: [REST API request validation (Doc, R606)](#r606) — models; [API Gateway throttling (Doc, R604)](#r604) — stage and method limits.  
  Pillar: Security

### Prepare DNS

[3. Migrate](03-migrate.md) moves users by changing DNS answers. These items make sure Route 53 can give those answers, and that a changed answer reaches resolvers in minutes, not days.

- [ ] **Lower TTLs before the first traffic change.**  
  Why: Lower each record that has its own TTL (aliases use their target's) to 60–300 seconds, at least one previous-TTL period before the first change. If the zone moves, lower the NS TTL at both providers and allow for the parent zone's cached NS records, which you do not control.  
  Read: [Route 53 zone migration (Doc, R101)](#r101) — TTL and name-server sequence.  
  Pillar: Reliability

- [ ] **If DNS moves, prepare the Route 53 zone now and move it in [3. Migrate](03-migrate.md).**  
  Why: Import the zone, compare every record, and keep both edges' certificate-validation records. The name-server and DNSSEC changes touch live traffic, so they run as their own stage before any cohort moves.  
  Read: [Route 53 zone migration (Doc, R101)](#r101) — import and comparison steps.  
  Pillar: Reliability

- [ ] **If the parent zone stays elsewhere, delegate a steering subdomain to Route 53.**  
  Why: Route 53 must answer for the IP-based, geolocation and weighted records. Production subdomains point to the steering name with CNAME records, and `<host>` in the tree below means the steering name, for example `www.edge.example.com`. The apex cannot be a CNAME, so it cannot use one. Never hold a record in both the parent and the child zone.  
  Read: [Routing traffic for subdomains (Doc, R122)](#r122) — delegation while keeping the parent zone.  
  Pillar: Reliability

### Build the rollout records

Route 53 picks each answer in three steps: by the resolver's IP range, then by country, then by weight. Each step is a record set with its own name; its CloudFront entry points to `cf.<host>`, its fallback entry to the next set, and the last fallback to `current.<host>`. `cf.<host>` holds the distribution's domain name (a CNAME on a subdomain, alias A and AAAA records at the apex); `current.<host>` holds today's production value. Fail back swaps the value of `cf.<host>` for the value of `current.<host>` in one all-or-none change. For the store: `www.example.com`, `geo.www.example.com`, `ramp.www.example.com`, `cf.www.example.com` and `current.www.example.com`.

| Step | Record set | Routing policy | CloudFront entry | Fallback entry |
|---|---|---|---|---|
| Entry | `<host>` | IP-based | Test cohort CIDRs → `cf.<host>` | `*` (every other resolver address) → `geo.<host>` |
| Pilot | `geo.<host>` | Geolocation | Pilot country → `cf.<host>` | Default (every other country) → `ramp.<host>` |
| Ramp | `ramp.<host>` | Weighted | CloudFront share → `cf.<host>` | Remaining share → `current.<host>` |

- [ ] **Create the whole tree, including the `*` and Default fallbacks, before you publish it.**  
  Why: A geolocation set without a Default entry returns no answer for countries you did not list. In the IP-based set, `*` is the entry for every resolver address outside the test CIDRs.  
  Read: [IP-based routing (Doc, R114)](#r114) — CIDR collections and `*`; [Geolocation routing (Doc, R115)](#r115) — the Default record.  
  Pillar: Reliability

- [ ] **Use one record type through each branch.**  
  Why: An alias must have the same type as its target, and an apex record cannot be a CNAME. Subdomain branches use CNAME-type records. A weighted apex works only if the current edge gives you static A and AAAA addresses; otherwise cut the apex over in one change, or redirect it to a migrated subdomain.  
  Read: [Values for alias records (Doc, R118)](#r118) — type rules; [Weighted routing (Doc, R104)](#r104) — weights.  
  Pillar: Reliability

- [ ] **At the apex, decide when to publish the AAAA alias.**  
  Why: Apex A and AAAA records are separate weighted sets. If the current edge has no IPv6 address, leave the AAAA alias out until 100%. Or accept that clients that prefer IPv6 reach CloudFront from the first step, outside the cohort controls. CNAME-type branches do not have this problem.  
  Read: [Values for alias records (Doc, R118)](#r118) — A and AAAA aliases.  
  Pillar: Reliability

- [ ] **Give each edge a dedicated health hostname.**  
  Why: A check on the production hostname tests whichever edge DNS picks. Allow the health path through the Host, Challenge, bot and rate rules by path plus the Route 53 health-checker IP ranges. Do not attach checks to the weighted records unless you chose automatic failover: a failed current-edge check would send everyone to a cold CloudFront cache.  
  Read: [Values for Route 53 health checks (Doc, R123)](#r123) — Host, SNI and IP-version behavior; [Health-based detection with Shield Advanced (Doc, R503)](#r503) — associating the check.  
  Pillar: Reliability

### Observe

Every rollout gate in [3. Migrate](03-migrate.md) is a decision you make from one dashboard. Build it now and run it against the tests below, so the first production cohort is not its first use.

- [ ] **Turn on additional metrics and build one gate dashboard.**  
  Why: One dashboard shows traffic share, errors, origin latency, cohort hit rate, AWS WAF actions, Shield events, health checks, RUM and the business KPI. Hit rate and origin latency are additional metrics that you turn on; CloudFront alarms live in `us-east-1`.  
  Read: [CloudFront additional metrics (Doc, R702)](#r702) — hit rate and origin latency; [CloudFront metrics in CloudWatch (Doc, R701)](#r701) — metrics and Region.  
  Pillar: Operational Excellence

- [ ] **Turn on the logs the migration needs.**  
  Why: CloudFront standard logs for the record, sampled real-time logs for per-country hit rate during rollout, and AWS WAF logs with filters and redacted fields.  
  Read: [CloudFront standard logs (Doc, R703)](#r703) — durable logs; [CloudFront real-time logs (Doc, R704)](#r704) — sampled, best-effort delivery; [AWS WAF logs (Doc, R706)](#r706) — filters and redaction.  
  Pillar: Operational Excellence

### Test

Everything runs on the test hostname with the production tree shape, before any public user reaches CloudFront. The fail-back test is the one that opens Phase 4.

- [ ] **Compare both edges on every functional path.**  
  Why: Status codes, headers, cookies, redirects, HSTS, cache behavior, client-IP headers and origin routing must match. Test through a test hostname of your own that resolves to the new distribution, or a local DNS override.  
  Read: [CloudFront cache behavior settings (Doc, R245)](#r245) — request behavior.  
  Pillar: Reliability

- [ ] **Run a two-user cache test on every personalized path.**  
  Why: It finds a cache key that serves one user's basket or account data to another.  
  Read: [Managed cache policies (Doc, R207)](#r207) — cache-key controls.  
  Pillar: Security

- [ ] **Replay the regression set against every protection-pack version.**  
  Why: Known-good login, checkout, API and upload requests expose false positives before users do.  
  Read: [Testing and tuning AWS WAF protections (Doc, R401)](#r401) — sampled requests and Count.  
  Pillar: Reliability

- [ ] **Load-test a separate test distribution.**  
  Why: AWS's load-testing policy restricts which distribution configurations you can test against. Build a test distribution that follows it, and keep the production distribution out of the test.  
  Read: [Load testing CloudFront (Doc, R801)](#r801) — topology and exclusions.  
  Pillar: Performance Efficiency

- [ ] **Test fail back on the test hostname with the production tree shape.**  
  Why: Build the same record tree for the test hostname and change `cf.<testhost>` in one batch. Time how long CloudFront's request count takes to fall below the Phase 1 drained value, applied here to CloudFront. DNS caches and reused connections keep clients on CloudFront past the TTL.  
  Read: [ChangeResourceRecordSets (API Reference, R124)](#r124) — all-or-none change batches.  
  Pillar: Reliability

- [ ] **Refine the cost estimate from the staged design.**  
  Why: Traffic by geography, CloudFront requests and transfer, Origin Shield, AWS WAF rule groups, logs, functions, Route 53 and API Gateway now have real configuration behind them.  
  Read: [AWS Pricing Calculator (Tool, R911)](#r911) — estimate; [AWS WAF pricing (Pricing, R913)](#r913) — rule and inspection drivers.  
  Pillar: Cost Optimization

**Phase 3 gate.** Functional, cache-isolation, regression, load and fail-back tests pass. The dashboard, runbooks, both origin access paths, the DNS tree and the refined estimate are ready.

[Next: 3. Migrate →](03-migrate.md)

## References

| Ref | Resource | Type | What you get |
|---|---|---|---|
| <a id="r101"></a>R101 | [Making Route 53 the DNS service for a domain in use](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/migrate-dns-domain-in-use.html) | Doc | Zone migration, TTLs, name servers and DNSSEC |
| <a id="r104"></a>R104 | [Weighted routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html) | Doc | DNS traffic weighting |
| <a id="r114"></a>R114 | [IP-based routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-ipbased.html) | Doc | CIDR cohorts and the `*` fallback location |
| <a id="r115"></a>R115 | [Geolocation routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html) | Doc | Country steering and the Default record |
| <a id="r118"></a>R118 | [Values for alias records](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-values-alias.html) | Doc | Alias type and target rules |
| <a id="r121"></a>R121 | [Quotas for Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html) | Doc | Record, value and hosted zone limits |
| <a id="r122"></a>R122 | [Routing traffic for subdomains](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-routing-traffic-for-subdomains.html) | Doc | Delegating a steering subdomain to Route 53 |
| <a id="r123"></a>R123 | [Values for Route 53 health checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-creating-values.html) | Doc | Endpoint names, Host, SNI and IPv4 behavior |
| <a id="r124"></a>R124 | [ChangeResourceRecordSets](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html) | API Reference | Transactional, all-or-none DNS change batches |
| <a id="r203"></a>R203 | [CloudFront certificate requirements](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html) | Doc | Viewer certificate names and Region |
| <a id="r204"></a>R204 | [CloudFront VPC origins](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html) | Doc | Private origins and feature limits |
| <a id="r205"></a>R205 | [Origin access control](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-origin.html) | Doc | Restricting supported AWS origins to CloudFront |
| <a id="r207"></a>R207 | [Managed cache policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html) | Doc | Cache-key controls |
| <a id="r208"></a>R208 | [Managed origin request policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-origin-request-policies.html) | Doc | Values sent to origins without entering the cache key |
| <a id="r209"></a>R209 | [Managed response headers policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html) | Doc | Security and CORS headers |
| <a id="r210"></a>R210 | [CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) | Doc | An extra cache layer that reduces origin load |
| <a id="r215"></a>R215 | [CloudFront cache-tag invalidation](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/invalidation-by-tags.html) | Doc | Invalidating objects by tag |
| <a id="r245"></a>R245 | [CloudFront cache behavior settings](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html) | Doc | Behavior order, methods and protocol policy |
| <a id="r247"></a>R247 | [HTTPS to custom origins](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-cloudfront-to-custom-origin.html) | Doc | Origin certificate-name matching |
| <a id="r251"></a>R251 | [Restricting access to an Application Load Balancer](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html) | Doc | Secret origin headers and rotation |
| <a id="r401"></a>R401 | [Testing and tuning AWS WAF protections](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-testing.html) | Doc | Count-to-production rollout |
| <a id="r404"></a>R404 | [AWS WAF Bot Control](https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-bot.html) | Doc | Common and targeted rules and labels |
| <a id="r409"></a>R409 | [Automatic AWS WAF SDK injection](https://docs.aws.amazon.com/waf/latest/developerguide/waf-challenge-javascript-sdk-injection.html) | Doc | Side effects and unsupported configurations |
| <a id="r411"></a>R411 | [AWS WAF body inspection limits](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-setting-body-inspection-limit.html) | Doc | Inspection limits by resource |
| <a id="r412"></a>R412 | [Oversize request components](https://docs.aws.amazon.com/waf/latest/developerguide/waf-oversize-request-components.html) | Doc | Continue, Match or No match |
| <a id="r415"></a>R415 | [Rule action overrides](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-rule-group-override-options.html) | Doc | Changing one managed rule's action |
| <a id="r417"></a>R417 | [Label-match rules](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-label-match.html) | Doc | Acting on labels from earlier rules |
| <a id="r419"></a>R419 | [Example scenarios of false positives with Bot Control](https://docs.aws.amazon.com/waf/latest/developerguide/waf-bot-control-false-positives.html) | Doc | Known false-positive cases and the user-agent exception |
| <a id="r426"></a>R426 | [Implementing a bot control strategy on AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/bot-control/introduction.html) | Guide | Bot-control planning and operations |
| <a id="r428"></a>R428 | [Intelligent threat mitigation best practices](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-protections-best-practices.html) | Doc | Rule order and scope-down guidance |
| <a id="r448"></a>R448 | [Request components in AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-fields-list.html) | Doc | JSON body parsing and invalid-body fallback |
| <a id="r453"></a>R453 | [Using the anti-DDoS rule group](https://docs.aws.amazon.com/waf/latest/developerguide/waf-anti-ddos-rg-using.html) | Doc | Priority, testing, sensitivity and URI exemptions |
| <a id="r460"></a>R460 | [AWS WAF rule statements](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statements.html) | Doc | Match and logical statements |
| <a id="r462"></a>R462 | [Resources that AWS WAF can protect](https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works-resources.html) | Doc | CloudFront scope and Region placement |
| <a id="r464"></a>R464 | [Application Security: Managing false positives in AWS WAF](https://builder.aws.com/content/2wCgDzCjFNtJgRM4yhNu9GCNCXx/application-security-managing-false-positives-in-aws-waf) | Builder Center | Designing, detecting and excluding false positives |
| <a id="r465"></a>R465 | [Enhance your security posture and reduce false positives using client JA3 fingerprint and HTTP header order](https://aws.amazon.com/blogs/networking-and-content-delivery/enhance-your-security-posture-and-reduce-false-positives-using-client-ja3-fingerprint-and-http-header-order/) | Blog | Signals beyond IP for rate and block rules |
| <a id="r466"></a>R466 | [How do I detect false positives caused by AWS Managed Rules?](https://repost.aws/knowledge-center/waf-detect-false-positives-from-amrs) | AWS re:Post | Reading the log and fixing with labels or scope-down |
| <a id="r501"></a>R501 | [AWS best practices for DDoS resiliency](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/aws-best-practices-ddos-resiliency.html) | Guide | DDoS-resilient architectures |
| <a id="r502"></a>R502 | [Shield Advanced and the anti-DDoS managed rule group](https://aws.amazon.com/blogs/security/aws-shield-advanced-is-embracing-the-aws-waf-anti-ddos-managed-rule-group-what-changes-and-how-to-prepare/) | Blog | The change, its dates, alarms and IaC guidance |
| <a id="r503"></a>R503 | [Health-based detection with Shield Advanced](https://docs.aws.amazon.com/waf/latest/developerguide/ddos-advanced-health-checks.html) | Doc | Associating health checks |
| <a id="r601"></a>R601 | [Protecting API endpoints](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/protecting-api-endpoints-bp4.html) | Guide | CloudFront in front of Regional APIs |
| <a id="r602"></a>R602 | [API Gateway with a CloudFront distribution](https://repost.aws/knowledge-center/api-gateway-cloudfront-distribution) | AWS re:Post | Regional-target origin pattern |
| <a id="r604"></a>R604 | [API Gateway throttling](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-request-throttling.html) | Doc | Account, stage and method throttles |
| <a id="r606"></a>R606 | [REST API request validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-request-validation.html) | Doc | Models and parameter validation |
| <a id="r617"></a>R617 | [API Gateway custom domains behind CloudFront](https://repost.aws/knowledge-center/api-gateway-domain-cloudfront) | AWS re:Post | Custom-domain origin pattern |
| <a id="r618"></a>R618 | [Restricting an HTTP API with a Lambda authorizer](https://aws.amazon.com/blogs/networking-and-content-delivery/restricting-access-http-api-gateway-lambda-authorizer/) | Blog | Secret headers, authorizer checks and rotation |
| <a id="r701"></a>R701 | [CloudFront metrics in CloudWatch](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/monitoring-using-cloudwatch.html) | Doc | Metrics and Region |
| <a id="r702"></a>R702 | [CloudFront additional metrics](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/viewing-cloudfront-metrics.html) | Doc | Cache hit rate and origin latency |
| <a id="r703"></a>R703 | [CloudFront standard logs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/standard-logging.html) | Doc | Durable logs and destinations |
| <a id="r704"></a>R704 | [CloudFront real-time logs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/real-time-logs.html) | Doc | Sampled per-request data within seconds |
| <a id="r706"></a>R706 | [AWS WAF logs](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html) | Doc | Destinations, filters and redaction |
| <a id="r801"></a>R801 | [Load testing CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/load-testing.html) | Doc | Test topology and exclusions |
| <a id="r806"></a>R806 | [Continuous deployment limits](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/continuous-deployment-quotas-considerations.html) | Doc | Traffic cap and incompatible features |
| <a id="r810"></a>R810 | [CloudFormation change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html) | Doc | Reviewing changes before deployment |
| <a id="r812"></a>R812 | [CloudFormation Hooks](https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/what-is-cloudformation-hooks.html) | Doc | Enforcing controls during deployment |
| <a id="r911"></a>R911 | [AWS Pricing Calculator](https://calculator.aws/) | Tool | Workload estimate |
| <a id="r913"></a>R913 | [AWS WAF pricing](https://aws.amazon.com/waf/pricing/) | Pricing | AWS WAF cost drivers |
