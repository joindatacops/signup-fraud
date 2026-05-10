# How PillarlabAI was built: the conversion-pollution thesis behind DataCops

Let's start with the moment that became the company.

A SaaS marketing team we were working with had spent four months optimizing their Meta CAPI events. Event Match Quality climbed from 6.2 to 8.4. CPA dropped 11 percent. Then it stopped dropping. Then it crept back up. The team installed Sift to filter signup fraud. The risk scores looked clean. CAPI events kept flowing. CPA kept climbing. Meta started reporting strange audience drift in the Lookalike models.

A month of forensics later, the answer was obvious in retrospect. The fraud signups were getting filtered by Sift on the risk dashboard. Real customers, the team thought. But the same signups had already fired CAPI events to Meta during the form submit. Meta had already trained on them. The Lookalike was pointed at fake users. The CPA was climbing because the bidding model was hunting for more of the same fake users.

Sift did its job. The risk team saw the fraud. The marketing team didn't. The CAPI pipeline didn't. The bidding model trained on garbage anyway.

That's the gap PillarlabAI was built to close. We call it conversion pollution. It's the silent failure mode where a fraud-tool-clean signup still trains your ad algorithms on garbage and inflates CAC by 20 to 30 percent. Every existing tool in the category (Sift, SEON, Fingerprint, Arkose) ships to risk teams. None of them sit in the marketer's CAPI pipeline.

This is the founding story of DataCops and PillarlabAI. The numbers, the gap, the architectural choice, and what we shipped.

---

## Quick stuff people keep asking

**What is signup fraud actually?** In 2026 it's four things at once. Bots scraping free trials. Free-trial abusers stacking fake accounts to extend free credits. Synthetic identities passing KYC and seeding longer cons. Competitor scrapers pretending to be customers. Each requires different signal classes (device, behavioral, identity, network) and most tools only handle one or two.

**How big is the problem?** Intellicheck pegged 8.3 percent of digital account creations as suspected fraudulent in H1 2025. Industry composite data puts SaaS free-trial fraud at 20 to 30 percent of new account creations. Synthetic account fraud attempts grew 153 percent year over year per FinTech Global. Bots reached 46 percent of online signups in 2025.

**What does conversion pollution mean?** Fraudulent signups don't just waste seats. They corrupt downstream optimization. Meta CAPI events from fake users train Lookalike audiences on fake patterns. Smart Bidding learns to find more of the same fake users. CPA rises. The fraud-tool risk dashboard says clean. The bidding model is being poisoned anyway.

**Why are existing fraud tools missing this?** They were built for risk teams, not marketers. Sift, SEON, Fingerprint, Arkose all ship strong risk dashboards. None of them filter the CAPI event before it hits Meta or Google. The architectural assumption is that risk and ad attribution are separate problems. They aren't anymore.

**What's the DataCops compliance posture?** GDPR-compliant data processing active. CCPA active. TCF 2.2 first-party consent active. EU and US data residency available. SOC 2 Type II in progress, not complete. ISO 27001 planned. We don't gate features behind certifications we don't hold yet. We say so on the site.

---

## The conversion pollution thesis

This is the unique frame the existing signup-fraud category misses.

Every existing signup-fraud tool ships to a risk team. The risk team sees the fraud, scores it, blocks the signups they're confident about, queues the rest for review. Done.

The marketer never sees the fraud the risk team blocked. The marketer also never sees the fraud the risk team scored 'medium' and let through. The marketer's view is the CAPI event log and the Meta ROAS dashboard and the Smart Bidding performance.

When fraud signups fire CAPI events before the risk team's score arrives, the bidding model trains on those events. Lookalikes learn to find more of the same. CPA climbs. The risk dashboard still shows clean.

This is conversion pollution. Two failure modes:

1. Real-time CAPI events fire on signup before any fraud scoring completes. The bidding model already saw the event. Filtering after the fact doesn't unwind the training.

