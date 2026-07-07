# JobFlow — Build Plan & Technical Guide

> A personalized, research-driven job search workspace. Think Linear meets Overleaf meets PitchBook.

---

## 1. Vision & Core Problem

Mass applying creates a black hole. Personalization at scale is the answer — but it requires deep research, good tooling, and a workflow that ties everything together. This app is that workflow.

**The three-sentence pitch:**
Paste a job link. The app researches the company, finds the recruiter, rewrites your resume, and drafts a cover letter using the exact language of the posting. You spend your time deciding — not formatting.

---

## 2. Feature Map (Prioritized)

### TIER 1 — MVP (Build First)
These are the features that create the core value loop.

| Feature | What It Does |
|---|---|
| **Job Intake** | Paste URL → scrape & parse job posting (title, description, requirements, YOE) |
| **Resume Workspace** | View, upload, and manage multiple resumes |
| **AI Resume Tailoring** | Generate 4 resume variants based on job + your base resumes |
| **Cover Letter Generator** | Draft using the job posting's own language and tone |
| **Company Profile** | Auto-pull company info (stack, size, what they build) |

### TIER 2 — Core Product
| Feature | What It Does |
|---|---|
| **Recruiter Sourcing** | Find recruiters at target company via LinkedIn/Crunchbase |
| **Recruiter Profile Cards** | Background, recent activity, what they're hiring for |
| **Explore Dashboard** | YC/a16z startups, recent news, acquisitions, new batches |
| **Resume Editor** | Monaco-based LaTeX/Markdown editor with live preview |

### TIER 3 — Future Scope
| Feature | What It Does |
|---|---|
| **Cold Email Drafts** | Research company → suggest relevant project → draft outreach |
| **Recruiting Calendar** | Company recruiting timelines (when they hire, when to apply) |
| **Project Suggestions** | "Build X to get into Y" — tailored project ideas per company |
| **Referral Tracker** | Who you've contacted, when, and follow-up reminders |

---

## 3. Tech Stack

### Frontend
```
Next.js 14 (App Router)    → Routing, SSR, API routes (one framework to rule them all)
TypeScript                 → Catch errors before they happen
Tailwind CSS               → Utility-first styling, fast iteration
Framer Motion              → Animations (the orb loader, tab transitions)
Monaco Editor              → VS Code-style resume editor (LaTeX/Markdown)
shadcn/ui                  → Accessible, unstyled component primitives
KaTeX / react-latex        → Render LaTeX resumes in the browser
Zustand                    → Lightweight global state (resume list, active job, etc.)
```

### Backend (via Next.js API Routes — no separate server in MVP)
```
Next.js API Routes         → Your backend lives in /app/api/
Prisma                     → ORM — write TypeScript, get SQL
PostgreSQL (via Supabase)  → Database — free tier is generous
Redis (via Upstash)        → Cache API responses (job scrapes, company data)
```

### AI & Data
```
Anthropic Claude API       → Resume generation, cover letters, analysis
Playwright / Cheerio       → Scrape job postings (LinkedIn blocks bots — use Playwright)
Crunchbase API             → Company profiles, funding, stack info (free tier exists)
Hacker News Algolia API    → "Who is hiring" threads, startup pulse
NewsAPI.org                → Headlines for Explore dashboard (free tier: 100 req/day)
PitchBook API              → (Paid, enterprise) Replace with Crunchbase for MVP
LinkedIn API               → (Very restricted) Use Hunter.io or Apollo.io for recruiter emails
```

### Auth & Storage
```
Supabase Auth              → Email/password, Google OAuth — free tier
Supabase Storage           → Store resume PDFs, LaTeX files
Vercel                     → Deploy Next.js with zero config (free tier)
```

---

## 4. Architecture Overview

```
┌────────────────────────────────────────────────────┐
│                     Browser                         │
│  Next.js App                                       │
│  ┌──────────┐  ┌───────────┐  ┌────────────────┐  │
│  │ Resume   │  │  Explore  │  │  Job Workflow  │  │
│  │ Workspace│  │ Dashboard │  │  (Core Loop)   │  │
│  └──────────┘  └───────────┘  └────────────────┘  │
└────────────────────┬───────────────────────────────┘
                     │ API Routes (/app/api/)
         ┌───────────┴────────────────┐
         │                            │
    ┌────▼─────┐              ┌───────▼──────┐
    │ Supabase │              │  Claude API   │
    │ Postgres │              │  Anthropic    │
    │ Storage  │              └───────┬───────┘
    └──────────┘                      │
                               uses prompts +
                               job data + resumes
         ┌───────────────────────────────────┐
         │        External APIs              │
         │  Playwright (job scraper)         │
         │  Crunchbase (company data)        │
         │  NewsAPI (explore feed)           │
         │  Apollo.io (recruiter sourcing)   │
         └───────────────────────────────────┘
```

