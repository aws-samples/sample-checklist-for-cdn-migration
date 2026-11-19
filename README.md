# CDN and WAF migration checklist

<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. SPDX-License-Identifier: MIT-0 -->

A four-file checklist for moving HTTP web and API delivery from a third-party content delivery network (CDN) and web application firewall (WAF) to AWS. The target services are Amazon CloudFront, AWS WAF, AWS Shield Advanced, Amazon Route 53 and Amazon API Gateway. It is written for the architects and engineers who run the current edge and will run the AWS one, and it assumes no prior use of AWS edge services.

**Core rule:** Move the CDN and WAF together for each hostname. Keep the current edge serving as the DNS fail-back target until the hold period ends.

This checklist describes the method. Service features, quotas and prices change; the linked pages hold the current values. Links and claims were last verified against AWS documentation on 2026-11-17.

## The journey

This checklist follows one hostname, an online store at `www.example.com`, from decision to decommission. Run it first on a low-risk PoC hostname, then on the store, then on each hostname after it.

You start by writing down why you are migrating and which numbers will prove it (Phase 1). You make the architecture decisions that everything later depends on: where DNS answers from, how you pay, which API type, what you build and what you buy (Phase 2). You build the AWS edge stack beside the current edge and restore every control the current edge enforces today. You prepare DNS records that can send any share of users to either edge, and you prove all of it on a test hostname, fail back included (Phase 3). You move real users in cohorts: your own staff, then one country, then a weighted ramp, with one DNS change ready to send everyone back. You hold at 100% until the current edge is drained, then decommission it in the safe order (Phase 4). You tune cost, performance and security on real traffic and hand the stack to the operating team (Phase 5).

Each phase ends with a gate. What you leave one file with is what the next file assumes you have.

| File | Who leads | You arrive with | You leave with (the gate) |
|---|---|---|---|
| This file: Phases 1 and 2 | Sponsor and architect | A reason to move and a contract renewal date | Approved goals, KPI targets, thresholds and architecture decisions |
| [2. Build and test](02-build-and-test.md): Phase 3 | Build team | The decisions above | An AWS edge stack beside the current edge, parity rules in place, the DNS tree built, every test passed including fail back |
| [3. Migrate](03-migrate.md): Phase 4 | Migration lead and on-call | The tested stack and the Phase 1 thresholds | Every user on CloudFront, hold period passed, current edge drained and decommissioned |
| [4. Operate](04-operate.md): Phase 5 | Operating team | A live stack and its first month of data | KPIs at target, every runbook run once, handover done |

Each checklist item has four lines: the action, why it matters, what to read, and the Well-Architected pillar it serves. A **Read** link names the resource and its type, then jumps to the reference table at the bottom of that file. The table links to the public AWS page.

## Before you start

- Use an AWS account where the team can create CloudFront, AWS WAF, Shield Advanced, AWS Certificate Manager (ACM) and Route 53 resources.
- CloudFront reads what you attach to a distribution (protection packs, viewer certificates, Lambda@Edge functions) from the US East (N. Virginia) Region, `us-east-1`, and publishes its metrics there. Create those resources and their alarms in `us-east-1`, whatever Region your origin uses.
- Confirm the support plan meets the Shield Response Team (SRT) and proactive-engagement prerequisites.
- Pick one low-risk hostname for the proof of concept (PoC).
- Edge logs contain IP addresses and request data. Confirm retention, redaction, residency and third-party terms with your privacy and legal teams before you turn logs on.

## Terms