2. Fraud scoring throttles for cost reasons (per-API-call pricing on tools like SEON, Sift) and only the highest-confidence fraud gets blocked. The medium-risk band leaks into CAPI.

The fix is not 'better risk scoring'. The fix is filtering at the CAPI event layer, not at the dashboard layer. Decide which signups fire CAPI events at all, before they fire.

That's the architectural choice PillarlabAI is built around.

---

## The four-axis taxonomy

We built PillarlabAI around the four classes of signup fraud, not a single 'is this signup bad' score.

**Axis 1: Bots.** Automated signups by scrapers, scripted attackers, or cloud-hosted automation. Detection signal: IP class (datacenter vs residential vs mobile carrier), browser fingerprint anomalies, behavioral patterns (form-fill speed, mouse motion, clipboard).

**Axis 2: Free-trial abusers.** Real humans creating multiple accounts to stack free credits. Detection signal: behavioral patterns (the same fingerprint or device returning), email infrastructure (subaddressing, throwaway domains), payment method reuse.

**Axis 3: Synthetic identities.** Fabricated identities (often AI-generated) passing email checks, browser checks, and even some KYC. Detection signal: identity enrichment, social graph absence, infrastructure correlation across multiple synthetic accounts.

**Axis 4: Competitor scrapers.** Real humans (or their agents) signing up to scrape product, pricing, or feature data. Detection signal: behavioral patterns post-signup (trial usage shape), referrer and IP class, account graph.

Each axis requires different signal classes. Most tools in the category strong-cover one axis and skip the rest. Sift is strong on Axis 1 plus Axis 2. SEON is strong on Axis 3. FingerprintJS is one signal class within Axis 1. PillarlabAI was built to score across all four axes at once, with the scoring tied to the CAPI event decision rather than the risk dashboard.

---

## Tier 1: the existing risk-team tools

These ship to risk and security teams. None of them sit in the marketer's CAPI pipeline.

**1. Sift**

The Good: Enterprise-grade. ThreatClusters consortium model. Strong on ATO and Axis 1 plus Axis 2.

Frustrations: Risk-team-shaped product. Per-API-call pricing limits real-time CAPI gating. Long sales cycle. Six-figure pricing typical.

Wish List: CAPI event filter. Marketer-facing dashboard.

Value for Money: 8/10 enterprise risk.

Pricing: Six figures.

---

**2. SEON**

The Good: Strong identity enrichment. Social profile lookups. EU-friendly data residency.

Frustrations: Per-API-call pricing. UI is heavier than competitors. Risk-team-shaped.

Wish List: CAPI integration.

Value for Money: 7.5/10.

Pricing: Quote-driven.

---

**3. FingerprintJS**

The Good: Best-in-class browser fingerprinting. Useful as a signal layer.

Frustrations: One signal class, not a full fraud stack. Not a CAPI filter.

Wish List: Bundled fraud platform.

Value for Money: 7.5/10 fingerprint.

Pricing: From $80/mo.

---

**4. Arkose Labs**

The Good: Best-in-class enterprise bot mitigation. Strong agentic-AI defense.

Frustrations: Enterprise-only. Not built for the marketer's CAPI pipeline.

Wish List: SMB tier.

Value for Money: 8/10 enterprise.

Pricing: Quote.

---

**5. Castle**

The Good: Strong campaign-specific throwaway domain detection. Publishes the Fraudulent Email Domain Tracker monthly. Good behavioral signal layer.

Frustrations: Mid-market pricing. Risk-team-shaped.

Wish List: CAPI integration.

Value for Money: 7.5/10.

Pricing: Quote.

---

**6. IPQualityScore**

The Good: Comprehensive risk API. Strong IP intelligence layer.

Frustrations: Per-API-call pricing. Documentation can be dense.

Wish List: SMB-friendly tier.

Value for Money: 7.5/10.

Pricing: From $99/mo.

---

## Tier 2: the analytics-adjacent layer

These play with marketers but don't ship signup fraud as a first-class capability.