---

## 5. Database Schema (Core Tables)

```sql
-- Resumes you own
resumes (
  id, user_id, title, content_latex TEXT, content_md TEXT,
  file_url, created_at, updated_at, tags[]
)

-- Companies you've researched
companies (
  id, name, website, description, tech_stack[],
  funding_stage, headcount, crunchbase_url, notes
)

-- Recruiters at companies
recruiters (
  id, company_id, name, linkedin_url, role,
  email, background_notes, last_active
)

-- Job postings you've imported
jobs (
  id, user_id, company_id, title, raw_description,
  parsed_requirements, years_of_experience, salary_range,
  posted_url, scraped_at, status ENUM(saved, applied, interviewing, rejected)
)

-- Generated resumes for a specific job
job_resumes (
  id, job_id, base_resume_id, generated_content,
  variant_number INT, selected BOOLEAN, created_at
)

-- Generated cover letters
cover_letters (
  id, job_id, job_resume_id, content TEXT, created_at
)

-- Applications tracker
applications (
  id, job_id, applied_at, follow_up_date,
  notes, referral_contact, outcome
)
```

---

## 6. Core Workflow — Step by Step

This is the most important part of the app. Here is the exact data flow:

```
User pastes job URL
       ↓
[API] /api/jobs/parse
  → Playwright opens URL in headless browser
  → Extracts: title, company, description, requirements, YOE, salary
  → Stores raw + parsed in jobs table
       ↓
[API] /api/companies/enrich
  → Search Crunchbase for company
  → Pull: description, tech stack, headcount, funding
  → Store in companies table
       ↓
[API] /api/recruiters/source
  → Search Apollo.io or Hunter.io for recruiters at that company
  → Claude summarizes each recruiter's background from LinkedIn text
  → Store recruiter cards
       ↓
[API] /api/resumes/generate
  → Pull user's base resumes from DB
  → Construct Claude prompt with:
      - Job description
      - Company context
      - Recruiter priorities
      - User's existing resumes
      - Resume guidelines (from MCP or hardcoded system prompt)
  → Generate 4 variants (aggressive ATS, narrative, skills-first, hybrid)
  → Stream back to UI with loading orb animation
       ↓
User selects a variant
       ↓
[API] /api/cover-letters/generate
  → Extract key phrases, descriptors, and qualifiers from job description
  → Claude matches these in the cover letter using same language
  → Returns cover letter draft
       ↓
User edits, exports, and applies
```

---

## 7. Design Language — Apple iOS Aesthetic

### Color Palette
```
Background:   #F2F2F7   (iOS system background)
Surface:      #FFFFFF   (cards, modals)
Surface 2:    #EBEBF0   (secondary backgrounds)
Accent:       #0A84FF   (iOS blue — primary actions)
Success:      #30D158   (iOS green)
Warning:      #FF9F0A   (iOS orange)
Text Primary: #000000
Text Muted:   #8E8E93
Separator:    #C6C6C8
```

### Typography
```
SF Pro Display   → headings (use system-ui or Inter as web fallback)
SF Pro Text      → body text
SF Mono          → code, LaTeX editor
```

### Key Design Tokens
- Border radius: `12px` (cards), `8px` (buttons), `16px` (modals)
- Shadows: soft, layered — `0 2px 8px rgba(0,0,0,0.08)` 
- Blur: `backdrop-filter: blur(20px)` for frosted glass panels
- Transitions: `200ms ease` for all state changes
- No sharp corners anywhere — everything rounded

### Signature Element
The **loading orb** during resume generation — a soft glowing sphere with orbiting particles (Framer Motion), inspired by Dia browser's AI indicator. This is the moment the app feels alive.

---

## 8. Folder Structure

```
/jobflow
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── signup/page.tsx
│   ├── (app)/
│   │   ├── layout.tsx            ← App shell, sidebar nav
│   │   ├── resumes/
│   │   │   ├── page.tsx          ← Resume list (card grid)
│   │   │   └── [id]/page.tsx     ← Monaco editor + preview
│   │   ├── explore/
│   │   │   └── page.tsx          ← Dashboard: startups, news, funding
│   │   └── jobs/
│   │       ├── page.tsx          ← Job list / search
│   │       └── [id]/page.tsx     ← Full workflow view
│   └── api/
│       ├── jobs/
│       │   ├── parse/route.ts
│       │   └── [id]/route.ts
│       ├── companies/
│       │   └── enrich/route.ts
│       ├── recruiters/
│       │   └── source/route.ts
│       ├── resumes/
│       │   └── generate/route.ts
│       └── cover-letters/
│           └── generate/route.ts
├── components/
│   ├── ui/                       ← shadcn components
│   ├── resume-card.tsx
│   ├── job-intake-modal.tsx
│   ├── recruiter-card.tsx
│   ├── resume-variant-card.tsx
│   ├── loading-orb.tsx           ← The orb animation
│   └── explore-feed.tsx
├── lib/
│   ├── db.ts                     ← Prisma client
│   ├── claude.ts                 ← Claude API wrapper
│   ├── scraper.ts                ← Playwright job parser
│   └── prompts/
│       ├── resume-generation.ts
│       └── cover-letter.ts
├── prisma/
│   └── schema.prisma
└── types/
    └── index.ts
```

