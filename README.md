<div align="center">

# ResumeCraft AI

**AI-assisted resume builder with a live PDF preview, four ATS-friendly templates, bring-your-own-key AI and full Arabic (RTL) support.**

[![Status](https://img.shields.io/badge/status-live-2ea44f?style=flat-square)](https://resume.everbuild.cloud)
[![Platform](https://img.shields.io/badge/platform-web-0a66c2?style=flat-square)](https://resume.everbuild.cloud)
[![Next.js](https://img.shields.io/badge/Next.js-16-000000?style=flat-square&logo=nextdotjs&logoColor=white)](https://nextjs.org)
[![React](https://img.shields.io/badge/React-19-149eca?style=flat-square&logo=react&logoColor=white)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/TypeScript-5-3178c6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-Prisma_7-4169e1?style=flat-square&logo=postgresql&logoColor=white)](https://www.prisma.io)
[![Vercel AI SDK](https://img.shields.io/badge/Vercel_AI_SDK-v6-111111?style=flat-square&logo=vercel&logoColor=white)](https://ai-sdk.dev)

[**Live demo**](https://resume.everbuild.cloud) · [**Portfolio**](https://portfolio.everbuild.cloud) · [**LinkedIn**](https://linkedin.com/in/dafallah-ibrahim)

</div>

> [!NOTE]
> This is a showcase repository. The source code is private (personal project). It documents what the product does, how it's built and my role. Happy to walk through the code in an interview.

<p align="center">
  <img src="docs/screenshots/editor-live-preview.webp" width="100%" alt="ResumeCraft AI editor: step-by-step form on the left (Experience step with two jobs and bullet points) and a live A4 PDF preview of the resume on the right">
</p>

<sub>All screenshots are real captures of the live app, using a made-up demo persona ("Alex Morgan").</sub>

## Screenshots

<p align="center">
  <img src="docs/screenshots/landing.webp" width="49%" alt="Landing page with the headline 'Build Your Perfect Resume with AI' and three feature cards: AI-powered enhancement, professional templates and a live preview editor">
  <img src="docs/screenshots/new-resume-template-picker.webp" width="49%" alt="Create New Resume dialog with a title field, four template thumbnails (Modern, Classic, Minimal, Creative) and a paper size selector">
</p>

<p align="center">
  <img src="docs/screenshots/template-preview-creative.webp" width="49%" alt="Template preview modal rendering the Creative template as a real PDF with built-in sample data">
  <img src="docs/screenshots/editor-creative-template-font-picker.webp" width="49%" alt="Editor with the Creative template applied and the font picker open, listing ATS-friendly sans-serif and serif fonts">
</p>

<p align="center">
  <img src="docs/screenshots/ai-tailor-to-job.webp" width="49%" alt="Tailor to Job dialog with a pasted job description and an Analyze and Tailor button">
  <img src="docs/screenshots/settings-bring-your-own-ai-key.webp" width="49%" alt="Settings page: language selector and AI configuration with provider dropdown open (OpenAI, Anthropic, Google, ZhiPu), model selector and API key field">
</p>

<p align="center">
  <img src="docs/screenshots/sign-in-arabic-rtl.webp" width="49%" alt="Sign-in page in Arabic with a right-to-left layout: email and password form, Google sign-in and translated feature highlights">
  <img src="docs/screenshots/editor-dark-mode.webp" width="49%" alt="Editor in dark mode with the PDF preview slightly dimmed">
</p>

<p align="center">
  <img src="docs/screenshots/arrange-sections.webp" width="49%" alt="Arrange Sections modal with drag handles to reorder Experience, Education, Skills, Projects, Certifications and Languages, plus an input to add a custom section">
  <img src="docs/screenshots/mobile-landing.webp" width="24%" alt="Landing page on a phone-sized screen">
</p>

## Features

**Resume management**
- Email/password or Google sign-in; each user has their own dashboard of resumes.
- Create a resume with a title, one of four templates and a paper size (A4 or US Letter). Rename, duplicate and delete from the dashboard.

**Editor**
- Step-by-step wizard: Personal info, Experience, Education, Skills, Projects, Certifications, Languages, then a Review step. A progress bar lets you jump back to any step you've already reached.
- Split screen with a live PDF preview (zoom in and out) that redraws shortly after you stop typing.
- Rich-text bullet points (Tiptap) and month/year pickers with a "currently working here" option.
- Drag and drop to reorder sections and entries, plus custom sections such as "Volunteer Work".
- Switch template, font (12 ATS-friendly families) and paper size at any time; changes show up in the preview straight away.
- Changes save automatically, and a "Saved" indicator shows the current state. Export a PDF with one click.

**AI assistant (bring your own key)**
- Rewrite a field or the summary, with streamed output you can accept, reject or regenerate.
- Write a summary from your experience, write bullet points for a role, and get skill suggestions grouped by category.
- **Tailor to Job:** paste a job description to get a keyword analysis and suggestions per section. Apply them one at a time or all at once.
- **ATS Score:** an overall score plus four sub-scores (content quality, keyword match, formatting, section completeness), each with specific fixes.
- **Import CV:** upload a PDF, DOCX or TXT file and the AI fills in every editor section from it.
- Target-role autocomplete from a built-in list of 500+ job titles, with AI suggestions when the list has few matches.
- Works with OpenAI, Anthropic, Google Gemini or Z.AI (GLM). Each user picks a provider and model and adds their own API key in Settings.

**Language, theme and devices**
- English and Arabic interface with a full right-to-left layout.
- Arabic PDF output with translated section headings and an Arabic font (Amiri).
- AI answers in the user's interface language.
- Light and dark themes. On phones the editor gets a bottom action bar and a slide-up preview drawer.

## Architecture

```mermaid
flowchart LR
  subgraph Client["Browser"]
    UI["Next.js client components<br/>stepper editor · Tiptap · dnd-kit"]
    PV["Live preview<br/>@react-pdf/renderer → Blob"]
    OBJ["AI SDK useObject<br/>(partial JSON streaming)"]
  end

  subgraph Server["Next.js 16 app (standalone Node server, Docker)"]
    SA["Server Actions<br/>resume CRUD · autosave · settings"]
    AIR["AI route handlers<br/>enhance · summary · bullets · skills<br/>tailor · ATS · CV parse · job titles"]
    PDF["PDF export route<br/>renderToBuffer (same templates)"]
    AUTH["Better Auth<br/>email + Google OAuth · proxy guard"]
    ENC["AES-256-GCM<br/>key vault"]
  end

  DB[("PostgreSQL<br/>Prisma 7")]
  LLM{{"User's AI provider<br/>OpenAI · Anthropic · Google · Z.AI"}}

  UI -- "server actions" --> SA
  UI --> PV
  OBJ -- "fetch + stream" --> AIR
  UI -- "download" --> PDF
  SA --> DB
  PDF --> DB
  AUTH --> DB
  AIR --> ENC --> DB
  AIR -- "user's own key" --> LLM
```

More detail (request flows, data model, deployment) is in [docs/architecture.md](docs/architecture.md).

### Key engineering decisions

- **Users bring their own AI key, and the server encrypts it.** A small provider registry builds an AI SDK model for the user's chosen provider (OpenAI, Anthropic, Google or Z.AI) on every request. Keys are encrypted at rest with AES-256-GCM (random IV plus auth tag) and stored per provider, so users can switch providers without entering keys again. Keys are decrypted only on the server and never sent back to the browser.
- **AI output is typed and streamed.** Tailoring, ATS scoring, bullets and skills use `streamText` with Zod object schemas. On the client, `useObject` renders the partial JSON as it arrives, so suggestions appear as they're generated rather than after a long wait. ATS scoring uses an explicit rubric at low temperature (0.2) so results stay consistent between runs.
- **The preview and the export use the same templates.** The four templates are `@react-pdf/renderer` components. The browser renders them to a Blob for the live preview (debounced, lazy-loaded), and the export route renders the same component to a buffer on the server. The preview therefore matches the exported file. Fonts, including Arabic, are self-hosted TTFs registered with the PDF engine.
- **Resumes are validated JSON documents.** Resume content lives in one Postgres JSON column and is checked against a Zod schema on every save. Autosave is debounced (1.5 s) and reports saved, saving or unsaved status. Every resume query is scoped by both the resume ID and the signed-in user's ID.
- **Language is handled end to end.** next-intl reads the locale from a cookie, and the locale is also saved on the user profile. The `<html dir>` attribute follows the locale. PDF templates receive the locale to translate headings and lay out right to left. The AI writing and scoring prompts get an instruction to answer in the user's language.
- **CV import picks a path by file type.** PDFs go straight to the model as a file part. DOCX files are converted to text with mammoth first. All imports are parsed into the same schema as the editor, with new IDs added, a 10 MB size limit and an allowlist of file types.

## Tech stack

| Layer | Tech |
|---|---|
| Framework | Next.js 16 (App Router, Server Actions, standalone output), React 19, TypeScript |
| UI | Tailwind CSS 4, shadcn/ui (Radix), lucide icons, vaul drawers, sonner toasts, next-themes |
| Editor | Tiptap (rich-text bullets), dnd-kit (drag-and-drop), react-hook-form + Zod |
| PDF | @react-pdf/renderer (client preview and server export), self-hosted TTF fonts incl. Amiri for Arabic |
| AI | Vercel AI SDK v6 (`ai`, `@ai-sdk/react`), providers: OpenAI, Anthropic, Google Gemini, Z.AI (GLM) |
| Auth | Better Auth (email/password, Google OAuth, 7-day sessions with cookie cache) |
| Data | PostgreSQL, Prisma 7 (pg adapter) |
| i18n | next-intl (English, Arabic), RTL layout |
| Infra | Docker multi-stage build (Node 22 Alpine, non-root user), Docker Compose, nginx reverse proxy with HTTPS, GitHub Actions deploy over SSH |

## My role

Solo personal project. I came up with the product, designed it and built it end to end: data model, auth, editor, PDF templates, AI features, i18n/RTL, deployment and CI/CD.

- Built over 9 planned phases, with a written plan and summary for each: foundation and auth, dashboard, PDF templates, editor, AI features, i18n/RTL, UX polish (mobile and dark mode), the stepper editor and CV import.
- Git history: 141 commits between 20 and 25 February 2026. 134 are under my name. The other 7 were committed under a bot identity (Cortex Bot) tied to my GitHub account and cover the Docker/CI deployment, locale-aware AI output and job-title autocomplete.

## Status

- **Live:** [resume.everbuild.cloud](https://resume.everbuild.cloud). Sign up with an email address or Google.
- Built in February 2026. It deploys automatically on every push to `main` (GitHub Actions → Docker Compose on my VPS).
- The AI features need the user's own API key, added in Settings. Without a key, the editor, templates and PDF export still work in full.
- Free to use, with no payments.

---

<p align="center"><sub>© 2026 Dafallah Ibrahim. Screenshots and documentation only. Source code is private.</sub></p>