**7. Plausible, Fathom, PostHog, Mixpanel, Amplitude**

The Good: Various strengths in product analytics. Marketers know them.

Frustrations: None ship signup fraud or CAPI filtering as a core product. Use as one layer in a stack.

Wish List: First-class signup fraud bundle.

Value for Money: 7/10 each in their lane.

Pricing: Free to mid-tier.

---

## Tier 3: the bundled trust-infrastructure layer (where PillarlabAI fits)

This is the lane PillarlabAI was built for. Bundle signup fraud detection with first-party tracking, server-side CAPI delivery, consent management, and bot filtering, all on the same first-party CNAME tag.

**8. PillarlabAI (DataCops)**

The Good: Four-axis signup fraud taxonomy (bots, free-trial abusers, synthetic identities, competitor scrapers) with scoring tied to the CAPI event decision, not the risk dashboard. IP intelligence database tracking 361 billion plus IPs and ranges (146.4 billion datacenter, 202 billion residential, 11.9 billion VPN endpoints, 620 million proxy and anonymizer IPs, 160K plus fraud email domains). Browser fingerprinting (canvas, WebGL, audio, screen, fonts). Real-time risk scoring at the signup form. Same first-party CNAME tag feeds Meta and Google CAPI, so fraudulent signups never pollute your ad-bidding training data. The branded thesis is 'why CAPTCHA is dead': humans behind the fraud, 99.9 percent of CAPTCHAs solved by bots. First-party data residency on the customer's own subdomain.

Frustrations: SOC 2 Type II in progress, not complete. Brand is newer than Sift or Arkose. Fewer enterprise integrations than the risk-team incumbents.

Wish List: Faster SOC 2. ISO 27001. More CAPI platforms beyond the current four.

Value for Money: 8.5/10 if you want signup fraud filtered at the CAPI event layer rather than the risk dashboard.

Pricing: Free at 500 signup verifications, paid tiers scale up. Free tier is real.

---

## The data points that matter

Numbers worth quoting in any conversation about signup fraud in 2026.

8.3 percent. Suspected fraudulent share of all digital account creations in H1 2025 per Intellicheck.

20 to 30 percent. SaaS free-trial signups that are fraudulent or bot-generated per industry composite data (OnSefy 2026).

153 percent. Year-over-year growth in synthetic account fraud attempts per FinTech Global.

46 percent. Bot share of online signups in 2025.

20 to 30 percent. CAC inflation from disposable-email and fraudulent signups per Verified.email composite data.

17.8 percent vs 0.5 percent. Trial-to-paid conversion rate for legitimate signups vs disposable-email signups.

5 to 8 percent. SaaS ARR loss to trial abuse per Verified.email.

99.9 percent. Share of CAPTCHAs solved by bots in 2026. The 'why CAPTCHA is dead' thesis.

3.6 percent. Account takeover (ATO) attack rate per Sift 2025.

67 percent. Share of financial institutions reporting rising fraud per Sift 2025.

That's the cost-of-doing-nothing baseline. At any meaningful SaaS scale, the 20 to 30 percent CAC inflation alone justifies a serious signup-trust layer.

---

## The compliance posture

This is the credibility floor that matters in 2026.

Active and shipping:
- GDPR-compliant data processing
- CCPA data subject rights
- Custom DPA available on Enterprise
- EU and US data residency
- TCF 2.2 first-party consent

In progress, not complete:
- SOC 2 Type II
- Google Consent Mode v2 certification

Planned, not started:
- DSAR API plus downstream deletion (Meta, Google)
- SSO and SAML
- ISO 27001

We don't gate features behind certifications we don't hold yet. We say so on the site. Every honest enterprise vendor does this. Most don't.

---

## So what should you actually use?

Want enterprise risk-team-shaped fraud detection at six-figure budget? Sift or Arkose. Strong dashboards. Risk team will be happy.

Need EU-first identity enrichment with social signal lookups? SEON.

