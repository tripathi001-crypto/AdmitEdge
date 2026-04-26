# AdmitEdge — MBA Application Platform
## Complete Product Blueprint

---

## 1. SYSTEM ARCHITECTURE

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT (Next.js 14)                   │
│  App Router · React Server Components · Tailwind CSS    │
│                                                         │
│  /onboarding  /dashboard  /profile  /schools  /essays   │
└────────────────────────┬────────────────────────────────┘
                         │ HTTPS / REST
┌────────────────────────▼────────────────────────────────┐
│                 API LAYER (Next.js API Routes)           │
│                                                         │
│  /api/profile   /api/evaluate   /api/schools            │
│  /api/essays    /api/dashboard                          │
└────────────────────────┬────────────────────────────────┘
                         │
          ┌──────────────┴──────────────┐
          │                             │
┌─────────▼──────────┐     ┌───────────▼────────────┐
│   Supabase (PG)     │     │    Claude API           │
│                     │     │  claude-sonnet-4-...    │
│  users              │     │                        │
│  profiles           │     │  profile_eval prompt   │
│  schools            │     │  school_strat prompt   │
│  essays             │     │  essay_assist prompt   │
│  deadlines          │     │                        │
└─────────────────────┘     └────────────────────────┘
```

**Stack Decisions:**
- **Frontend**: Next.js 14 (App Router) — best DX for solo dev, file-based routing, RSC
- **Backend**: Next.js API Routes — no separate server needed for MVP
- **Database**: Supabase (Postgres + Auth + Realtime) — free tier, built-in auth, PG
- **AI**: Claude API (claude-sonnet) — better at voice preservation than GPT-4
- **Auth**: Supabase Auth (email/password + Google OAuth)
- **Hosting**: Vercel — zero config Next.js deployment

---

## 2. DATABASE SCHEMA

```sql
-- Users managed by Supabase Auth (auth.users)

CREATE TABLE profiles (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),

  -- Academics
  undergrad_school TEXT,
  undergrad_major TEXT,
  undergrad_gpa NUMERIC(3,2),
  gpa_scale NUMERIC(3,1) DEFAULT 4.0,
  grad_school TEXT,
  grad_gpa NUMERIC(3,2),

  -- Test Scores
  gmat_total INT,
  gmat_quant INT,
  gmat_verbal INT,
  gmat_ir INT,
  gre_quant INT,
  gre_verbal INT,
  gre_awa NUMERIC(2,1),
  ea_score INT,
  toefl_score INT,
  ielts_score NUMERIC(2,1),

  -- Work Experience
  total_yoe INT,                       -- years of experience
  current_company TEXT,
  current_title TEXT,
  current_industry TEXT,
  work_history JSONB,                  -- [{company, title, start, end, description}]
  leadership_examples JSONB,           -- [{title, description, impact}]
  promotions_count INT,

  -- Achievements
  key_achievements TEXT[],
  publications TEXT[],
  awards TEXT[],
  extracurriculars TEXT[],
  community_service TEXT[],
  international_experience TEXT[],

  -- Goals
  short_term_goal TEXT,
  long_term_goal TEXT,
  why_mba TEXT,
  target_industries TEXT[],
  target_functions TEXT[],
  post_mba_location TEXT,

  -- Demographics (optional, for school strategy)
  citizenship TEXT,
  work_authorization TEXT,
  gender TEXT,
  ethnicity TEXT,
  first_gen_college BOOLEAN,

  -- Evaluation (AI-generated, cached)
  profile_evaluation JSONB,            -- {strengths, weaknesses, gaps, positioning}
  evaluation_generated_at TIMESTAMPTZ
);

