# 4. Operate

<!-- Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved. SPDX-License-Identifier: MIT-0 -->

[← 3. Migrate](03-migrate.md) · [Start](README.md)

You arrive with every user on CloudFront, the current edge gone and a first month of real data. This file tunes cost, performance and security on that data and moves the new rules out of Count one at a time. Then it hands the stack to the operating team with reviewed IaC and tested runbooks. You leave with the Phase 5 gate and a migration that meets the completion criteria in the [README](README.md#completion-criteria).

## Phase 5: Optimize after cutover

### Tune cost and performance

The refined estimate from Phase 3 meets the first bill, and the migration-only settings come off.

- [ ] **Compare the first month of actual cost with the refined estimate, line by line.**  
  Why: AWS Cost Explorer per resource and per hour shows where the estimate was wrong: Bot Control scope, log volume, transfer, edge functions, Origin Shield or a cache setting. Alert on the gap with AWS Budgets.  
  Read: [AWS Pricing Calculator (Tool, R911)](#r911) — the estimate; [AWS Budgets (Doc, R926)](#r926) — alerts.  
  Pillar: Cost Optimization

- [ ] **Turn down migration-only observability.**  
  Why: Lower real-time log sampling, apply AWS WAF logging filters, and set retention at each log destination. Cutover logging keeps billing after it stops being useful.  
  Read: [AWS WAF logs (Doc, R706)](#r706) — filters and redaction; [CloudFront real-time logs (Doc, R704)](#r704) — sampling rate.  
  Pillar: Cost Optimization

- [ ] **Raise the cache hit ratio, confirm compression, and serve modern image formats.**  
  Why: Remove cache-key values the origin does not vary on, lengthen TTLs on versioned objects, confirm CloudFront compresses every compressible content type, and serve AVIF or WebP to clients that accept them.  
  Read: [Increase the CloudFront cache hit ratio (Doc, R243)](#r243) — cache levers; [Serve compressed files (Doc, R244)](#r244) — compression rules.  
  Pillar: Performance Efficiency

### Harden security

The new rules that waited in Count through the rollout now earn their action, one at a time, against the same Phase 1 ceiling.

- [ ] **Harden rules and remove capacity bloat.**  
  Why: Move remaining Count rules to their action, raise anti-DDoS sensitivity where the data supports it, and re-scope paid rule groups to the paths that need them. Every rule costs capacity units and a protection pack has a fixed maximum, so remove unused rules, duplicate groups and oversized regex sets.  
  Read: [Web ACL capacity units (Doc, R422)](#r422) — capacity and cost; [Intelligent threat mitigation best practices (Doc, R428)](#r428) — scoping paid rule groups.  
  Pillar: Cost Optimization

- [ ] **Promote new rules one at a time.**  
  Why: Move positive-model, Bot Control, ATP and ACFP rules out of Count only when the regression set passes and the false-positive rate is under the Phase 1 ceiling.  
  Read: [Testing and tuning AWS WAF protections (Doc, R401)](#r401) — safe promotion.  
  Pillar: Security

- [ ] **Pin managed rule group versions and track their expiry.**  
  Why: A pinned version changes only when you choose, so a rule update does not reach production unannounced. Pinned versions expire; track the dates and move before they do.  
  Read: [Managed rule group versions (Doc, R413)](#r413) — static and default versions; [Expiration of managed rule group versions (Doc, R458)](#r458) — expiry behavior.  
  Pillar: Reliability

### Manage change

The migration ran through a pipeline; so does everything after it. The people who run it from here need the design reasons, not only the templates.

- [ ] **Keep every change in the IaC pipeline.**  
  Why: Use change-set review, a test deployment and scheduled drift detection. Console changes are for break-glass response only, and you reconcile them into IaC afterward.  
  Read: [CloudFormation change sets (Doc, R810)](#r810) — review before deployment; [CloudFormation drift detection (Doc, R809)](#r809) — unmanaged changes.  
  Pillar: Operational Excellence

- [ ] **Use continuous deployment only for distribution changes it supports.**  
  Why: Continuous deployment tests a distribution change on a share of live traffic, with restrictions on features, traffic share and protection-pack association. Test protection-pack changes in Count instead.  
  Read: [Continuous deployment limits (Doc, R806)](#r806) — operational limits.  
  Pillar: Reliability

- [ ] **Name an owner and a runbook for each alarm.**  
  Why: False positives, DDoS events, invalidation, certificate renewal, fail back and cost anomalies each need a person and a tested procedure. Choose the governance model that decides who writes and approves rule changes.  
  Read: [Operating models for web application security governance (Blog, R521)](#r521) — central, distributed and shared ownership.  
  Pillar: Operational Excellence

- [ ] **Hand over with training and developer sandboxes.**  
  Why: Walk the operating team through each distribution, protection pack and record tree with the design reasons. Give developers sandbox distributions and protection packs in a non-production account.  
  Read: [CloudFront continuous deployment sample (Sample, R807)](#r807) — a pipeline pattern for the sandbox.  
  Pillar: Operational Excellence

- [ ] **Complete a Well-Architected review.**  
  Why: Record each open item across the six pillars with an owner and a date.  
  Read: [AWS Well-Architected Framework (Guide, R901)](#r901) — the six pillars.  
  Pillar: Operational Excellence

**Phase 5 gate.** KPIs meet their Phase 1 targets, every open risk has an owner and a date, and the operating team has run each runbook once.

## Out of scope

Video and media delivery, multi-tenant SaaS delivery and non-HTTP traffic need workload-specific guidance beyond this checklist. Third-party rule groups are add-on protections; evaluate them through AWS Marketplace and your third-party software review.

## References

| Ref | Resource | Type | What you get |
|---|---|---|---|
| <a id="r243"></a>R243 | [Increase the CloudFront cache hit ratio](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cache-hit-ratio.html) | Doc | Cache keys, TTLs and Origin Shield |
| <a id="r244"></a>R244 | [Serve compressed files](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/ServingCompressedFiles.html) | Doc | When CloudFront compresses, and when it does not |
| <a id="r401"></a>R401 | [Testing and tuning AWS WAF protections](https://docs.aws.amazon.com/waf/latest/developerguide/web-acl-testing.html) | Doc | Count-to-production rollout |
| <a id="r413"></a>R413 | [Managed rule group versions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-rule-groups-versioning.html) | Doc | Static and default versions |
| <a id="r422"></a>R422 | [Web ACL capacity units](https://docs.aws.amazon.com/waf/latest/developerguide/aws-waf-capacity-units.html) | Doc | How rule cost counts against web ACL capacity |
| <a id="r428"></a>R428 | [Best practices for intelligent threat mitigation](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-protections-best-practices.html) | Doc | Placing and scoping Bot Control, ATP and ACFP |
| <a id="r458"></a>R458 | [Expiration of managed rule group versions](https://docs.aws.amazon.com/waf/latest/developerguide/waf-managed-rule-groups-versioning-expiration.html) | Doc | What happens when a pinned version expires |
| <a id="r521"></a>R521 | [Operating models for web application security governance](https://aws.amazon.com/blogs/architecture/operating-models-for-web-app-security-governance-in-aws/) | Blog | Central, distributed and shared ownership |
| <a id="r704"></a>R704 | [CloudFront real-time logs](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/real-time-logs.html) | Doc | Sampled per-request data within seconds |
| <a id="r706"></a>R706 | [AWS WAF logs](https://docs.aws.amazon.com/waf/latest/developerguide/logging.html) | Doc | Destinations, filters and redaction |
| <a id="r806"></a>R806 | [Continuous deployment limits](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/continuous-deployment-quotas-considerations.html) | Doc | Traffic cap and incompatible features |
| <a id="r807"></a>R807 | [CloudFront continuous deployment sample](https://github.com/aws-samples/amazon-cloudfront-continuous-deployment) | Sample | A CDK pipeline that releases through a staging distribution |
| <a id="r809"></a>R809 | [CloudFormation drift detection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html) | Doc | Finding unmanaged changes |
| <a id="r810"></a>R810 | [CloudFormation change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-changesets.html) | Doc | Reviewing changes before deployment |
| <a id="r901"></a>R901 | [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/framework.html) | Guide | Six-pillar review |
| <a id="r911"></a>R911 | [AWS Pricing Calculator](https://calculator.aws/) | Tool | Workload estimate |
| <a id="r926"></a>R926 | [AWS Budgets](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html) | Doc | Budget thresholds and alerts |
