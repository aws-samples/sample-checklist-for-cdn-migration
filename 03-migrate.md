# 3. Migrate

<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. SPDX-License-Identifier: MIT-0 -->

[← 2. Build and test](02-build-and-test.md) · [Start](README.md) · [Next: 4. Operate →](04-operate.md)

You arrive with a tested stack, the DNS tree from [2. Build and test](02-build-and-test.md) and the Phase 1 thresholds. No public user has reached CloudFront yet. This file moves the store's users in cohorts, each one a DNS change. The current edge keeps serving everyone the tree has not yet moved and stays ready as the fail-back target. You leave with every user on CloudFront, the hold period passed and the current edge decommissioned.

## Phase 4: Execute the migration

### If DNS moves: change name servers first, as its own stage

Only if you chose to move the zone in Phase 2. The name-server change finishes and settles before any cohort moves, so you can tell a DNS problem from an edge problem.

- [ ] **Move the zone before any cohort moves, and let it settle.**  
  Why: Lower the NS TTL at both providers, remove the DS record from the parent, wait for the previous TTLs to expire, then change name servers at the registrar. Wait the parent zone's NS TTL before the first cohort so every resolver sees the new servers. Leave the zone at the current provider untouched so you can revert the name-server change.  
  Read: [Route 53 zone migration (Doc, R101)](#r101) — name-server and DNSSEC sequence.  
  Pillar: Reliability

- [ ] **Re-enable DNSSEC after the name-server change settles and before the first cohort moves.**  
  Why: Two providers cannot sign one zone, so the zone is unsigned from DS removal until Route 53 signing is on and you have published the new DS record. Keep that window short and record it as an accepted risk.  
  Read: [Route 53 zone migration (Doc, R101)](#r101) — DNSSEC steps.  
  Pillar: Security

### Roll out by cohort

Each stage is one DNS change to the tree from [2. Build and test](02-build-and-test.md), and each widens the audience: your staff, then one country, then a growing share of everyone.

| Stage | Who reaches CloudFront | DNS change | Move on when |
|---|---|---|---|
| Test cohort | Your resolver IP ranges | Publish the tree with your CIDRs in `<host>`, no pilot country, CloudFront weight 0 | Production tests pass; anti-DDoS and parity rules tuned in Count |
| Parity enforced | Same cohort | None | Every parity rule is in its production action |
| Pilot geography | One low-risk country | Add the country to `geo.<host>` | Gates pass for one business cycle |
| Weighted ramp | 1%, 5%, 25%, 50%, 100% of the remaining traffic | Raise the CloudFront weight in `ramp.<host>` | Gates pass at each step |
| Hold | Everyone | None; keep the whole tree until the hold period ends | Hold period passes with every alarm inside its threshold |

- [ ] **Move the test cohort first, with anti-DDoS and parity rules in Count.**  
  Why: Your own resolvers give the first production traffic for tuning with no customer exposure. Keep the anti-DDoS rule group in Count only as long as validation needs; while it counts, it mitigates nothing.  
  Read: [IP-based routing (Doc, R114)](#r114) — CIDR cohorts; [Using the anti-DDoS rule group (Doc, R453)](#r453) — Count, then Block.  
  Pillar: Reliability

- [ ] **Switch parity rules and the anti-DDoS rule group to their production actions before the pilot geography.**  
  Why: The first public user on CloudFront gets the same protection as on the current edge. Start the anti-DDoS rule group in Block at Low sensitivity, which acts only on the most obvious participants in a DDoS event. Turn on its Challenge option so suspected browsers get a silent check instead of a block, and exempt paths that do not return HTML, where the check cannot run.  
  Read: [Testing and tuning AWS WAF protections (Doc, R401)](#r401) — Count to action.  
  Pillar: Security

- [ ] **Open the pilot geography, then start the weighted ramp at 1%.**  
  Why: Each step limits the number of users a problem can reach. Measure the real share with the CloudFront request count and the current edge's analytics; DNS weights apply to resolver answers, not users, and real-time logs are sampled.  
  Read: [Geolocation routing (Doc, R115)](#r115) — pilot country; [Weighted routing (Doc, R104)](#r104) — weights.  
  Pillar: Reliability

### Keep the origin inside its capacity

Every cohort that moves meets a cold cache first. These items keep the origin from being the part that fails.

- [ ] **Size each step against spare origin capacity, and raise the origin scaling floor before the step.**  
  Why: New origin requests equal the added requests multiplied by the miss rate, which peaks right after each step. Before each geography opens, request its top URLs (from the current edge's logs) through the new distribution at a normal rate. Origin Shield then holds a copy, and edges fill from it instead of from the origin. This is not a load test.  
  Read: [Increase the CloudFront cache hit ratio (Doc, R243)](#r243) — cache levers; [CloudFront Origin Shield (Doc, R210)](#r210) — cache hierarchy.  
  Pillar: Reliability

- [ ] **Advance only when every gate passes.**  
  Why: The gates: real share stable, cohort hit rate no longer rising, origin latency and 5xx inside the Phase 1 ceilings, parity rules enforcing, false positives under the ceiling, business KPI holding.  
  Read: [CloudFront real-time logs (Doc, R704)](#r704) — per-country hit rate; [CloudFront additional metrics (Doc, R702)](#r702) — hit rate and origin latency.  
  Pillar: Reliability

- [ ] **Keep token-dependent new rules in Count until the weighted ramp reaches 100%.**  
  Why: Some ACFP, ATP and targeted Bot Control rules work only on requests that carry an AWS WAF token. Tokens issued on CloudFront do not exist on the current edge, and a user whose DNS answer changes moves between edges.  
  Read: [How AWS WAF uses tokens (Doc, R463)](#r463) — which rules need tokens.  
  Pillar: Reliability

### Respond and fail back

Three levers, smallest first. The runbook from [2. Build and test](02-build-and-test.md) says which one each alarm pulls.

- [ ] **Use the smallest recovery lever for each trigger.**  
  Why: An AWS WAF false positive needs one rule override to Count. A function or cache error needs an IaC rollback. DNS fail back is for edge-wide availability, latency or business-KPI failures only.  
  Read: [Rule action overrides (Doc, R415)](#r415) — one-rule change.  
  Pillar: Operational Excellence

- [ ] **Fail back by changing the shared leaf in one transactional change batch.**  
  Why: Every CloudFront-facing branch points to `cf.<host>`, so one `ChangeResourceRecordSets` batch returns all cohorts to the current edge. Keep CloudFront serving until its request count drains; answers already cached keep their TTL.  
  Read: [ChangeResourceRecordSets (API Reference, R124)](#r124) — all-or-none change batches.  
  Pillar: Reliability

- [ ] **Hold at 100% for the full hold period.**  
  Why: The current edge's certificates, WAF rules and origin access path stay unchanged until the hold period from the Phase 1 gate ends and the current edge is drained. Watch the current edge's request count fall below the drained threshold.  
  Read: [CloudFront metrics in CloudWatch (Doc, R701)](#r701) — request counts for the CloudFront side.  
  Pillar: Reliability

### Decommission the current edge

Only after the hold period has passed and the current edge is drained, both as set in the Phase 1 gate. The order below matters: DNS first, then access, then certificates, then the contract.

- [ ] **Collapse the rollout tree to simple records and raise TTLs.**  
  Why: The IP-based, geolocation and weighted records have done their job; simple alias or CNAME records are easier to run.  
  Read: [Weighted routing (Doc, R104)](#r104) — removing weighted sets.  
  Pillar: Operational Excellence

- [ ] **Remove the current edge's DNS records and health checks, then wait for their TTLs to expire.**  
  Why: Deleting the current-edge resource while a record still points at it leaves a dangling record that an unauthorized party can claim.  
  Read: [Threat tactic spotlight: Subdomain takeover (Blog, R255)](#r255) — dangling records; [Remove an alternate domain name (Doc, R254)](#r254) — DNS before resource removal.  
  Pillar: Security

- [ ] **Remove the current edge's origin access path and its Allow rules, then rotate every secret and key it held.**  
  Why: The origin must accept only CloudFront, and any credential the current edge saw is no longer trusted.  
  Read: [Restricting access to an Application Load Balancer (Doc, R251)](#r251) — CloudFront-only origin access.  
  Pillar: Security

- [ ] **Remove the current edge's certificates and validation records, keep the ACM validation records, and end the contract last.**  
  Why: ACM renews CloudFront's certificate through its own DNS validation records; removing those breaks renewal. The contract ends only when nothing depends on the current edge.  
  Read: [DNS validation in ACM (Doc, R125)](#r125) — validation records and renewal.  
  Pillar: Security

**Phase 4 gate.** The hold period has passed, the current edge is drained, you have tested fail back, and you have recorded the decommissioning evidence.

[Next: 4. Operate →](04-operate.md)

## References

| Ref | Resource | Type | What you get |
|---|---|---|---|
| <a id="r101"></a>R101 | [Making Route 53 the DNS service for a domain in use](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/migrate-dns-domain-in-use.html) | Doc | Zone migration, TTLs, name servers and DNSSEC |
| <a id="r104"></a>R104 | [Weighted routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html) | Doc | DNS traffic weighting |
| <a id="r114"></a>R114 | [IP-based routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-ipbased.html) | Doc | CIDR cohorts and the `*` fallback location |
| <a id="r115"></a>R115 | [Geolocation routing](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html) | Doc | Country steering and the Default record |
| <a id="r124"></a>R124 | [ChangeResourceRecordSets](https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html) | API Reference | Transactional, all-or-none DNS change batches |
| <a id="r125"></a>R125 | [DNS validation in AWS Certificate Manager](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html) | Doc | Validation CNAME records and automatic renewal |
| <a id="r210"></a>R210 | [CloudFront Origin Shield](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) | Doc | An extra cache layer that reduces origin load |
| <a id="r243"></a>R243 | [Increase the CloudFront cache hit ratio](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cache-hit-ratio.html) | Doc | Cache keys, TTLs and Origin Shield |
| <a id="r251"></a>R251 | [Restricting access to an Application Load Balancer](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html) | Doc | Secret origin headers and rotation |
| <a id="r254"></a>R254 | [Remove an alternate domain name](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/alternate-domain-names-remove-domain.html) | Doc | DNS-first removal order |
| <a id="r255"></a>R255 | [Threat tactic spotlight: Subdomain takeover](https://aws.amazon.com/blogs/security/threat-tactic-spotlight-subdomain-takeover/) | Blog | Dangling-record risks |
| <a id="r401"></a>R401 | [Testing and tuning AWS WAF protections](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-testing.html) | Doc | Count-to-production rollout |
| <a id="r415"></a>R415 | [Rule action overrides](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-rule-group-override-options.html) | Doc | Changing one managed rule's action |
| <a id="r453"></a>R453 | [Using the anti-DDoS rule group](https://docs.aws.amazon.com/waf/latest/developerguide/waf-anti-ddos-rg-using.html) | Doc | Priority, testing, sensitivity and URI exemptions |
| <a id="r463"></a>R463 | [How AWS WAF uses tokens](https://docs.aws.amazon.com/waf/latest/developerguide/waf-tokens-usage.html) | Doc | Which rules require tokens |
| <a id="r701"></a>R701 | [CloudFront metrics in CloudWatch](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/monitoring-using-cloudwatch.html) | Doc | Metrics and Region |
| <a id="r702"></a>R702 | [CloudFront additional metrics](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/viewing-cloudfront-metrics.html) | Doc | Cache hit rate and origin latency |
| <a id="r704"></a>R704 | [CloudFront real-time logs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/real-time-logs.html) | Doc | Sampled per-request data within seconds |
