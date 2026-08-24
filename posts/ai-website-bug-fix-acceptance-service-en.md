
## The conclusion: sell acceptance evidence, not “I use AI”

“Fix our old website” is too broad for a first freelance offer. A safer, easier-to-explain unit is **one authorized problem, one agreed acceptance path, a change note, and a short recording that another person can review**.

AI is an accelerator, not the promise. It can structure a report, explain a console error, propose causes, and draft a test. You still prove the result with the original reproduction, the same steps after the patch, repeatable checks, and human review. The client is buying a reproducible, verifiable, reversible result.

> **Quotable summary:** The smallest sellable legacy-website bug-fix service is one problem, one acceptance path, and one evidence bundle. AI speeds up investigation; the human owns authorization, patch judgment, risk control, and the final acceptance decision.

## Who this service is for—and who it is not for

You should be able to read browser console output, run basic commands, and make small HTML/CSS/JavaScript or template changes. You do not need to promise a full rewrite, but you must know when to stop and escalate.

| In scope for a bounded job | Out of scope |
| --- | --- |
| A form button produces no result message | Bypassing login, permissions, or CAPTCHA |
| A mobile layout overflows | Account takeover, payments, or money movement |
| A page asset returns a known 404 | Direct production-database edits |
| One page has a clear API error state | Unauthorized security testing |
| A small template change with a rollback point | Medical, legal, or financial decision logic |

Before accepting, confirm ownership or written maintenance authorization, a test environment or reproducible deployment, redacted reproduction information, and a rollback path. Without those, offer only a report or consultation; do not probe or edit production.

## Turn the offer into a one-page scope

Replace “let me take a look” with a short scope note:

1. **Problem:** one sentence, such as “The product form shows a spinner but no success state.”
2. **Scope:** page, browser or viewport, test account, repository or permitted folder.
3. **Acceptance path:** every action from opening the page to the expected result.
4. **Deliverables:** reproduction note, changed-file list, test result, acceptance recording, and rollback note.
5. **Exclusions:** redesign, dependency upgrades, server migration, sensitive-data handling, and unrelated bugs.
6. **Change rule:** a new page, endpoint, or business rule pauses the job for a new scope decision.

Send a small intake table before quoting:

| Problem | Client provides | You confirm | Acceptance |
| --- | --- | --- | --- |
| Form submission fails | URL, steps, redacted screenshot | test account and authorization | success state appears and refresh behaves as agreed |
| Mobile overflow | device or viewport and page | whether a design reference exists | no horizontal scroll in the agreed viewport |
| Asset 404 | failing URL and deploy method | repository and rollback point | asset returns the expected state without new console errors |

## Seven steps from report to recording

### 1. Authorization and redaction

Ask for the report first, not a production password. Replace names, phone numbers, order IDs, cookies, tokens, and real user content with placeholders. If login is required, use a short-lived test account and agree on its expiration.

Write down which environment, page, time window, and actions are authorized. No authorization means no probing “just to verify.”

### 2. Minimal reproduction before code changes

Record environment, browser, viewport, role, steps, actual result, expected result, and timestamp. Use Console and Network panels, but keep only the minimum redacted request information required to explain the issue.

```text
Environment: staging.example.test (example domain)
Browser: Chromium, desktop viewport
Steps: open product page -> enter redacted data -> submit
Actual: button loads, then resets without a result message
Expected: success message appears and form resets
Evidence: timestamp, screenshot, redacted console summary
```

This is a fictional format example, not a client case. Reproduction prevents a slow network or missing permission from being misdiagnosed as a code defect.

### 3. Ask AI for hypotheses, not a verdict

Give the model a redacted reproduction card, a minimal code fragment, and an error summary. Ask for ranked hypotheses, a verification action for each, a minimal patch, and regression risks. Never paste an entire repository, environment file, or client dataset into an unapproved service.

```text
You are assisting with a Web bug investigation.
Known symptom: <reproduction steps and actual result>
Known error: <redacted error summary>
Relevant code: <minimal fragment>
Return: 1) ranked hypotheses; 2) verification actions; 3) smallest patch; 4) regression risks.
Do not invent business rules or request passwords, cookies, or tokens.
```

