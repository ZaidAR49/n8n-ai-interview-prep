# 🎯 AI Interview Prep — n8n Workflow

> Submit a form with your LinkedIn, your interviewer's LinkedIn, and your CV — get a personalized, HTML-formatted interview preparation report in your inbox within minutes.

---

## What It Does

This n8n automation takes a few inputs from a candidate and produces a tailored interview prep report, emailed directly to them. It scrapes LinkedIn profiles via Apify, extracts CV text from an uploaded PDF, feeds everything to a GPT-4o-mini agent, and emails the resulting HTML report via Gmail.

---

## Workflow Overview

```
Form Submission
      │
      ▼
Validate (need ≥1 of: Interviewer or Company LinkedIn)
      │
   ┌──┴──────────────────────────────┐
   │ Invalid → Send Error Email      │
   └─────────────────────────────────┘
      │ Valid
      ▼
Four parallel branches:
  ├── Always:   Scrape User LinkedIn
  ├── If given: Scrape Interviewer LinkedIn
  ├── If given: Scrape Company LinkedIn
  └── If given: Extract CV text from PDF
      │
      ▼
Consolidate All Data (Code node)
      │
      ▼
AI Interview Coach (GPT-4o-mini LangChain Agent)
      │
      ▼
Send Interview Report (Gmail)
```

---

## Nodes

| Node | Type | Purpose |
|------|------|---------|
| Form Trigger | n8n Form | Collects email, LinkedIn URLs, and optional CV upload |
| Validate: Need At Least One | IF | Rejects submissions missing both interviewer and company URLs |
| Send Error Email | Gmail | Sends a styled error email if validation fails |
| Run: Scrape User LinkedIn | Apify | Scrapes the candidate's LinkedIn profile |
| Has Interviewer LinkedIn? | IF | Gates scraping on whether the URL was provided |
| Run: Scrape Interviewer LinkedIn | Apify | Scrapes the interviewer's LinkedIn profile |
| Has Company LinkedIn? | IF | Gates scraping on whether the URL was provided |
| Run: Scrape Company LinkedIn | Apify | Scrapes the company's LinkedIn page |
| Extract from File | Extract From File | Extracts plain text from the uploaded PDF CV |
| Consolidate All Data | Code (JS) | Merges all parallel branch outputs into one clean object |
| AI Interview Coach | LangChain Agent | Generates the full HTML prep report |
| OpenAI GPT-4o (Coach) | OpenAI Chat Model | Powers the LangChain agent (gpt-4o-mini) |
| Send Interview Report | Gmail | Emails the final HTML report to the candidate |

---

## Form Fields

| Field | Required | Description |
|-------|----------|-------------|
| Your Email | ✅ | Recipient address for the final report |
| User LinkedIn | ✅ | Candidate's LinkedIn profile URL |
| Interviewer LinkedIn | ⬜ | LinkedIn URL of the interviewer (at least one of these two is required) |
| Company LinkedIn | ⬜ | LinkedIn URL of the company (at least one of these two is required) |
| CV Upload | ⬜ | PDF résumé/CV (optional but improves report quality) |

---

## Report Sections

The AI agent produces a single self-contained HTML document containing:

1. **Candidate name and meta line** — interviewer, company, generation date
2. **Company Snapshot** — what they do, culture signals *(omitted if no company data)*
3. **The Opportunity** — likely role, responsibilities, problems to solve
4. **Fit Score** — 0–100% match with explanation *(omitted if no company data)*
5. **What You Have in Common** — verifiable shared traits between candidate, interviewer, and company
6. **Questions to Ask the Interviewer** — grounded in specific profile details *(omitted if no interviewer data)*
7. **Questions to Ask About the Company** — grounded in company profile *(omitted if no company data)*
8. **Questions They Will Likely Ask You** — with coaching guidance per question
9. **Disclaimer** — auto-generated report notice

---

## Prerequisites & Credentials

| Service | Used For | Credential Type |
|---------|----------|-----------------|
| Gmail | Sending the error email and final report | Gmail OAuth2 |
| Apify | Scraping LinkedIn profiles | Apify OAuth2 |
| OpenAI | GPT-4o-mini for the AI agent | OpenAI API key |

### Apify Actor

This workflow uses the Apify actor with ID `LpVuK3Zozwuipa5bp` in `"Profile details no email ($4 per 1k)"` mode. Ensure your Apify account has sufficient credits.

---

## Setup Instructions

1. **Import** the workflow JSON into your n8n instance.
2. **Connect credentials** for Gmail OAuth2, Apify OAuth2, and OpenAI.
3. **Activate** the workflow — the Form Trigger will generate a public URL automatically.
4. **Test** by submitting the form with your own LinkedIn and at least one of: interviewer or company LinkedIn.

---

## Conditional Logic & Edge Cases

| Scenario | Behavior |
|----------|----------|
| Neither interviewer nor company LinkedIn provided | Error email sent; workflow stops |
| Only company LinkedIn provided | Interviewer-specific sections omitted |
| Only interviewer LinkedIn provided | Company, fit score, and company Q&A sections omitted |
| No CV uploaded | Report generated from LinkedIn data only; absence not mentioned in report |
| LinkedIn scrape returns no data | Section uses graceful fallback; no placeholder text shown |
| Both optional fields missing | Report contains candidate profile, expected questions, and common ground only |

---

## Known Limitations & Risks

- **LinkedIn ToS** — Scraping LinkedIn via Apify may violate LinkedIn's Terms of Service. Use at your own risk and review Apify's compliance guidance before deploying in production.
- **No retry logic** — If the AI agent call or final email send fails, the workflow does not retry. Consider adding an error handler node for production use.
- **Apify costs** — Each run costs approximately $0.004 per profile scraped (up to 3 profiles per submission).
- **Gmail HTML rendering** — The agent is prompted to produce Gmail-safe HTML (no flexbox, no CSS variables, no external fonts). If the LLM deviates, the report may render incorrectly.

---

## Prompt Engineering Notes

The AI Interview Coach agent operates under a strict system prompt that enforces:

- **HTML-only output** — first character `<`, last character `>`; no markdown fences or preamble
- **No fabrication** — every claim must trace to the input data
- **Fit score calibration** — weak matches must score 30–50%; inflating to 70%+ without evidence is treated as a failure
- **Exact-match common ground** — shared institutions only count if the exact school name matches; same country or same field is not sufficient
- **Question specificity** — questions for the interviewer must reference a specific, verifiable detail from their profile; generic questions are a failure condition

---

## License

MIT — free to use, modify, and distribute.
