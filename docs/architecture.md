# ResumeCraft AI: architecture

This page goes deeper than the [README](../README.md). It covers how requests flow through the app, what is stored and how the app is shipped. Everything here comes from the private source code. Code is not reproduced.

## 1. System overview

ResumeCraft AI is a single Next.js 16 application (App Router) that serves the marketing page, the signed-in app and its API from one Node process. There is no separate backend. Reads and writes go through **Server Actions**, and AI and PDF work goes through **route handlers**.

```mermaid
flowchart TB
  subgraph Browser
    M["Marketing + auth pages"]
    D["Dashboard"]
    E["Editor shell<br/>stepper · forms · toolbar"]
    P["Live PDF preview<br/>(react-pdf in the browser)"]
    A["AI panel (sheet)<br/>useObject / text stream"]
  end

  subgraph NextJS["Next.js 16 server"]
    PX["proxy (route guard)<br/>dashboard, login, register"]
    SA["Server Actions<br/>createResume · duplicateResume · renameResume<br/>saveResumeContent · updateResumeTemplate<br/>updateResumeFontFamily · saveJobDescription<br/>deleteResume · AI settings · locale"]
    AI["AI routes /api/ai/*<br/>enhance · summary · bullets · skills<br/>tailor · ats · parse-cv"]
    JT["Job titles route<br/>local list → AI fallback"]
    PDF["PDF export route<br/>server-side render"]
    BA["Better Auth routes"]
  end

  DB[("PostgreSQL")]
  LLM{{"OpenAI · Anthropic · Google · Z.AI"}}

  M --> BA
  D --> SA
  E --> SA
  E --> P
  A --> AI
  E --> JT
  E --> PDF
  PX --> BA
  SA --> DB
  AI --> DB
  AI --> LLM
  JT --> LLM
  PDF --> DB
  BA --> DB
```

## 2. Editing and autosave

```mermaid
sequenceDiagram
  participant U as User
  participant H as useResumeEditor / useAutoSave
  participant PV as Preview panel
  participant SA as saveResumeContent (Server Action)
  participant DB as PostgreSQL

  U->>H: edit a field
  H->>PV: new ResumeData (debounced re-render)
  PV->>PV: lazy-import react-pdf, render template to Blob, swap iframe URL
  H-->>H: status = "unsaved", reset 1.5 s timer
  H->>SA: latest ResumeData (after 1.5 s idle)
  SA->>SA: auth check, Zod resumeDataSchema.parse
  SA->>DB: UPDATE resume SET content WHERE id AND userId
  SA-->>H: success → status = "saved"
```

- Editor state is one `ResumeData` object: personal info, experience, education, skills, projects, certifications, languages, custom sections and section order.
- The latest data is kept in a ref, so the debounced save always sends the newest version, and pending saves are cancelled when the editor unmounts.
- The preview revokes the previous Blob URL whenever it renders a new one, which avoids memory leaks during long sessions.

## 3. AI request flow

```mermaid
sequenceDiagram
  participant C as AI panel (useObject)
  participant R as /api/ai/tailor (example)
  participant K as Key vault
  participant L as User's provider

  C->>R: POST resumeData, jobDescription, locale
  R->>R: Better Auth session check
  R->>K: load AISettings for user
  K-->>R: provider, model, AES-256-GCM decrypted key
  R->>L: streamText(system prompt + language instruction, Output.object(Zod schema))
  L-->>R: token stream
  R-->>C: text stream of partial JSON
  C->>C: render suggestions as they arrive, apply one or all
```