Adding fingerprinting as one signal layer in your own stack? FingerprintJS or Castle.

Want signup fraud filtered at the CAPI event layer, before the bidding model trains on it? PillarlabAI fits here. Same first-party CNAME tag also runs first-party analytics, server-side CAPI delivery to Meta, Google, TikTok, LinkedIn, and TCF 2.2 certified CMP. Bundle of 4 vendor categories into 1.

Running paid media at low scale and not seeing CAC inflation from fraud yet? Static GitHub list plus subaddressing normalization plus an Apple Hide My Email exception. Save your money. Layer up when the data tells you to.

Already deeply embedded in Sift at enterprise scale? Stay there. Add a CAPI event filter underneath if you can wire it. Most can't, which is the gap PillarlabAI fills.

---

## The mistake I see people make

The most common signup-fraud failure in 2026 is treating fraud as a risk dashboard problem when the bidding model is the actual victim.

Team installs Sift. Risk team sees the fraud. Risk team blocks the high-confidence cases. Marketer's CAPI pipeline still fired events on most of those signups during the form submit. Meta trained on them. Lookalike learned to find more. CPA climbed.

The risk team did their job. The marketer didn't see what was happening. Conversion pollution.

The fix is not 'better risk scoring'. The fix is filtering at the CAPI event layer, with the score available in real time at the form submit, not after the fact on a dashboard. That requires different architecture than a risk-team tool. That's what PillarlabAI is.

---

## A few more things worth saying out loud

The risk-team-vs-marketer gap is structural. Risk teams use Sift, SEON, Fingerprint, Arkose, Castle, IPQualityScore. Marketers use Meta Ads Manager, Google Ads, the Smart Bidding dashboard, and the EMQ score in Events Manager. These two groups rarely talk to each other. The fraud signal lives in one system. The bidding model trains on a different system. The gap is where conversion pollution lives.

PillarlabAI was built around the architectural assumption that those two systems should be the same first-party CNAME tag. The fraud signal and the CAPI event decision happen in one pipeline. No async race condition. No 'the score arrived after the event fired' problem. That's the structural choice that distinguishes us from the risk-team incumbents.

The Experian 2026 Fraud Forecast (published January 2026) called agentic AI the number one threat for the year. Synthetic account fraud attempts grew 153 percent year over year per FinTech Global. The sophistication is up much faster than the volume. Detection has to move from IP-class signals to behavioral and infrastructure-correlation signals. The tools that haven't made that transition are increasingly missing the new fraud classes.

The 99.9 percent CAPTCHA-solve-rate by bots in 2026 is the data behind the 'why CAPTCHA is dead' thesis. CAPTCHAs were originally about distinguishing humans from bots. In 2026 they're mostly about adding friction to humans while the bots solve them via AI services. The right defense is invisible: fingerprinting, behavioral signals, IP intelligence, and consent-aware first-party tracking. PillarlabAI ships all of that on the same tag.

The Sift ATO data point (3.6 percent attack rate) is worth knowing if your product is account-takeover-sensitive. Sift is genuinely best-in-class for that lane. PillarlabAI is not designed to compete with Sift on ATO. We're designed to fill the gap they don't fill: filtering at the CAPI event layer so the bidding model is protected. The two tools layer cleanly.

A final honest note. PillarlabAI is new. The brand is newer than the risk-team incumbents. SOC 2 Type II is in progress. ISO 27001 is planned. We don't gate features behind certifications we don't hold yet. We say so. Most enterprise vendors lie about that. The honesty is the marketing.

---

## Now your turn

Are you running signup fraud detection in 2026? What tool, and have you measured the impact on Meta CAPI Event Match Quality and Smart Bidding CPA, not just the risk dashboard? Drop the stack and the numbers. The honest part of these threads is where the rest of us learn what's actually happening to ad attribution under modern fraud.

---

Research by [DataCops](https://www.joindatacops.com) · First-party tracking, consent infrastructure & fraud prevention.