---

## 9. Claude Prompt Strategy (The Brain)

### Resume Generation Prompt Structure
```typescript
const systemPrompt = `
You are a senior technical recruiter and resume strategist.
You will receive a job description, company context, recruiter background,
and a candidate's existing resume. Generate 4 resume variants.

Rules:
- Maintain ALL factual claims from the original resume (no hallucination)
- Translate passive descriptions into active, metric-driven bullets
- Mirror the language and terminology from the job description
- Variant 1: ATS-optimized (keyword dense, clean structure)
- Variant 2: Narrative-led (story of growth, company-fit focused)
- Variant 3: Skills-first (technical depth, tools front and center)
- Variant 4: Hybrid (balanced, your call as the expert)

Format each variant as valid LaTeX using the resume template provided.
`;
```

### Cover Letter Prompt Strategy
```typescript
// Extract language fingerprint from job posting first
const extractionPrompt = `
Extract from this job description:
1. Key action verbs they use (e.g., "drive", "own", "scale")
2. Technical descriptors (e.g., "high-throughput", "distributed")
3. Cultural signals (e.g., "fast-paced", "collaborative")
4. Specific qualifications they emphasize

Return as JSON: { verbs, descriptors, cultural, qualifications }
`;

// Then generate cover letter using extracted language
const coverLetterPrompt = `
Using ONLY this vocabulary fingerprint: ${fingerprint}
Write a cover letter that feels like it was written by someone who
already works at this company and thinks in their language.
`;
```

---

## 10. Phase-by-Phase Build Plan

### Phase 0 — Setup (Day 1–2)
- [ ] `npx create-next-app@latest jobflow --typescript --tailwind --app`
- [ ] Install dependencies: `shadcn/ui`, `framer-motion`, `zustand`, `prisma`
- [ ] Set up Supabase project (free) → get connection string
- [ ] Configure Prisma with your schema
- [ ] Deploy a hello world to Vercel
- [ ] Set up environment variables (`.env.local`)

### Phase 1 — Resume Workspace (Week 1)
- [ ] Build the resume card grid layout
- [ ] File upload (PDF + LaTeX) → Supabase Storage
- [ ] Resume detail page (just display for now)
- [ ] Install Monaco Editor → render LaTeX source
- [ ] Add basic KaTeX preview panel (side by side)

**Skill you'll learn:** React state, file uploads, component composition

### Phase 2 — Job Intake & Parsing (Week 2)
- [ ] Build the "paste URL" intake modal (Cmd+V listener)
- [ ] Write the Playwright scraper for LinkedIn/Greenhouse/Lever job pages
- [ ] Parse and display: title, company, requirements, YOE
- [ ] Save job to database
- [ ] Build job detail page (two-column: posting left, workspace right)

**Skill you'll learn:** API routes, web scraping, data modeling

### Phase 3 — AI Resume Generation (Week 3)
- [ ] Write the Claude resume prompt in `/lib/prompts/resume-generation.ts`
- [ ] Build streaming API route (`/api/resumes/generate`)
- [ ] Build the loading orb component (Framer Motion)
- [ ] Display 4 generated resume variant cards
- [ ] Let user select one and mark it as "chosen"

**Skill you'll learn:** Streaming APIs, prompt engineering, async state

### Phase 4 — Cover Letter + Company/Recruiter (Week 4)
- [ ] Company enrichment via Crunchbase API
- [ ] Recruiter sourcing via Apollo.io free tier
- [ ] Cover letter generation using language fingerprinting
- [ ] Recruiter profile cards component

**Skill you'll learn:** Third-party API integration, data transformation