CREATE TABLE school_list (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),

  school_name TEXT NOT NULL,
  program TEXT,                        -- 'MBA', 'EMBA', etc.
  tier TEXT CHECK (tier IN ('reach', 'target', 'likely')),
  round INT,                           -- 1, 2, 3
  deadline DATE,
  status TEXT DEFAULT 'researching'
    CHECK (status IN ('researching','applying','submitted','interview','admitted','waitlisted','rejected','withdrawn','enrolled')),
  notes TEXT,
  ai_reasoning TEXT,                   -- why AI suggested this school
  priority INT                         -- user-defined ordering
);

CREATE TABLE essays (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  school_id UUID REFERENCES school_list(id) ON DELETE SET NULL,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),

  school_name TEXT,
  prompt TEXT NOT NULL,
  word_limit INT,
  draft TEXT,                          -- current user draft
  ai_feedback JSONB,                   -- {issues, suggestions, improved_version, changes, explanation}
  mode TEXT CHECK (mode IN ('improve', 'rewrite')),
  version INT DEFAULT 1,
  status TEXT DEFAULT 'drafting'
    CHECK (status IN ('drafting','in_review','polishing','final'))
);

CREATE TABLE essay_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  essay_id UUID REFERENCES essays(id) ON DELETE CASCADE,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  version_number INT,
  draft TEXT,
  ai_feedback JSONB,
  word_count INT
);