- **Provider registry.** It maps `openai | anthropic | google | zhipu` plus a model ID to an AI SDK model created with the user's own key. The models offered come from a typed config: GPT-4.1 family, Claude Sonnet/Haiku, Gemini 2.5 family and GLM-4.x.
- **Structured output.** Tailoring, ATS score, bullets and skills each have a Zod schema. Free-text features (enhance, summary) stream plain text.
- **Errors.** Routes return stable codes such as `UNAUTHORIZED`, `AI_NOT_CONFIGURED`, `UNSUPPORTED_FORMAT` and `PARSE_FAILED`. The UI turns these into translated messages. If no AI provider is configured, the AI panel shows a "Set up AI" prompt that links to Settings.
- **Locale.** The writing and scoring routes (enhance, summary, bullets, skills, tailor, ATS) add a language instruction to the system prompt, so Arabic users receive Arabic suggestions.
- **CV parsing.** PDFs are sent to the model as a file part. DOCX files are converted to text with mammoth, and plain-text files are passed through as they are. The result is post-processed: IDs are added to entries and a section order is derived from the sections that have content. Uploads are limited to 10 MB and an allowlist of file types.
- **Job-title autocomplete.** The server searches a local list first. It calls the user's AI provider only when there are fewer than five matches and the user is signed in with AI configured.

## 4. PDF generation

| | Live preview | Export |
|---|---|---|
| Where | Browser | Server route handler |
| How | `pdf(element).toBlob()` → iframe | `renderToBuffer(element)` → `application/pdf` attachment |
| Input | In-memory editor state | Stored resume JSON, user locale, paper size, font |
| Template | Same React component from the template registry | Same |

The four templates are Modern, Classic, Minimal and Creative. They support A4 and US Letter, 12 font families, custom sections, rich-text bullets and a locale prop that switches headings and direction for Arabic. The fonts are self-hosted TTF files (Inter, Roboto, Lora, EB Garamond, Amiri and others) registered with the PDF engine, so the output looks the same on every machine. `@react-pdf/renderer` is marked as an external server package, which keeps it out of the server bundle.

## 5. Data model

```mermaid
erDiagram
  USER ||--o{ RESUME : owns
  USER ||--o| AI_SETTINGS : has
  USER ||--o{ SESSION : has
  USER ||--o{ ACCOUNT : "auth providers"

  USER {
    string id
    string name
    string email
    string locale
  }
  RESUME {
    string id
    string title
    string templateId
    json content
    string paperSize
    string fontFamily
    string jobDescription
  }
  AI_SETTINGS {
    string provider
    string model
    json encryptedApiKeys "per-provider AES-256-GCM ciphertext"
  }
```

- When a user is deleted, their resumes, AI settings, sessions and accounts are deleted with them (cascading deletes).
- Resume content is stored as one validated JSON document. This keeps sections flexible (custom sections, ordering) and avoids a wide relational schema.

## 6. Auth and route protection

- Better Auth with the Prisma adapter supports email and password (8–128 characters) and Google OAuth. Sessions last 7 days, are refreshed daily and use a 5-minute cookie cache to reduce database reads.
- A Next.js `proxy` guards `/dashboard`, `/login` and `/register`: signed-out users are redirected to log in, and signed-in users are redirected away from the auth pages. Server Actions and route handlers check the session again themselves.

## 7. Internationalisation

- next-intl loads `en` or `ar` messages based on a `locale` cookie. When the user changes language, the choice is also saved on their profile.
- The root layout sets `lang` and `dir` from the locale. Components use logical CSS properties (start/end), so the RTL layout works without separate stylesheets.

## 8. Build and deployment

```mermaid
flowchart LR
  Dev["git push main"] --> GA["GitHub Actions<br/>deploy workflow"]
  GA -- "SSH" --> G
  subgraph VPS["VPS"]
    direction TB
    G["git pull"] --> B["docker compose build"]
    B --> MIG["one-off migrate container<br/>prisma db push"]
    B --> APP["app container<br/>Next.js standalone server<br/>(non-root user)"]
    NG["nginx (HTTPS)"] --> APP
    APP --> PG[("shared PostgreSQL<br/>(Docker network)")]
  end
```

- Multi-stage Dockerfile on Node 22 Alpine. The builder stage generates the Prisma client and runs `next build` with `output: "standalone"`. The runtime stage copies only the standalone server, static and public assets, the Prisma schema and the generated client, and runs as an unprivileged user.
- The app container listens only on the loopback interface and is published through nginx with TLS.
- Old Docker images are pruned after each deploy.