### Phase 5 — Explore Dashboard (Week 5)
- [ ] YC companies list (HN API + YC's public directory)
- [ ] News feed via NewsAPI
- [ ] Company cards with funding info
- [ ] Filter by stage, sector, batch year

**Skill you'll learn:** Data fetching patterns, list virtualization

---

## 11. What You Specifically Need to Know

### JavaScript / TypeScript Fundamentals
- `async/await` and Promises — you'll use these constantly
- Array methods: `.map()`, `.filter()`, `.reduce()` — for transforming job/resume data
- Destructuring, spread operator, optional chaining (`?.`)
- TypeScript interfaces and types (not complex generics — just basic typing)

### React Essentials
- `useState` and `useEffect` — most of your local UI logic
- `useRef` — for the Monaco editor and Cmd+V listener
- Custom hooks — e.g., `useJobParser()`, `useResumeStore()`
- Component composition (passing props, children pattern)

### Next.js Specifics
- App Router (not Pages Router — start with App Router)
- Server Components vs Client Components (`"use client"` directive)
- API Routes (`route.ts` files) — this is your backend
- `fetch` in Server Components (Next.js supercharges this)

### CSS / Tailwind
- Flexbox and Grid — you'll use these for every layout
- Responsive prefixes (`sm:`, `md:`, `lg:`)
- The Tailwind config file for custom colors (your iOS palette goes here)

### Database Basics
- What a foreign key is (jobs belong to companies)
- Basic Prisma queries: `findMany`, `create`, `update`, `upsert`
- You don't need raw SQL — Prisma abstracts it

### Claude API
- How to structure a `messages` array
- System prompts vs user prompts
- Streaming responses (`stream: true`) for the generation animation

---

## 12. Day 1 Action Plan

Do exactly this today. In order.

```bash
# 1. Create the project
npx create-next-app@latest jobflow --typescript --tailwind --app --src-dir
cd jobflow

# 2. Install core dependencies
npm install framer-motion zustand @monaco-editor/react
npm install @prisma/client prisma
npm install @supabase/supabase-js

# 3. Set up shadcn/ui
npx shadcn@latest init
npx shadcn@latest add button card tabs input badge separator

# 4. Initialize Prisma
npx prisma init

# 5. Set up your folder structure
mkdir -p src/components/ui src/lib src/types src/lib/prompts
```

Then do these three things:
1. **Build the nav shell** — a sidebar with "Resumes", "Explore", "Jobs" links. No functionality. Just the layout.
2. **Build one static resume card** — hardcode fake data. A card with name, last modified, a "View" button. Get it looking right.
3. **Call the Claude API once** — write a single test route at `/api/test/route.ts` that sends "Say hello" to Claude and returns the response. Confirm it works.

That's it. Three things. By end of day you'll have the skeleton of the app running and confirmation that your AI integration works.

---

## 13. Key Libraries Reference

| What you need | Library | Docs |
|---|---|---|
| VS Code editor in browser | `@monaco-editor/react` | github.com/suren-atoyan/monaco-react |
| Animations + orb | `framer-motion` | framer.com/motion |
| Global state | `zustand` | docs.pmnd.rs/zustand |
| Database ORM | `prisma` | prisma.io/docs |
| Auth + DB hosting | `supabase` | supabase.com/docs |
| UI components | `shadcn/ui` | ui.shadcn.com |
| LaTeX rendering | `katex` | katex.org |
| Web scraping | `playwright` | playwright.dev |
| HTTP requests | `axios` or native `fetch` | — |
| Date formatting | `date-fns` | date-fns.org |

---

## 14. Potential Blockers (Solve These Early)

| Blocker | Why It's Hard | Solution |
|---|---|---|
| LinkedIn scraping | LinkedIn actively blocks bots | Use Playwright + residential proxy, or use Apify's LinkedIn Actor |
| PitchBook API | Enterprise pricing ($$$) | Use Crunchbase free tier + HN Algolia API instead |
| LaTeX rendering in browser | Complex dependency | Use KaTeX for display, Monaco for editing raw `.tex` |
| Claude streaming in Next.js | Needs Edge runtime | Use `export const runtime = 'edge'` in your API route |
| Job sites blocking scraping | Greenhouse/Lever are okay; LinkedIn is not | Use Playwright for LinkedIn, Cheerio for Greenhouse/Lever |

---

## 15. The Honest Learning Path

If you're coding this yourself with occasional AI help:

**Week 1–2:** You'll feel slow. That's normal. Every bug is a lesson.
**Week 3:** The Claude integration will feel like magic and that momentum will carry you.
**Week 4+:** You'll start predicting what you need before you search for it.

The rule: **write the code first, then ask AI to explain what you didn't understand**. Not the other way around. When you get stuck, describe the problem in words first — if you can name it, you can search for it.

The app you're describing is genuinely ambitious. The core workflow (URL → resume + cover letter) is buildable in 3–4 focused weeks. Everything else layers on top.
```

---

*Stack summary: Next.js 14 + TypeScript + Tailwind + Supabase + Prisma + Framer Motion + Monaco + Claude API*