| Term | Meaning in this checklist |
|---|---|
| Current edge | The CDN and WAF you are leaving |
| Distribution | CloudFront's configuration for a hostname: origins, cache behaviors, certificate, edge functions and protection pack. What the current edge calls a property or site |
| AWS edge stack | The distribution, protection pack, Shield Advanced protection (the entry that tells Shield Advanced to watch the distribution), certificate and DNS records for one hostname |
| Policy | A named, reusable bundle of CloudFront settings: cache policy (cache key, TTLs), origin request policy (what the origin receives outside the cache key), response headers policy (headers added to responses) |
| Edge function | Code CloudFront runs on a request or response: a CloudFront Function or a Lambda@Edge function |
| Origin access path | How one edge reaches the origin and proves who it is: an IP allow rule, a secret header, OAC or a VPC origin. Not CloudFront's "origin path" URL-prefix setting |
| Protection pack | The AWS WAF web ACL attached to CloudFront. The only WAF in this design |
| Managed rule group | A prebuilt rule set from AWS (Core rule set, IP reputation, Bot Control, ATP, ACFP, anti-DDoS) or from AWS Marketplace |
| Rule action | Count (record only), CAPTCHA or Challenge (the client proves it is a human or a browser), or Block. New rules start in Count |
| Label | A tag that one AWS WAF rule adds to a request and a later rule acts on |
| Token | A cookie AWS WAF sets after a CAPTCHA or Challenge. Some rules need it |
| Positive model | Rules that allow only the hosts, methods, paths, content types and sizes the application accepts, and block the rest |
| Parity rule | A rule that enforces traffic on the current edge today. It must enforce on CloudFront before any public cohort moves |
| New rule | An AWS control the current edge does not have: positive-model rules, Bot Control, ATP, ACFP |
| Alias record | A Route 53 A or AAAA record that answers with its target's addresses. The target is an AWS resource or another record in the same zone. Allowed at the apex, where a CNAME is not |
| Cohort | The set of users whose DNS answers point to CloudFront at a given rollout stage |
| Test cohort | Your own staff and test agents, which you select by resolver IP range |
| Pilot geography | One low-risk country, which you select by geolocation |
| Shared leaf | The one DNS name (`cf.<host>`) that every CloudFront-facing branch points to. Fail back changes only this name's records, in one all-or-none change |
| Regression set | Known-good requests (login, checkout, API, upload) that you replay after every protection-pack change |
| Fail back | Return traffic to the current edge through DNS |
| Rollback | Restore an earlier AWS configuration through infrastructure as code (IaC) |
| Hold period | Time at 100% CloudFront before decommissioning. Set in the Phase 1 gate |
| Drained | Current-edge traffic at or below the residual threshold for the agreed duration. Set in the Phase 1 gate |

Acronyms: ACFP (account creation fraud prevention), ACM (AWS Certificate Manager), ALB (Application Load Balancer), ASN (autonomous system number), ATP (account takeover prevention), CAA (Certification Authority Authorization), CDK (AWS Cloud Development Kit), CIDR (IP address range notation), CORS (cross-origin resource sharing), DDoS (distributed denial of service), DNSSEC (DNS Security Extensions), DS (Delegation Signer), HSTS (HTTP Strict Transport Security), IaC (infrastructure as code), IAM (AWS Identity and Access Management), KPI (key performance indicator), mTLS (mutual TLS), NS (name server record), OAC (origin access control), PoC (proof of concept), RUM (real user monitoring), SaaS (software as a service), SDK (software development kit), SNI (Server Name Indication), SPA (single-page application), SRT (Shield Response Team), TTL (time to live), VPC (Amazon Virtual Private Cloud).

## Completion criteria

The migration is complete when:

- All traffic for the hostname runs through the AWS edge stack.
- Parity rules enforce the same outcomes as the current edge's WAF.
- Performance, availability, security and business KPIs meet their Phase 1 targets.
- The hold period has passed and traffic to the current edge is drained.
- Fail back, false-positive response, DDoS response, invalidation and certificate-renewal runbooks have each run once.
- You have removed the current edge's DNS records, origin access, secrets and certificates in the order given in [3. Migrate](03-migrate.md).

## Phase 1: Define why you are migrating

The goals, KPIs and thresholds you set here are the gates for every later phase. Common drivers: cost, one toolchain, availability, performance, security, vendor consolidation, a wider AWS migration, a merger. Reasons not to migrate: a feature gap, a team built for buying instead of building, a multi-cloud requirement. Write both lists down; Phase 2 turns each gap into a native, build or third-party decision.

