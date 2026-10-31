# Cynbit Backend Technical Audit

**Target:** [cynbit.in](https://www.cynbit.in/)  
**Assessment date:** 31 October 2026  
**Perspective:** Anonymous, public-facing backend and data-handling review

## Executive summary

The publicly visible site is a Next.js application delivered through Vercel. Its pages appear primarily pre-rendered and content-oriented, with hashed static assets cached immutably. A public internship application is rendered in the browser and collects personal and educational information.

The most actionable improvement is to reduce and better explain the personal data required by that form. The privacy notice describes limited, typical inquiry data, but the application form requires additional data such as date of birth, gender, WhatsApp number, and permanent address. The public pages do not expose enough information to verify how submissions are validated, stored, protected, retained, or deleted.

No exploitable vulnerability or backend compromise was established. This review did not submit forms, probe guessed endpoints, test authentication, or generate load.

## Scope and method

Reviewed ordinary public GET responses and rendered pages at the homepage, contact page, internship application page, privacy policy, cookie policy, terms, robots file, and sitemap. Inspected public response headers and the browser-rendered application form. No form data was entered or submitted.

This is not a source-code review, penetration test, legal assessment, or verification of private/admin systems. API behavior, persistence, authentication, rate limiting, and error handling are therefore **not assessable** from the observations below.

## Observable architecture and behavior

| Area | Public observation |
|---|---|
| Web framework / hosting | HTML references Next.js App Router chunks; response headers identify Vercel and Next.js prerendering. This identifies the public delivery layer, not the private service architecture. |
| Rendering and caching | Homepage response was 200, 100,498 bytes, `X-Vercel-Cache: HIT`, with `Cache-Control: public, max-age=0, must-revalidate`. A hashed JavaScript asset was served with `public,max-age=31536000,immutable`. |
| Public routes | The [sitemap](https://cynbit.in/sitemap.xml) lists marketing, career, blog, case-study, and policy routes. [`robots.txt`](https://www.cynbit.in/robots.txt) disallows crawling `/api/`, `/admin/`, and `/portal/`; robots rules are crawler guidance, not access control. No such route was tested. |
| Contact | The [contact page](https://www.cynbit.in/contact) presents email and telephone contact details. No contact submission was made. |
| Application | The [backend-intern page](https://www.cynbit.in/careers/backend-developer-intern) renders a multi-section application form. Required fields include name, email, phone and WhatsApp numbers, date of birth, gender, education and score details, current city, permanent address, profile links, and an application statement. |
| Data handling disclosures | The [privacy page](https://www.cynbit.in/privacy) describes typical inquiry data as name, email, and professional context; names Resend and Prisma; and provides a copy/deletion contact. The [cookie policy](https://www.cynbit.in/cookies) mentions Google Analytics. No retention period or detailed application-data lifecycle was found in the reviewed policy text. |
| API / payloads | No API contract or submission payload was observed. The form’s visible fields indicate the categories a submission may contain, but the request format, destination, server-side validation, and storage behavior remain unknown. |

## Findings

### 1. Medium — Application data minimization and notice gap

**Evidence:** The [public internship form](https://www.cynbit.in/careers/backend-developer-intern) requires date of birth, gender, two phone numbers, and permanent residential address, in addition to education/score information, contact details, profile links, and an application statement. The [privacy page](https://www.cynbit.in/privacy)'s data description focuses on name, email, and professional context and discusses BitElevate applications, but does not explain the internship form's additional required categories or state a retention period.

**Risk:** Collecting more personal information than needed at the initial application stage increases the impact of account misuse, accidental disclosure, and unnecessary retention. Applicants may not understand why each required field is collected or how long it is kept.

**Recommendation:** Make fields optional unless they are demonstrably needed for an early-stage hiring decision. In particular, reassess date of birth, gender, WhatsApp number, and full address; explain the purpose beside any field that remains required. Update the privacy notice to cover recruitment applications, processors, access, retention/deletion, and applicant rights. Restrict application records to authorized recruiting staff, redact personal data from logs, and define a deletion/anonymization schedule.

**Confidence:** High for the visible form and notice text; no conclusion about actual storage or access controls.

### 2. Medium — Analytics disclosure and consent behavior need verification

**Evidence:** The public page source loads Google Analytics. The [cookie policy](https://www.cynbit.in/cookies) identifies Google Analytics as an analytics cookie, while the [privacy policy](https://www.cynbit.in/privacy)'s third-party section names Resend and Prisma but does not mention Google Analytics. No consent banner or preference control was visible in the reviewed anonymous page view; the cookie policy directs visitors to browser settings.

**Risk:** The mismatch makes it difficult for visitors to understand what analytics data is collected and how to opt out. Whether consent is legally required depends on audience and jurisdiction and was not assessed here.

**Recommendation:** Reconcile the privacy and cookie notices, identify the analytics provider and relevant data/use/retention details, and verify analytics behavior before consent. Where applicable, add a clear consent/preference control, honor the selected choice and Global Privacy Control where relevant, and avoid sending analytics events until the applicable consent state permits them.

**Confidence:** High for the policy wording and analytics script; moderate for consent behavior because no privacy settings or regional variants were tested.

### 3. Low — Recommended browser security headers are not visible

**Evidence:** Responses for the homepage and internship application page included HSTS. The inspected responses did not include a Content Security Policy, `X-Content-Type-Options`, `Referrer-Policy`, or a framing restriction such as CSP `frame-ancestors`.

**Risk:** This is a defense-in-depth gap, not evidence of an exploitable vulnerability. These controls can reduce the impact of certain injection, content-sniffing, referrer-leakage, and clickjacking scenarios.

**Recommendation:** Add a tested CSP (using a nonce or hashes and explicit allowances for required third-party scripts), `X-Content-Type-Options: nosniff`, an appropriate `Referrer-Policy`, and `frame-ancestors 'none'` or a narrowly defined framing policy. Keep HSTS, which was present. Validate policy changes against the live application and required analytics/font resources before enforcement.

**Confidence:** High for the sampled responses; headers may differ on other routes or response types.

### 4. Low — Public HTML is revalidated by browsers

**Evidence:** The homepage HTML is about 100 KB and has `Cache-Control: public, max-age=0, must-revalidate`, despite a Vercel edge cache hit. The inspected hashed JavaScript asset uses a one-year immutable cache policy, which is appropriate for content-hashed assets.

**Risk:** Browsers may revalidate the HTML on repeat visits. No latency or Core Web Vitals measurement was performed, and the edge cache hit suggests this is an optimization opportunity rather than a demonstrated performance defect.

**Recommendation:** Measure real-user and repeat-visit performance first. If appropriate for these public pages, configure a short browser TTL with stale-while-revalidate while preserving safe freshness behavior. Continue using long-lived immutable caching for hashed assets. Keep personalized or applicant-specific responses private and non-cacheable.

**Confidence:** High for the sampled headers and content length; low that this currently causes a user-visible performance problem.

## Recommended backend design for the application flow

Treat an application as an unauthenticated, privacy-sensitive submission—not as trusted browser data. The following is an illustrative TypeScript/Zod pattern, not a description of Cynbit's implementation. It intentionally excludes fields that should be collected only when justified.

```ts
import { z } from "zod";

const ApplicationInput = z.object({
  fullName: z.string().trim().min(1).max(120),
  email: z.string().trim().email().max(254),
  phone: z.string().trim().min(7).max(32),
  qualification: z.string().trim().min(1).max(120),
  skills: z.string().trim().min(1).max(1_000),
  resumeUrl: z.string().url().max(2_048).optional(),
  linkedInUrl: z.string().url().max(2_048).optional(),
  statement: z.string().trim().min(1).max(5_000),
}).strict();

type ApplicationInput = z.infer<typeof ApplicationInput>;

function validateApplication(input: unknown) {
  const result = ApplicationInput.safeParse(input);
  if (!result.success) {
    return {
      ok: false as const,
      response: Response.json(
        { error: { code: "VALIDATION_FAILED", message: "Check the highlighted fields." } },
        { status: 422, headers: { "Cache-Control": "no-store" } },
      ),
    };
  }

  return { ok: true as const, data: result.data };
}
```

In the route handler, enforce a request-body size limit, apply per-IP and per-address rate limits and proportionate bot controls, then call this validator on the server. Persist the application and an email-notification outbox event in one database transaction so a mail-provider outage does not lose a valid application. Make retries idempotent and return a non-cacheable receipt identifier rather than echoing submitted data.

Suggested response contract:

| Status | Meaning |
|---|---|
| `201` | Application accepted; return a receipt ID only. |
| `400` / `422` | Malformed JSON or field validation failure; return stable error codes without submitted values. |
| `429` | Rate limit reached; include `Retry-After`. |
| `500` | Generic error plus correlation ID; keep stack traces and personal data out of the response. |

Use structured server logs with correlation IDs and redaction, least-privilege access to applicant records, encryption in transit and at rest, and a documented deletion schedule. If profile URLs are ever fetched server-side, do not fetch arbitrary user-supplied URLs; prevent SSRF with strict URL/host validation and network egress controls. If future uploads are supported, use size/type allowlists, malware scanning, private object storage, and short-lived signed access.

## Recommended next verification (with system-owner authorization)

1. Review the application route and API implementation to confirm server-side schema validation, request-size limits, anti-abuse controls, idempotency, and explicit error mapping.
2. Verify persistence, access roles, email-provider payloads, encryption, logging redaction, retention, and deletion end-to-end using test data in a non-production environment.
3. Confirm the application endpoint's CORS/CSRF posture and that public submission endpoints cannot expose applicant records.
4. Test consent choices and analytics network behavior in supported regions, then document the observed data flow.
5. Measure repeat-visit caching and Core Web Vitals before changing HTML cache policy.
