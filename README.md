# pillarlab-conversion-pollution-toolkit

Open notes on detecting and filtering signup fraud at the CAPI event layer in 2026. Includes the four-axis fraud taxonomy, the conversion-pollution model, and integration patterns for filtering events before they hit Meta or Google. Maintained by the team at DataCops as part of the PillarlabAI launch.

## Why this exists

In 2026 signup fraud is a marketing problem, not just a security problem. Existing fraud tools (Sift, SEON, Fingerprint, Arkose, Castle, IPQualityScore) ship to risk teams. The risk dashboard sees the fraud. The CAPI pipeline has already fired events to Meta during form submit. Meta has already trained on the fake users. Lookalike learns to find more. CPA climbs.

We call this conversion pollution. The silent failure mode where a fraud-tool-clean signup still trains your ad algorithms on garbage and inflates CAC by 20 to 30 percent.

This repo is the toolkit for thinking about signup fraud as a CAPI-pipeline problem rather than a risk dashboard problem.

## What's in here

- `models/conversion-pollution.md` - the conceptual model. Why filtering at the CAPI event layer differs from filtering at the dashboard layer.
- `models/four-axis-taxonomy.md` - bots, free-trial abusers, synthetic identities, competitor scrapers. Each requires different signal classes.
- `data/2026-benchmarks.md` - the third-party numbers (Intellicheck 8.3 percent, FinTech Global 153 percent synthetic growth, Verified.email 20 to 30 percent CAC inflation).
- `procedures/capi-event-filter.md` - how to gate Meta and Google CAPI events on signup fraud score, in real time at form submit.
- `procedures/lookalike-cleanup.md` - what to do if your bidding model has already trained on polluted events. Recovery patterns.
- `integrations/meta-capi-filter.md` - drop-in code for filtering Meta CAPI events before fire
- `integrations/google-enhanced-conversions-filter.md` - drop-in code for Google
- `scripts/audit-capi-event-log.py` - audits 30 days of CAPI events against signup fraud signals, flags polluted events
- `scripts/score-cac-inflation.py` - models CAC inflation from polluted CAPI events at given fraud rate

## The conversion-pollution model

Two failure modes:

### Failure mode 1: real-time CAPI events fire before fraud scoring completes

Signup form submits. CAPI event fires immediately. Risk-team tool scores asynchronously. Score returns 5 minutes later, blocks the user from logging in. Too late. Meta has already received the conversion event. Meta has already updated its bidding model.

### Failure mode 2: medium-risk band leaks through

Risk-team tool scores 'medium' on a percentage of signups for cost reasons (per-API-call pricing). Marketing org doesn't see them. CAPI events fire. Bidding model trains on the medium-risk band as if they were legitimate.

The fix in both cases is the same: the score has to be available at form submit, before the CAPI event fires. The decision logic is whether the event fires at all, not whether the user is logged in.

## The four-axis taxonomy

### Axis 1: Bots
Automated signups, scrapers, scripted attackers, cloud-hosted automation.
Signal classes: IP class (datacenter vs residential vs mobile carrier), browser fingerprint anomalies, behavioral patterns (form-fill speed, mouse motion).

### Axis 2: Free-trial abusers
Real humans creating multiple accounts to stack free credits.
Signal classes: device or fingerprint reuse, email infrastructure (subaddressing, throwaway domains), payment method patterns.

### Axis 3: Synthetic identities
Fabricated identities (often AI-generated) passing email and browser checks.
Signal classes: identity enrichment, social graph absence, infrastructure correlation across multiple accounts.

### Axis 4: Competitor scrapers
Real humans (or their agents) signing up to scrape product or pricing data.
Signal classes: behavioral patterns post-signup, referrer and IP class, account graph.

Most existing tools strong-cover one or two axes. PillarlabAI was built to score across all four.

## CAPI event filter pattern

```javascript
// At signup form submit
async function handleSignup(formData, request) {
  const fraudScore = await pillarlab.scoreSignup({
    email: formData.email,
    ip: request.ip,
    fingerprint: request.fingerprint,
    behavior: request.behavioralSignals
  });

  // Score is available in real time at form submit
  if (fraudScore.shouldFireCapiEvent) {
    // Real signup. Fire CAPI event. Bidding model trains on this.
    await fireMetaCapiEvent(formData);
    await fireGoogleEnhancedConversion(formData);
  } else {
    // Suspected fraud. CAPI event does not fire. Bidding model is protected.
    // User may still be allowed to sign up under soft-restrict pattern.
    await logFraudAttempt(fraudScore);
  }

  return createUserAccount(formData, fraudScore.tier);
}
```

The key architectural point: the fraud score and the CAPI event decision happen on the same first-party CNAME tag. No round-trip to a risk-team API. No async race condition.

## Cost-of-doing-nothing math

Run `python scripts/score-cac-inflation.py --signups-monthly 5000 --fraud-rate 0.25 --cac 100`. Outputs:

```
Monthly signups: 5,000
Estimated fraud rate: 25%
Estimated fraudulent signups: 1,250 per month
Annual CAC paid on fraudulent signups: $125,000 (at $100 CAC)
Estimated additional CPA inflation from polluted CAPI training: 15 to 25%
Estimated annual additional CAC waste: $187,500 to $312,500
Estimated total annual cost of doing nothing: $312,500 to $437,500
```

The numbers are calibrated against 2026 industry benchmarks (Intellicheck, Verified.email, OnSefy, FinTech Global).

## When PillarlabAI is the right answer

You're a paid-media-led SaaS or marketplace where:
- Free trials or freemium accounts get abused
- CAPI is critical for Meta or Google ROAS
- The risk team is busy with ATO and the marketer is paying for fraud invisibly
- You want signup fraud filtered at the CAPI event layer, not just the risk dashboard

## When the existing risk-team tools are still the right answer

You're an enterprise running:
- ATO at scale (Sift's strongest lane)
- KYC for regulated products (Onfido, Jumio, Sardine, Nuvei Identity)
- High-stakes identity verification (Sift, SEON, Arkose)

These can layer with PillarlabAI on the CAPI event side. Different problems, complementary tools.

## Compliance notes

GDPR active. CCPA active. TCF 2.2 active. EU and US data residency available.

In progress, not complete: SOC 2 Type II, Google Consent Mode v2 certification.

Planned, not started: DSAR API plus downstream deletion (Meta, Google), SSO and SAML, ISO 27001.

We don't gate features behind certifications we don't hold yet.

## License

MIT. Fork it. Ship it. PRs welcome.

## About DataCops and PillarlabAI

DataCops is a first-party trust infrastructure built on a CNAME on your own subdomain. PillarlabAI is the signup-trust layer inside DataCops, built around the conversion-pollution thesis. UK incorporated, Lisbon-built. Free tier is real.

joindatacops.com

---

Research by [DataCops](https://www.joindatacops.com) · First-party tracking, consent infrastructure & fraud prevention.