Label the model response as a candidate hypothesis. A model saying “it may be CORS” is not evidence; a reproducible request and response are.

### 4. Patch the smallest path and keep a rollback point

Create a branch or patch and record the pre-change commit. Fix the trigger path only. Avoid opportunistic dependency upgrades, CSS rewrites, or unrelated pages. If configuration must change, preserve the old value and write the rollback command.

Use an understandable commit message such as `fix: show form result after submit`. Explain what changed, why, and what did not change.

### 5. Use Playwright and Lighthouse as repeatable checks

The [official Playwright introduction](https://playwright.dev/docs/intro) describes Playwright Test as an end-to-end framework supporting Chromium, WebKit, and Firefox with a test runner, assertions, isolation, parallelism, and HTML reports. Turn the client's acceptance path into the smallest useful test:

```js
import { test, expect } from '@playwright/test';

test('form shows a result after submit', async ({ page }) => {
  await page.goto('https://staging.example.test/contact');
  await page.getByLabel('Name').fill('Test User');
  await page.getByLabel('Email').fill('test@example.test');
  await page.getByRole('button', { name: 'Submit' }).click();
  await expect(page.getByRole('status')).toContainText('Success');
});
```

Save an HTML report and, when useful, a trace for failed or reviewed runs. The [official Trace Viewer documentation](https://playwright.dev/docs/trace-viewer) explains local and browser viewing; its browser-hosted version says the trace is loaded in the browser and not transmitted externally. Still inspect the trace for client data before delivery.

Use Lighthouse as a supplementary performance and quality check, not as the sole proof that the bug is fixed. Acceptance returns to the agreed path: the button works, the result appears, and the state is correct after refresh.

### 6. Record a reviewable acceptance video

The video is an evidence index, not an advertisement. A few minutes is a delivery estimate, not a platform requirement:

1. Show date, environment, and URL while hiding accounts and personal data.
2. Reproduce the symptom with redacted data.
3. Repeat the exact path on the patched version.
4. Show the result, assertion, or report without secrets.
5. State covered items, gaps, and rollback steps.

Deliver a checklist alongside it:

- [ ] authorization and test account confirmed; no password, cookie, or token in the video;
- [ ] pre-fix and post-fix actions use the same path;
- [ ] key assertion, console, or network evidence is reviewable;
- [ ] uncovered browser, device, and edge cases are listed;
- [ ] commit, deployment time, and rollback steps are recorded.

### 7. Handoff, release, and rollback

Send the report, patch or commit, test report, video, and rollback note for confirmation before the agreed release window. Re-run the path after release. If the result differs, roll back first and record the environmental difference instead of experimenting repeatedly in production.

```text
bug-acceptance/
├── 01-reproduction.md
├── 02-change-summary.md
├── 03-test-report.html
├── 04-acceptance-video.mp4
└── 05-rollback.md
```

## Fictional demo: a form button has no success state

This is a fictional SOP demo, not a client story or a real website.

The scope covers a staging contact page, a Chromium desktop viewport, and a success message. The redacted response contains `ok: true`, but the success callback updates a state field that the template does not read. The minimal patch maps the state field, keeps the previous commit for rollback, and does not touch the email service.

Playwright asserts that the status role contains Success. A human refreshes the page and confirms the agreed state. The recording shows the same path and the report. Safari, email delivery, and real user data remain uncovered and are explicitly listed as such.

## Tools, costs, time, and the revenue model

| Tool | Purpose | How to account for cost |
| --- | --- | --- |
| General-purpose LLM | report structure, hypotheses, test drafts | use your actual plan or API bill; do not infer profit from marketing pages |
| Browser DevTools | Console, Network, device and viewport reproduction | record the tools actually available on your machine |
| Playwright | critical-path regression, HTML reports, traces | separate open-source tooling from browser-download cost |
| Lighthouse | supplementary performance check | keep the actual environment and report; do not promise a score |
| Git | branches, diffs, rollback points | record actual hosting or storage cost |
| Screen recorder | acceptance evidence | record actual subscription or storage cost |

Without historical timing, these are estimates only: intake 15–30 minutes, minimal reproduction 20–60 minutes, patch and regression 30–120 minutes, and recording/handoff 15–30 minutes. Split the job when it exceeds one acceptance path.

Use the formula **revenue = acceptance price × completed jobs − direct costs**. For example, a hypothetical self-set price of ¥300, two jobs, and ¥40 of direct tool/storage cost gives an estimated arithmetic remainder of ¥560. This is an estimate, not a market quote, income guarantee, or client case; real review also includes communication, rework, tax, and platform costs.

## Finding first clients: sell the check before the rewrite

The easiest entry point is paid reproduction plus an acceptance checklist, not “I can rebuild your site with AI.” Create a controlled demo bug on your own site, publish the reproduction card and redacted recording, and explain what information you need before quoting. Start with people who can authorize the environment; do not ask for production passwords.

The [AI automation service guide](/en/posts/ai-automation-service-en/) explains why cross-system workflows deserve a larger, auditable project. The [Cursor Chrome extension guide](/en/posts/cursor-chrome-extension-en/) provides development-environment background. Neither is a success claim for this small service.

## Risk, privacy, and compliance boundaries

The [OWASP Top Ten page](https://owasp.org/www-project-top-ten/) currently identifies the 2025 edition as the latest released version. Its [Path Traversal page](https://owasp.org/www-community/attacks/Path_Traversal) explains how `../` variants can be used to reach files outside a web root. You do not need to turn every maintenance job into a full security audit, but suspected traversal, permission bypass, or sensitive-data exposure is a stop-and-escalate signal.

Check the address bar, query strings, Network requests, screenshots, traces, and local recording folder before delivery. Keep only necessary files and delete temporary copies according to the client agreement. Do not upload source code, environment variables, real user data, or credentials to an unapproved third party.

One passing E2E path proves only that path in that environment. It is not a security guarantee. Avoid “always safe,” “fixed once and for all,” or conversion guarantees in a contract or public post.

## Verification metrics: review evidence, not imagined income

Track only reviewable facts per job:

- another developer can independently replay the reproduction;
- the same path has the agreed result after the patch;
- the automated check passes in the agreed browser and viewport;
- the recording, report, and commit point to one another;
- covered and uncovered items are explicitly confirmed;
- rework is classified as original defect, scope change, or environment difference.

After a week, these records tell you which scenario is worth standardizing. An unsupported “saved 80% of labor” claim is not a metric.

## Frequently asked questions

### Can I take the job without the client's source code?

When there is no written authorization, test environment, or rollback path, do not modify or probe the system. You may offer a reproduction report, but exclude the patch and production release from scope.

### Can AI decide that the bug is fixed for me?

No. AI can draft investigation hypotheses and tests; the conclusion must come from human reproduction, automated results, a critical-path check, and client confirmation.

### What belongs in the acceptance recording?

Show a redacted environment, the pre-fix symptom, the same path after the fix, key assertions or reports, and known gaps. Never record passwords, cookies, tokens, or client data.

## Sources

All sources below are official pages. None exposed an explicit publication date; each was accessed on 2026-08-24.

1. **Microsoft Playwright, “Installation / Introduction”** — publication date not stated; HTTP `Last-Modified` was 2026-08-21; accessed 2026-08-24. Used to verify browser support, runner, assertions, isolation, parallelism, and HTML reports. <https://playwright.dev/docs/intro>
2. **Microsoft Playwright, “Trace viewer”** — publication date not stated; HTTP `Last-Modified` was 2026-08-21; accessed 2026-08-24. Used to verify trace viewing and the browser-hosted local loading statement. <https://playwright.dev/docs/trace-viewer>
3. **OWASP, “OWASP Top Ten Web Application Security Risks”** — publication date not stated; HTTP `Last-Modified` was 2025-12-24; accessed 2026-08-24. The page identifies OWASP Top 10 2025 as the current release. <https://owasp.org/www-project-top-ten/>
4. **OWASP, “Path Traversal”** — publication date not stated; HTTP `Last-Modified` was 2026-08-21; accessed 2026-08-24. Used to verify the `../` path-variant risk explanation. <https://owasp.org/www-community/attacks/Path_Traversal>

Any time, price, cost, and revenue figures in this article are formulas, delivery-model estimates, or arithmetic demonstrations—not market data from the sources above.