- [ ] **Write one or two measurable migration goals.**  
  Why: A goal with an endpoint, an outcome and a false-positive guardrail produces a clear go or no-go decision at the end.  
  Read: [Migration playbook for AWS large migrations (Guide, R006)](#r006) — wave planning and success criteria.  
  Pillar: Operational Excellence

- [ ] **Define tier 1 and tier 2 KPIs with target values.**  
  Why: Tier 1 proves the goal: cost by resource and by hour, web performance from synthetic tests and Amazon CloudWatch RUM, and one business outcome such as buy rate. Tier 2 catches regressions: error rate, bytes, requests and AWS WAF block rate. Every rollout gate uses these numbers.  
  Read: [CloudFront metrics in CloudWatch (Doc, R701)](#r701) — delivery metrics; [CloudWatch RUM (Doc, R710)](#r710) — user experience.  
  Pillar: Operational Excellence

- [ ] **Capture one normal business cycle of baseline data from the current edge and origin.**  
  Why: The baseline becomes the pass and fail threshold for staging and rollout. It has to come from the current edge's logs, because CloudFront logs start only after the build.  
  Read: [CloudFront standard logs (Doc, R703)](#r703) — fields to match when you map the current log format.  
  Pillar: Reliability

- [ ] **Set the migration deadline before the contract renewal date.**  
  Why: Recovery time before renewal stops a failed step from forcing a rushed cutover or an extension.  
  Read: [Migration playbook for AWS large migrations (Guide, R006)](#r006) — cutover planning and ownership.  
  Pillar: Operational Excellence

**Phase 1 gate.** The sponsor approves the goals, the KPI targets and the baseline, and signs off these values:

| Threshold | Value |
|---|---|
| Hold period at 100% CloudFront | `____ days` (at least the longest previous TTL plus one business cycle) |
| Drained | `≤ ____ current-edge requests per minute for ____ consecutive hours` |
| False-positive ceiling for parity rules | `____ %` of requests, which you measure on the regression set and sampled requests |
| False-positive ceiling for new rules before they leave Count | `____ %` of requests, set tighter than the parity ceiling |
| Origin latency ceiling | `____ ms` at the 90th percentile, below the origin response timeout |
| Origin error ceiling | `____ %` 5xx responses from the origin |
| Business KPI floor | `____` (for example, checkout completion rate) |

## Phase 2: Make the architecture decisions

Each decision here fixes something the build cannot change cheaply later. The decisions: where DNS answers from, how you pay, which API controls exist, whether you can test changes on live traffic, and what you build instead of buy. The Phase 2 gate records them in one place.

- [ ] **Decide whether the DNS zone moves to Route 53 before the edge, after it, or stays where it is.**  
  Why: The rollout steers traffic with Route 53 records, so Route 53 must answer for them, through the whole zone or a delegated subdomain. Move the zone first for a committed migration; delegate a subdomain for a PoC. Never change name servers and the edge in the same step.  
  Read: [Making Route 53 the DNS service for a domain in use (Doc, R101)](#r101) — zone migration order.  
  Pillar: Reliability

- [ ] **Choose the pricing model for each distribution and check its eligibility.**  
  Why: CloudFront offers pay-as-you-go pricing and flat-rate plans. Plans exclude some features and protections; compare the current list with the features this checklist uses before you commit.  
  Read: [CloudFront flat-rate pricing plans (Doc, R919)](#r919) — eligibility and exclusions.  
  Pillar: Cost Optimization

- [ ] **Choose the API type before you design API controls.**  
  Why: REST, HTTP and WebSocket APIs support different controls (request validation, usage plans, canary releases, authorizer types). The choice decides which API items in [2. Build and test](02-build-and-test.md) apply.  
  Read: [REST API and HTTP API differences (Doc, R614)](#r614) — feature comparison; [Request validation for WebSocket APIs (Doc, R619)](#r619) — route models.  
  Pillar: Reliability

- [ ] **Decide whether each distribution uses continuous deployment.**  
  Why: Continuous deployment tests a distribution change on a share of live traffic through a staging distribution. It restricts some features and some protection-pack changes, so decide before you build.  
  Read: [Continuous deployment limits (Doc, R806)](#r806) — restrictions and traffic limits.  
  Pillar: Reliability

- [ ] **Choose native, build or third-party for every feature gap.**  
  Why: Native first: hide the origin with OAC (CloudFront signs its requests to an AWS origin) or a VPC origin (an origin with no public address), cache policies, Bot Control, rate limits. Build second: redirects in CloudFront Functions, basic image optimization with the AWS Solution. Third-party last, through AWS Marketplace: advanced bot defense, API schema protection, advanced image operations.  
  Read: [CloudFront Functions and Lambda@Edge comparison (Doc, R301)](#r301) — runtime choice; [Dynamic Image Transformation for Amazon CloudFront (Solution, R309)](#r309) — image optimization; [AWS Marketplace managed rule groups (Doc, R424)](#r424) — add-on protections.  
  Pillar: Operational Excellence

### Pick the PoC starting point

Start with the low-risk hostname you picked in Before you start. The PoC proves the Phase 2 decisions on real configuration, and its distribution becomes the template for the store and every hostname after it.

| Workload | Starting configuration | Sample |
|---|---|---|
| Static site or single-page application | Amazon S3 origin with OAC, managed cache and response-header policies, CloudFront Function for route rewrites | [Secure static-site sample (Sample, R223)](#r223) |
| Dynamic web application (the store) | Application Load Balancer or Amazon EC2 origin, path-specific caching, a CloudFront-only origin access path alongside the current edge's | [Secure ecommerce demo (Sample, R222)](#r222) |
| API | Regional API Gateway behind CloudFront, caching off except on confirmed cacheable GET paths | [Bot-protected API sample (Sample, R437)](#r437) |

### Prepare the AWS environment

Access, quotas and help take days to arrive. Request them now so the build in [2. Build and test](02-build-and-test.md) starts the day the gate passes.

- [ ] **Grant least-privilege IAM access, raise quotas, and pick the deployment model.**  
  Why: The pipeline deploys; people get read access for testing. Raise quotas before the build, not during cutover. Decide now whether a central policy service manages the protection packs across accounts.  
  Read: [Quotas for CloudFront (Doc, R216)](#r216) — quotas and how to raise them; [What is AWS Network Security Manager? (Doc, R519)](#r519) — central AWS WAF and Shield Advanced policy.  
  Pillar: Operational Excellence

- [ ] **Line up help: your AWS account team, an AWS Partner if you need delivery capacity, and AI coding agents with current AWS skills.**  
  Why: The account team coordinates AWS-side preparation for large traffic shifts. AWS Partners add delivery capacity. Agents with current AWS skills translate rules and draft IaC that you review.  
  Read: [Agent Toolkit for AWS: install with the AWS CLI (Doc, R002)](#r002) — current skills for coding agents.  
  Pillar: Operational Excellence

**Phase 2 gate.** The architecture record names the DNS order, pricing model, API type, continuous-deployment choice, PoC hostname, an owner for every built or third-party capability, and the quota and access requests raised.

[Next: 2. Build and test →](02-build-and-test.md)

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for how to report a security issue.

## Contributing

Corrections and additions are welcome as issues and pull requests. See [CONTRIBUTING](CONTRIBUTING.md).

## License

The MIT-0 License covers this checklist. See [LICENSE](LICENSE).

## References

| Ref | Resource | Type | What you get |
|---|---|---|---|
| <a id="r002"></a>R002 | [Agent Toolkit for AWS: install with the AWS CLI](https://docs.aws.amazon.com/agent-toolkit/latest/userguide/aws-cli.html) | Doc | CLI commands to install and update AWS skills for your coding agent |
| <a id="r006"></a>R006 | [Migration playbook for AWS large migrations](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-migration-playbook/introduction.html) | Guide | Waves, ownership, success criteria and cutover planning |
| <a id="r101"></a>R101 | [Making Route 53 the DNS service for a domain in use](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/migrate-dns-domain-in-use.html) | Doc | Zone migration, TTLs, name servers and DNSSEC |
| <a id="r216"></a>R216 | [Quotas for CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html) | Doc | Self-service quota increases |
| <a id="r222"></a>R222 | [Secure ecommerce demo](https://github.com/aws-samples/fast-secure-ecommerce-demo) | Sample | Dynamic application with CloudFront and AWS WAF |
| <a id="r223"></a>R223 | [Secure static-site sample](https://github.com/aws-samples/amazon-cloudfront-secure-static-site) | Sample | S3, OAC, ACM and Route 53 |
| <a id="r301"></a>R301 | [CloudFront Functions and Lambda@Edge comparison](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions-choosing.html) | Doc | Runtime and event selection |
| <a id="r309"></a>R309 | [Dynamic Image Transformation for Amazon CloudFront](https://docs.aws.amazon.com/solutions/dynamic-image-transformation-for-amazon-cloudfront/) | Solution | Image resizing and format conversion behind CloudFront |
| <a id="r424"></a>R424 | [AWS Marketplace managed rule groups](https://docs.aws.amazon.com/waf/latest/developerguide/marketplace-rule-groups.html) | Doc | Third-party add-on protections |
| <a id="r437"></a>R437 | [Bot-protected API sample](https://github.com/aws-samples/aws-waf-bot-control-api-protection-with-captcha) | Sample | SPA tokens, Bot Control and CAPTCHA |
| <a id="r519"></a>R519 | [What is AWS Network Security Manager?](https://docs.aws.amazon.com/network-security-manager/latest/devguide/what-is.html) | Doc | Central AWS WAF and Shield Advanced policy across accounts |
| <a id="r614"></a>R614 | [REST API and HTTP API differences](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vs-rest.html) | Doc | Feature comparison |
| <a id="r619"></a>R619 | [Request validation for WebSocket APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-request-validation.html) | Doc | Route request models and the `$default` model |
| <a id="r701"></a>R701 | [CloudFront metrics in CloudWatch](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/monitoring-using-cloudwatch.html) | Doc | Metrics and Region |
| <a id="r703"></a>R703 | [CloudFront standard logs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/standard-logging.html) | Doc | Durable logs and destinations |
| <a id="r710"></a>R710 | [CloudWatch RUM](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM.html) | Doc | Real-user timing and errors |
| <a id="r806"></a>R806 | [Continuous deployment limits](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/continuous-deployment-quotas-considerations.html) | Doc | Traffic cap and incompatible features |
| <a id="r919"></a>R919 | [CloudFront flat-rate pricing plans](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/flat-rate-pricing-plan.html) | Doc | Eligibility and excluded features |