CREATE TABLE deadlines (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES auth.users(id) ON DELETE CASCADE,
  school_id UUID REFERENCES school_list(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  due_date DATE NOT NULL,
  type TEXT CHECK (type IN ('application','essay','recommendation','interview','decision')),
  completed BOOLEAN DEFAULT FALSE
);

-- Indexes
CREATE INDEX idx_profiles_user_id ON profiles(user_id);
CREATE INDEX idx_school_list_user_id ON school_list(user_id);
CREATE INDEX idx_essays_user_id ON essays(user_id);
CREATE INDEX idx_essays_school_id ON essays(school_id);
CREATE INDEX idx_deadlines_user_id ON deadlines(user_id);
CREATE INDEX idx_deadlines_due_date ON deadlines(due_date);
```

---

## 3. API DESIGN

### Base URL: `/api/`

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET/PUT | `/profile` | Get or update user profile |
| POST | `/profile/evaluate` | Generate AI profile evaluation |
| POST | `/schools/suggest` | AI school list recommendations |
| GET/POST/PUT | `/schools` | CRUD school list |
| GET/POST | `/essays` | List or create essays |
| GET/PUT/DELETE | `/essays/[id]` | Single essay operations |
| POST | `/essays/[id]/analyze` | AI essay feedback |
| GET | `/dashboard` | Aggregated dashboard data |
| GET | `/deadlines` | Upcoming deadlines |

### Request/Response Examples

**POST /api/essays/[id]/analyze**
```json
// Request
{
  "mode": "improve",
  "draft": "My journey to..."
}

// Response
{
  "issues": [
    { "category": "storytelling", "note": "Opening lacks specificity — replace abstract statement with a concrete moment" },
    { "category": "tone", "note": "Paragraph 3 shifts from personal to corporate-speak" }
  ],
  "suggestions": [
    { "line": "My passion for finance...", "suggestion": "Too generic. Pin this to a specific deal, moment, or turning point." }
  ],
  "improved_version": "...",
  "changes_highlight": [
    { "before": "...", "after": "...", "reason": "..." }
  ],
  "explanation": "..."
}
```

---

## 4. AI PROMPTS (Production Quality)

### 4A. Profile Evaluation Prompt

```
SYSTEM:
You are a senior MBA admissions consultant with 15+ years of experience at top firms. You have helped candidates gain admission to all M7 schools and top-15 programs. You are candid, specific, and never give generic advice. You speak like a trusted advisor, not a chatbot.

Your evaluations are known for being brutally honest yet constructive. You identify the real reasons candidates get rejected (not just surface issues), and you give positioning advice that is specific to this person's profile.

USER:
Evaluate this MBA applicant's profile. Be specific, direct, and consultant-quality. Do not be generic.

PROFILE:
- Undergrad: {undergrad_school}, {undergrad_major}, GPA: {undergrad_gpa}/{gpa_scale}
- Test Score: {test_type} {test_score} (Q: {quant}, V: {verbal})
- Work Experience: {total_yoe} years | {current_title} at {current_company} ({current_industry})
- Career History: {work_history_formatted}
- Key Achievements: {key_achievements}
- Leadership: {leadership_examples}
- Short-term Goal: {short_term_goal}
- Long-term Goal: {long_term_goal}
- Why MBA: {why_mba}

Respond ONLY in this JSON structure:
{
  "strengths": [
    { "area": "string", "detail": "2-3 sentence specific analysis" }
  ],
  "weaknesses": [
    { "area": "string", "detail": "2-3 sentence specific analysis, including real impact on admissions" }
  ],
  "gaps": [
    { "area": "string", "detail": "what's missing and why it matters to adcoms" }
  ],
  "positioning": {
    "narrative": "2-3 paragraph positioning statement for this specific candidate. How should they frame their story? What's the throughline?",
    "differentiators": ["what makes this candidate stand out from 100 others with similar stats"],
    "red_flags_to_address": ["specific things adcoms will question and how to proactively address them"],
    "immediate_actions": ["3-5 concrete things to do NOW to strengthen the application"]
  },
  "overall_assessment": "One honest paragraph. What does this profile's realistic range look like and why?"
}
```

### 4B. School Strategy Prompt

```
SYSTEM:
You are an MBA admissions strategist who builds school lists for a living. You know the nuances of each program — culture fit, what they value, typical profiles admitted, and how different backgrounds play. You give specific reasoning, not generic tier advice.

USER:
Build a strategic school list for this applicant. Include 8-12 schools across reach/target/likely tiers.

PROFILE SUMMARY:
- Stats: {gpa}, {test_type} {test_score}
- Background: {total_yoe} years in {current_industry}, {current_title} at {current_company}
- Goals: {short_term_goal} → {long_term_goal}
- Target sector: {target_industries}
- Location preference: {post_mba_location}
- Citizenship: {citizenship}
- Profile evaluation summary: {evaluation_summary}

For each school, explain:
1. Why this school fits THIS specific candidate (not generic school info)
2. What concerns adcoms will have and how to address them
3. Which programs/clubs/professors make this school strategically fit

Respond ONLY in this JSON:
{
  "schools": [
    {
      "name": "string",
      "tier": "reach|target|likely",
      "fit_score": 1-10,
      "why_apply": "2-3 sentences specific to THIS candidate",
      "adcom_concerns": "what they'll question",
      "strategic_angle": "how to position for this school specifically",
      "recommended_round": 1|2|3,
      "notes": "any additional strategic notes"
    }
  ],
  "strategy_summary": "Overall list strategy in 2 paragraphs",
  "prioritization": "Which 3-4 schools to invest most effort in and why"
}
```

### 4C. Essay Assistant Prompt (Core Feature)

```
SYSTEM:
You are an expert MBA essay editor who has helped hundreds of applicants get into M7 programs. Your specialty is making essays better while keeping the applicant's authentic voice completely intact.

CRITICAL RULES — violating these is a failure:
1. PRESERVE the applicant's voice, rhythm, and personality. If they write casually, keep it casual. If they write formally, keep it formal.
2. DO NOT introduce new ideas, new examples, or new achievements. Everything in the output must come from what the applicant wrote.
3. DO NOT make the essay sound "polished" in a generic AI way. Adcoms can spot AI-edited essays immediately.
4. In IMPROVE mode: change maximum 5-10% of wording. Fix grammar, clarity, and flow only.
5. In REWRITE mode: change maximum 25-30% of wording. Improve storytelling and impact while preserving all stories and examples.
6. The improved essay must sound like a human wrote it on their best day — not like AI helped.

MODE: {mode} (improve | rewrite)
SCHOOL: {school_name}
PROMPT: {essay_prompt}
WORD LIMIT: {word_limit}

APPLICANT'S DRAFT:
---
{draft}
---

Analyze and respond ONLY in this JSON:
{
  "issues": [
    {
      "category": "clarity|storytelling|tone|structure|generic_content|voice|word_count",
      "severity": "high|medium|low",
      "note": "specific observation with line reference if possible"
    }
  ],
  "line_suggestions": [
    {
      "original": "exact phrase from draft",
      "suggestion": "improved version",
      "reason": "why this specific change helps"
    }
  ],
  "improved_version": "The full improved essay. Same length as original unless word limit requires trimming.",
  "changes_highlight": [
    {
      "before": "original phrase",
      "after": "new phrase",
      "reason": "explanation of change"
    }
  ],
  "explanation": "2-3 paragraph explanation of the editing philosophy applied to THIS essay specifically. What were the biggest issues and why did you fix them the way you did?",
  "word_count_original": number,
  "word_count_improved": number,
  "voice_preservation_score": 1-10,
  "voice_notes": "How well does the improved version sound like the original author?"
}
```

---

## 5. FOLDER STRUCTURE

```
admitedge/
├── app/                          # Next.js App Router
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── signup/page.tsx
│   ├── (app)/
│   │   ├── layout.tsx            # Sidebar + nav shell
│   │   ├── dashboard/page.tsx
│   │   ├── onboarding/
│   │   │   ├── page.tsx          # Multi-step onboarding
│   │   │   └── steps/            # Step components
│   │   ├── profile/
│   │   │   ├── page.tsx          # View profile + evaluation
│   │   │   └── edit/page.tsx
│   │   ├── schools/
│   │   │   ├── page.tsx          # School list
│   │   │   └── [id]/page.tsx
│   │   └── essays/
│   │       ├── page.tsx          # Essay list
│   │       └── [id]/page.tsx     # Essay editor (MAIN FEATURE)
│   ├── api/
│   │   ├── profile/
│   │   │   ├── route.ts
│   │   │   └── evaluate/route.ts
│   │   ├── schools/
│   │   │   ├── route.ts
│   │   │   └── suggest/route.ts
│   │   ├── essays/
│   │   │   ├── route.ts
│   │   │   └── [id]/
│   │   │       ├── route.ts
│   │   │       └── analyze/route.ts
│   │   └── dashboard/route.ts
│   ├── layout.tsx                # Root layout
│   └── globals.css
├── components/
│   ├── ui/                       # Base components (Button, Input, etc.)
│   ├── onboarding/               # Step components
│   ├── essay/
│   │   ├── EssayEditor.tsx       # Main editor
│   │   ├── FeedbackPanel.tsx     # AI feedback display
│   │   ├── DiffView.tsx          # Before/after comparison
│   │   └── IssuesList.tsx
│   ├── profile/
│   │   ├── EvaluationCard.tsx
│   │   └── ProfileSummary.tsx
│   ├── schools/
│   │   └── SchoolCard.tsx
│   └── dashboard/
│       └── DeadlineTracker.tsx
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # Browser client
│   │   └── server.ts             # Server client
│   ├── claude/
│   │   ├── client.ts             # API wrapper
│   │   ├── prompts.ts            # All AI prompts
│   │   └── parsers.ts            # Response parsers
│   └── utils.ts
├── types/
│   └── index.ts                  # Shared TypeScript types
├── .env.local
├── next.config.ts
├── tailwind.config.ts
└── package.json
```

---

## 6. STEP-BY-STEP BUILD PLAN (7–10 Days)

### Day 1: Foundation
- [ ] `npx create-next-app@latest admitedge --typescript --tailwind --app`
- [ ] Set up Supabase project, run schema SQL
- [ ] Configure Supabase auth (email + Google)
- [ ] Create login/signup pages
- [ ] Set up environment variables
- [ ] Deploy to Vercel (even empty — get CI working)

### Day 2: Onboarding Flow
- [ ] Multi-step form component (wizard pattern)
- [ ] Steps: Academics → Test Scores → Work Experience → Achievements → Goals
- [ ] Save progress to Supabase on each step
- [ ] Handle partial saves (resume onboarding)
- [ ] Basic validation

### Day 3: Profile + Evaluation
- [ ] Profile view page
- [ ] Profile edit page
- [ ] Claude API integration (`/api/profile/evaluate`)
- [ ] Evaluation display component (strengths/weaknesses/gaps cards)
- [ ] Cache evaluation in DB, add "Regenerate" button

### Day 4: School Strategy
- [ ] School list page (table view)
- [ ] AI school suggestion endpoint (`/api/schools/suggest`)
- [ ] School card with tier badge + AI reasoning
- [ ] Manual add/edit/delete schools
- [ ] Status tracking dropdown

### Day 5–6: Essay Assistant (2 days — most important)
- [ ] Essay list page
- [ ] Essay editor layout (split pane: editor left, feedback right)
- [ ] Prompt + draft input
- [ ] Mode selector (Improve / Rewrite)
- [ ] Claude essay analysis endpoint (carefully tuned prompt)
- [ ] Issues list component
- [ ] Line-by-line suggestions component
- [ ] Improved version display
- [ ] Before/after diff view (highlight changes)
- [ ] Explanation panel
- [ ] Version history (save each analysis)
- [ ] Word count tracker

### Day 7: Dashboard
- [ ] Overview stats (schools applied, essays done, days to deadline)
- [ ] Upcoming deadlines list
- [ ] School list summary
- [ ] Essay progress by school
- [ ] Quick links to priority items

### Day 8: Polish + QA
- [ ] Error handling throughout
- [ ] Loading states on all AI calls
- [ ] Mobile responsive check
- [ ] Edge cases (empty states, no profile, etc.)
- [ ] Test all Claude prompts with real profile data
- [ ] Rate limiting on AI endpoints (prevent abuse)

### Day 9: Personal Use Testing
- [ ] Enter YOUR actual profile
- [ ] Run actual essays through the tool
- [ ] Tune prompts based on output quality
- [ ] Fix bugs found during real use

### Day 10: Buffer / Launch Prep
- [ ] Custom domain
- [ ] Basic analytics (Vercel Analytics)
- [ ] Waitlist page for R1 launch
- [ ] README

---

## 7. ENVIRONMENT SETUP

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key
ANTHROPIC_API_KEY=your_anthropic_key

# Install
npm install @supabase/supabase-js @supabase/ssr @anthropic-ai/sdk
npm install @radix-ui/react-dialog @radix-ui/react-select
npm install react-diff-viewer-continued  # for before/after diff
npm install react-textarea-autosize
npm install date-fns                     # deadline calculations
```

---

## 8. KEY IMPLEMENTATION NOTES

### Essay Editor — Critical Details
The essay editor is the product. Get this right.

1. **Auto-save draft** every 30 seconds to Supabase
2. **Word count** — live, show against limit with color warning
3. **Version history** — save snapshot before every AI analysis
4. **Streaming responses** — stream Claude output for better UX (use `stream: true`)
5. **Diff algorithm** — use Levenshtein distance to highlight actual changes
6. **Voice score display** — show the voice preservation score prominently

### Claude API Tips
```typescript
// Always use streaming for long outputs
const stream = await anthropic.messages.stream({
  model: 'claude-sonnet-4-20250514',
  max_tokens: 4096,
  messages: [{ role: 'user', content: prompt }]
});

// Parse JSON carefully — Claude sometimes adds prose before/after
function extractJSON(text: string) {
  const match = text.match(/\{[\s\S]*\}/);
  return match ? JSON.parse(match[0]) : null;
}
```

### Supabase Row-Level Security (RLS)
```sql
-- Enable RLS on all tables
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE school_list ENABLE ROW LEVEL SECURITY;
ALTER TABLE essays ENABLE ROW LEVEL SECURITY;

-- Users can only see their own data
CREATE POLICY "Users own their data" ON profiles
  FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "Users own their data" ON school_list
  FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "Users own their data" ON essays
  FOR ALL USING (auth.uid() = user_id);
```

---

## 9. UI WIREFRAMES (Text-Based)

### Dashboard
```
┌─ Sidebar ──────┐ ┌─ Main Content ──────────────────────────────────┐
│ AdmitEdge      │ │                                                 │
│                │ │  Good morning, Arjun                           │
│ ○ Dashboard    │ │  R1 deadline in 47 days                        │
│ ○ Profile      │ │                                                 │
│ ○ Schools      │ │  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐          │
│ ○ Essays       │ │  │  8   │ │  3   │ │  12  │ │  2   │          │
│ ○ Deadlines    │ │  │Schools│ │Essays│ │ Days │ │Done  │          │
│                │ │  └──────┘ └──────┘ └──────┘ └──────┘          │
│                │ │                                                 │
│                │ │  UPCOMING DEADLINES                            │
│                │ │  ── Wharton R1     Oct 1  ████░░ 6 essays      │
│                │ │  ── HBS R1         Sep 6  ███░░░ 4 essays      │
│                │ │  ── Booth R1       Oct 2  █░░░░░ 2 essays      │
│                │ │                                                 │
│                │ │  ESSAY PROGRESS                                │
│                │ │  Wharton  ██████░░░░  3/5 essays               │
│                │ │  HBS      ████░░░░░░  2/5 essays               │
└────────────────┘ └─────────────────────────────────────────────────┘
```

### Essay Editor
```
┌─ Essay Editor ─────────────────────────────────────────────────────┐
│ Wharton › Essay 1 › "What do you hope to gain..."    [Analyze ▶]  │
├─────────────────────────┬──────────────────────────────────────────┤
│ YOUR DRAFT              │ AI FEEDBACK                              │
│                         │                                          │
│ Mode: [Improve ▼]       │ ┌─ Issues (3) ───────────────────────┐  │
│ Word count: 487/500     │ │ ⚠ Storytelling  Opening too vague  │  │
│                         │ │ ⚠ Tone          P3 shifts to corp  │  │
│ [Draft textarea...]     │ │ ℹ Clarity       2 unclear referenc │  │
│                         │ └────────────────────────────────────┘  │
│                         │                                          │
│                         │ ┌─ Improved Version ─────────────────┐  │
│                         │ │ [Improved text...]                 │  │
│                         │ │                                    │  │
│                         │ │ [Copy] [Use This Draft]            │  │
│                         │ └────────────────────────────────────┘  │
│                         │                                          │
│                         │ ┌─ Changes Made ─────────────────────┐  │
│                         │ │ "My passion for..." →              │  │
│                         │ │ "In 2019, when..."                 │  │
│                         │ │ Reason: Opens with scene not claim │  │
│                         │ └────────────────────────────────────┘  │
│                         │                                          │
│                         │ Voice score: 9/10                        │
└─────────────────────────┴──────────────────────────────────────────┘
```

---

## 10. MONETIZATION (FOR LAUNCH)

When you launch for R1 applicants:

| Tier | Price | Features |
|------|-------|----------|
| Free | $0 | Profile eval, 2 essays, 3 schools |
| Applicant | $49/mo | Unlimited essays, full school list, deadlines |
| Premium | $149/mo | Everything + priority AI (faster), export to PDF |

Use Stripe for billing. Add a `/api/billing` route when ready.

---

## 11. WHAT MAKES THIS DIFFERENT FROM APPLICANTLAB

1. **Voice preservation is the product** — most tools rewrite essays in AI voice. This explicitly doesn't.
2. **Consultant-quality prompts** — evaluation reads like advice from a real advisor
3. **Diff view** — show exactly what changed and why (transparency builds trust)
4. **Voice score** — quantify how much the AI changed the essay
5. **School strategy is personalized** — not just tier matching, actual fit reasoning
