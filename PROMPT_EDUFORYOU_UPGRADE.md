# PROMPT: Upgrade EduForYou to Complete Platform

Paste this entire prompt into a Claude Code session opened on the `eduforyou-61697a69` repository.

---

## Context

I have two repositories:
1. **eduforyou-61697a69** (this repo) - Lovable project, React + Vite + Supabase, partially built
2. **highticket** (https://github.com/alin-blip/highticket) - Complete full-stack app with all features, built on Manus (Express + tRPC + Drizzle + SQLite)

I need you to **port all missing features from highticket into this eduforyou Lovable project**, adapting from tRPC/Express to Supabase. The eduforyou project already has the correct Lovable structure (src/, Supabase Edge Functions, supabase auth). Keep everything compatible with Lovable.

## Current State of eduforyou

### Tech Stack (DO NOT CHANGE)
- React 18 + TypeScript + Vite 5
- react-router-dom (NOT wouter)
- Tailwind CSS v3 (NOT v4)
- shadcn/ui components in src/components/ui/
- Supabase for auth + database + edge functions
- @tanstack/react-query for data fetching
- Supabase client at src/integrations/supabase/client.ts

### Existing Routes (src/App.tsx)
```
/ → Index (homepage)
/cursuri → Courses
/cursuri/:slug → CourseDetail
/cursuri-profesionale → Courses (reused)
/eligibilitate → Eligibility
/ikigai → IkigaiQuiz
/locatii → Locations
/locatii/:slug → LocationDetail
/contact → Contact
/about, /why-free, /team, /partners → About (all same page!)
/blog → Blog
/blog/:slug → BlogPost
/webinar → Webinar (single page)
/calculator-finantare → FinanceCalculator
/student-finance → StudentFinance
/careers → Careers
/reviews → Reviews
/ebook → Ebook
/agents → Agents
/book-appointment → BookAppointment
/payment-success → PaymentSuccess
/login → Login (single login page)
/admin/* → AdminDashboard (monolithic)
/agent/* → AgentDashboard (monolithic)
/student/* → StudentDashboard (monolithic)
/ceo → CeoDashboard (single page)
/legal/* → Legal pages
```

### Existing Supabase Tables (27 total - DO NOT recreate these)
profiles, user_roles, contacts, applications, quiz_results, blog_posts, campuses, appointments, sms_logs, abandoned_carts, referrals, student_documents, student_gamification, student_cv, ceo_tasks, ceo_okrs, messages, skill_entries, ikigai_results, offers, social_profiles, outreach_templates, orders, email_templates, email_sends, email_send_log, email_send_state, suppressed_emails, email_unsubscribe_tokens

**NOTE**: eduforyou already has `referrals`, `student_gamification`, `student_cv`, `student_documents`, `skill_entries`, `social_profiles`, `outreach_templates`, `suppressed_emails`, `profiles`, and `quiz_results` tables. Check existing tables before creating new ones.

### Existing DB Functions
- `handle_new_user()` - auto-creates profile on signup
- `assign_default_student_role()` - auto-assigns 'student' role
- `has_role(user_id, role)` - role check helper
- `get_agent_leaderboard()` - top 10 agents by referrals
- Email queue functions: `enqueue_email()`, `read_email_batch()`, `delete_email()`, `move_to_dlq()`

### Existing Email System (ALREADY SOPHISTICATED - DO NOT REBUILD)
Uses pgmq with 4 queues (auth_emails, transactional_emails + DLQs). pg_cron job runs every 5s to invoke process-email-queue. Has rate limiting, TTL, dedup, suppression list. Uses Lovable email API.

### Existing Storage
- `documents` bucket (private) with RLS per user folder

### Existing Supabase Edge Functions
auth-email-hook, ceo-ai-engine, check-subscription, create-checkout, customer-portal, ikigai-builder, offer-builder, outreach-generator, process-email-queue, profile-builder, send-appointment-reminders, send-sms, send-transactional-email, skill-scanner, stripe-webhook

### Auth System
- Supabase Auth with user_roles table (admin, agent, student)
- AuthContext at src/contexts/AuthContext.tsx
- ProtectedRoute component with requiredRole prop
- Subscription checking via edge function

---

## WHAT NEEDS TO BE ADDED

Work through these phases in order. Each phase should be a separate commit.

### PHASE 1: Database Schema (Supabase Migrations)

IMPORTANT: Many tables already exist (see list above). Only create tables that are truly missing. Check first with `\dt` or by reading the types.ts file.

Create new Supabase migration files for these MISSING tables:

```sql
-- supabase/migrations/TIMESTAMP_phase1_missing_tables.sql

-- 1. deals (pipeline management) - NEW
CREATE TABLE IF NOT EXISTS deals (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contact_id UUID REFERENCES contacts(id),
  title TEXT NOT NULL,
  stage TEXT NOT NULL DEFAULT 'lead' CHECK (stage IN ('lead','discovery_call','eligibility_check','application','documents','finance_application','admission_test','enrolled','lost')),
  value NUMERIC DEFAULT 0,
  course_slug TEXT,
  campus_id UUID REFERENCES campuses(id),
  probability INT DEFAULT 10,
  expected_close_date TIMESTAMPTZ,
  lost_reason TEXT,
  assigned_to UUID,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 2. touchpoints (contact activity timeline)
CREATE TABLE IF NOT EXISTS touchpoints (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contact_id UUID NOT NULL REFERENCES contacts(id),
  type TEXT NOT NULL CHECK (type IN ('email','call','meeting','note','system','form','chat')),
  title TEXT,
  description TEXT,
  created_by UUID,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 3. documents - SKIP, already exists as student_documents
-- Just verify student_documents has all needed columns

-- 3b. touchpoints is NEW, documents table already exists. Next new table:
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  application_id UUID REFERENCES applications(id),
  name TEXT NOT NULL,
  type TEXT NOT NULL,
  file_url TEXT,
  file_name TEXT,
  mime_type TEXT,
  file_size INT,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending','submitted','verified','approved','rejected')),
  review_notes TEXT,
  reviewed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 4. referrals - SKIP, already exists. Verify columns match.
-- student_gamification also already exists - maps to gamification needs.
-- student_cv already exists - maps to CV builder needs.
-- skill_entries already exists.
-- quiz_results already exists.

-- Next truly NEW table:
-- 4b. edu_journey (DOES NOT EXIST YET)
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  referrer_id UUID NOT NULL,
  referred_name TEXT,
  referred_email TEXT NOT NULL,
  referred_phone TEXT,
  referred_user_id UUID,
  status TEXT DEFAULT 'pending' CHECK (status IN ('pending','contacted','enrolled','rejected')),
  bonus_amount NUMERIC DEFAULT 0,
  bonus_paid BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 5. edu_journey (student E.D.U. progress tracking)
CREATE TABLE IF NOT EXISTS edu_journey (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE,
  current_phase TEXT DEFAULT 'evaluate' CHECK (current_phase IN ('evaluate','deliver','unlock')),
  current_step INT DEFAULT 1,
  steps_completed JSONB DEFAULT '[]'::jsonb,
  eligibility_status TEXT,
  course_match_result JSONB,
  plan_generated BOOLEAN DEFAULT false,
  test_prep_score INT,
  documents_submitted INT DEFAULT 0,
  cv_completed BOOLEAN DEFAULT false,
  finance_status TEXT,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 6. gamification
CREATE TABLE IF NOT EXISTS gamification (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL UNIQUE,
  points INT DEFAULT 0,
  level INT DEFAULT 1,
  streak_days INT DEFAULT 0,
  badges JSONB DEFAULT '[]'::jsonb,
  last_activity_at TIMESTAMPTZ DEFAULT now(),
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 7. newsletter_subscribers
CREATE TABLE IF NOT EXISTS newsletter_subscribers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email TEXT NOT NULL UNIQUE,
  name TEXT,
  phone TEXT,
  source TEXT,
  lead_magnet TEXT,
  utm_source TEXT,
  utm_medium TEXT,
  utm_campaign TEXT,
  gdpr_consent BOOLEAN DEFAULT false,
  subscribed_at TIMESTAMPTZ DEFAULT now(),
  unsubscribed_at TIMESTAMPTZ
);

-- 8. email_sequences
CREATE TABLE IF NOT EXISTS email_sequences (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  trigger_event TEXT NOT NULL,
  is_active BOOLEAN DEFAULT true,
  steps JSONB DEFAULT '[]'::jsonb,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 9. sms_logs
CREATE TABLE IF NOT EXISTS sms_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  recipient_phone TEXT NOT NULL,
  message TEXT NOT NULL,
  status TEXT DEFAULT 'pending',
  provider_id TEXT,
  error_message TEXT,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 10. test_prep_attempts
CREATE TABLE IF NOT EXISTS test_prep_attempts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  course_slug TEXT NOT NULL,
  test_type TEXT NOT NULL CHECK (test_type IN ('written','interview','ai_personalized')),
  score INT,
  total_questions INT,
  answers JSONB,
  feedback JSONB,
  completed_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 11. contracts
CREATE TABLE IF NOT EXISTS contracts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  agent_id UUID NOT NULL,
  contract_type TEXT NOT NULL,
  status TEXT DEFAULT 'draft',
  signed_at TIMESTAMPTZ,
  terms JSONB,
  created_at TIMESTAMPTZ DEFAULT now(),
  updated_at TIMESTAMPTZ DEFAULT now()
);

-- 12. careers
CREATE TABLE IF NOT EXISTS careers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  title TEXT NOT NULL,
  slug TEXT NOT NULL UNIQUE,
  department TEXT,
  location TEXT,
  type TEXT DEFAULT 'full-time',
  description TEXT,
  requirements JSONB,
  is_active BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT now()
);

-- 13. career_applications
CREATE TABLE IF NOT EXISTS career_applications (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  career_id UUID REFERENCES careers(id),
  full_name TEXT NOT NULL,
  email TEXT NOT NULL,
  phone TEXT,
  cv_url TEXT,
  cover_letter TEXT,
  status TEXT DEFAULT 'new',
  created_at TIMESTAMPTZ DEFAULT now()
);

-- Enable RLS on all new tables
ALTER TABLE deals ENABLE ROW LEVEL SECURITY;
ALTER TABLE touchpoints ENABLE ROW LEVEL SECURITY;
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
ALTER TABLE referrals ENABLE ROW LEVEL SECURITY;
ALTER TABLE edu_journey ENABLE ROW LEVEL SECURITY;
ALTER TABLE gamification ENABLE ROW LEVEL SECURITY;
ALTER TABLE newsletter_subscribers ENABLE ROW LEVEL SECURITY;
ALTER TABLE email_sequences ENABLE ROW LEVEL SECURITY;
ALTER TABLE sms_logs ENABLE ROW LEVEL SECURITY;
ALTER TABLE test_prep_attempts ENABLE ROW LEVEL SECURITY;
ALTER TABLE contracts ENABLE ROW LEVEL SECURITY;
ALTER TABLE careers ENABLE ROW LEVEL SECURITY;
ALTER TABLE career_applications ENABLE ROW LEVEL SECURITY;

-- RLS Policies (basic - admin can do all, users can see own data)
-- Repeat this pattern for each table:
CREATE POLICY "Admin full access" ON deals FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Admin full access" ON touchpoints FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Admin full access" ON documents FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Users own documents" ON documents FOR SELECT USING (user_id = auth.uid());
CREATE POLICY "Users insert own documents" ON documents FOR INSERT WITH CHECK (user_id = auth.uid());
CREATE POLICY "Admin full access" ON referrals FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Users own referrals" ON referrals FOR SELECT USING (referrer_id = auth.uid());
CREATE POLICY "Admin full access" ON edu_journey FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Users own journey" ON edu_journey FOR ALL USING (user_id = auth.uid());
CREATE POLICY "Admin full access" ON gamification FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Users own gamification" ON gamification FOR SELECT USING (user_id = auth.uid());
CREATE POLICY "Public newsletter subscribe" ON newsletter_subscribers FOR INSERT WITH CHECK (true);
CREATE POLICY "Admin full access" ON newsletter_subscribers FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Admin full access" ON email_sequences FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Admin full access" ON sms_logs FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Admin full access" ON test_prep_attempts FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Users own test attempts" ON test_prep_attempts FOR ALL USING (user_id = auth.uid());
CREATE POLICY "Admin full access" ON contracts FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Public careers view" ON careers FOR SELECT USING (is_active = true);
CREATE POLICY "Admin full access" ON careers FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
CREATE POLICY "Public career applications" ON career_applications FOR INSERT WITH CHECK (true);
CREATE POLICY "Admin full access" ON career_applications FOR ALL USING (
  EXISTS (SELECT 1 FROM user_roles WHERE user_id = auth.uid() AND role = 'admin')
);
```

After creating the migration, update `src/integrations/supabase/types.ts` to include the new table types.

---

### PHASE 2: Auth System Expansion

Currently there's only a single `/login` page. Create these pages:

1. **src/pages/auth/Login.tsx** - Proper login with email/password using Supabase Auth
   - Email + password form
   - "Forgot password?" link
   - Link to register
   - After login, redirect based on role: admin→/admin, agent→/agent/dashboard, student→/student/dashboard

2. **src/pages/auth/Register.tsx** - Registration page
   - Full name, email, phone, password
   - GDPR consent checkbox
   - Creates Supabase auth user + assigns 'student' role by default
   - Fires welcome email via send-transactional-email edge function

3. **src/pages/auth/ForgotPassword.tsx** - Password reset request
   - Email input
   - Calls supabase.auth.resetPasswordForEmail()

4. **src/pages/auth/ResetPassword.tsx** - New password form
   - New password + confirm
   - Calls supabase.auth.updateUser()

5. **src/pages/auth/AgentLogin.tsx** - Dedicated agent login
   - Same as login but styled differently, redirects to /agent/dashboard

Update App.tsx routes:
```tsx
<Route path="/auth/login" element={<AuthLogin />} />
<Route path="/auth/register" element={<AuthRegister />} />
<Route path="/forgot-password" element={<ForgotPassword />} />
<Route path="/reset-password" element={<ResetPassword />} />
<Route path="/agent/login" element={<AgentLogin />} />
```

---

### PHASE 3: Public Pages (Missing from highticket)

Create these pages that exist in highticket but NOT in eduforyou:

1. **src/pages/WhyFree.tsx** - Explains the free education model (SFE-funded). Currently `/why-free` just shows the generic About page.

2. **src/pages/Team.tsx** - Team members page. Currently `/team` shows generic About.

3. **src/pages/Partners.tsx** - University partners showcase. Currently `/partners` shows generic About.

4. **src/pages/AgentsOffer.tsx** - Sales page for becoming a recruitment agent
   - Commission tiers: Starter £500, Pro £750, Elite £1000 per enrollment
   - Milestone bonuses: 25 students = Dubai Holiday, 100 = BMW, 200 = Franchise opportunity
   - Registration CTA

5. **src/pages/AgentsSuccess.tsx** - Agent success stories + testimonials

6. **src/pages/webinar/WebinarList.tsx** - List of all available webinars
7. **src/pages/webinar/WebinarUniversity.tsx** - University-focused webinar registration
8. **src/pages/webinar/WebinarAgents.tsx** - Agent recruitment webinar
9. **src/pages/webinar/WebinarIkigai.tsx** - Ikigai-themed webinar with evergreen 48h countdown

10. **src/pages/ProfessionalCourses.tsx** - Professional/short courses catalog
11. **src/pages/ProfessionalCourseDetail.tsx** - Individual professional course detail

12. **src/pages/TestimonialsHub.tsx** - Central testimonials hub with video testimonials
13. **src/pages/SuccessStories.tsx** - Student success stories

14. **src/pages/BookLanding.tsx** - Book product landing page
15. **src/pages/AudiobookUpsell.tsx** - Audiobook upsell after ebook purchase
16. **src/pages/ThankYouEbook.tsx** - Post-purchase thank you
17. **src/pages/BlackbookSales.tsx** - "Blackbook" ebook sales landing

18. **src/pages/RoLanding.tsx** - Romanian-specific landing page variant
19. **src/pages/LeadMagnet.tsx** - Configurable lead magnet download page (accepts slug param for different guides: ghid-gratuit, ghid-finantare, ghid-transformare)

20. **src/pages/CareerApply.tsx** - Career application form with CV upload

Update App.tsx with all new routes. Use Romanian URL paths where appropriate (matching highticket):
```tsx
<Route path="/about/why-free" element={<WhyFree />} />
<Route path="/about/team" element={<Team />} />
<Route path="/about/partners" element={<Partners />} />
<Route path="/agents/offer" element={<AgentsOffer />} />
<Route path="/agents/success" element={<AgentsSuccess />} />
<Route path="/webinar" element={<WebinarList />} />
<Route path="/webinar/university" element={<WebinarUniversity />} />
<Route path="/webinar/agents" element={<WebinarAgents />} />
<Route path="/webinar/ikigai" element={<WebinarIkigai />} />
<Route path="/cursuri-profesionale" element={<ProfessionalCourses />} />
<Route path="/cursuri-profesionale/:slug" element={<ProfessionalCourseDetail />} />
<Route path="/testimoniale" element={<TestimonialsHub />} />
<Route path="/success-stories" element={<SuccessStories />} />
<Route path="/carte" element={<BookLanding />} />
<Route path="/book" element={<BookLanding />} />
<Route path="/upsell-audiobook" element={<AudiobookUpsell />} />
<Route path="/thank-you-ebook" element={<ThankYouEbook />} />
<Route path="/sfe-blackbook" element={<BlackbookSales />} />
<Route path="/ro" element={<RoLanding />} />
<Route path="/ghid-gratuit" element={<LeadMagnet />} />
<Route path="/ghid-finantare" element={<LeadMagnet />} />
<Route path="/ghid-transformare" element={<LeadMagnet />} />
<Route path="/careers/:slug/apply" element={<CareerApply />} />
```

---

### PHASE 4: CEO Dashboard Expansion (19 sub-pages)

The current CeoDashboard is a single monolithic page. Break it into 19 separate pages with a sidebar layout.

Create **src/components/CeoDashboardLayout.tsx**:
- Fixed 264px dark navy sidebar (#0a1628) with gold accents (#d4a843)
- Brain icon logo
- Navigation items with active state highlighting
- "Back to Admin" and "Back to Site" footer links
- Main content area with warm off-white (#f8f7f4) background

Create these CEO pages in **src/pages/ceo/**:

1. **CeoDashboard.tsx** - Executive command center
   - 5 KPI cards (Total Students, Active Applications, Revenue, Conversion Rate, Agent Performance)
   - Student Lifecycle Pipeline visualization: Lead → MQL → SQL → Applicant → Enrolled → Student
   - OKR progress tracker (reads from ceo_okrs table)
   - AI Morning Report generator (call ceo-ai-engine edge function)
   - Alerts panel
   - AI Agent Status grid
   - Recent Activity feed

2. **CeoSales.tsx** - Sales pipeline and performance metrics
3. **CeoTasks.tsx** - Task management (uses ceo_tasks table)
4. **CeoAnalytics.tsx** - Business analytics dashboard with recharts
5. **CeoOkrs.tsx** - OKR management (CRUD on ceo_okrs table)
6. **CeoAgents.tsx** - Agent roster and performance management
7. **CeoContentStudio.tsx** - AI-assisted content creation
8. **CeoCmoReview.tsx** - Marketing content review/approval queue
9. **CeoContentCalendar.tsx** - Content publishing calendar
10. **CeoSettings.tsx** - Platform settings
11. **CeoQuality.tsx** - Quality assurance metrics
12. **CeoAiChat.tsx** - CEO AI chat assistant (uses ceo-ai-engine edge function)
13. **CeoHr.tsx** - HR dashboard
14. **CeoSuccess.tsx** - Student success metrics
15. **CeoCmo.tsx** - CMO/marketing overview
16. **CeoCandidates.tsx** - HR candidate pipeline
17. **CeoAgentPlatform.tsx** - Agent platform monitoring
18. **CeoWorkflows.tsx** - Workflow automation management
19. **CeoApprovals.tsx** - Approval queue for pending decisions

Update App.tsx:
```tsx
<Route path="/ceo" element={<ProtectedRoute requiredRole="admin"><CeoDashboardLayout /></ProtectedRoute>}>
  <Route index element={<CeoDashboard />} />
  <Route path="sales" element={<CeoSales />} />
  <Route path="tasks" element={<CeoTasks />} />
  <Route path="analytics" element={<CeoAnalytics />} />
  <Route path="okrs" element={<CeoOkrs />} />
  <Route path="agents" element={<CeoAgents />} />
  <Route path="content" element={<CeoContentStudio />} />
  <Route path="cmo-review" element={<CeoCmoReview />} />
  <Route path="calendar" element={<CeoContentCalendar />} />
  <Route path="settings" element={<CeoSettings />} />
  <Route path="quality" element={<CeoQuality />} />
  <Route path="ai-chat" element={<CeoAiChat />} />
  <Route path="hr" element={<CeoHr />} />
  <Route path="success" element={<CeoSuccess />} />
  <Route path="cmo" element={<CeoCmo />} />
  <Route path="candidates" element={<CeoCandidates />} />
  <Route path="agent-platform" element={<CeoAgentPlatform />} />
  <Route path="workflows" element={<CeoWorkflows />} />
  <Route path="approvals" element={<CeoApprovals />} />
</Route>
```

---

### PHASE 5: Admin Dashboard Expansion (17 sub-pages)

Current AdminDashboard is monolithic with some tab components. Break into separate pages.

Create these admin pages in **src/pages/admin/**:

1. **AdminDashboard.tsx** - Overview with 8 KPI cards, pipeline chart (recharts), product sales breakdown, recent contacts, quick actions
2. **AdminContacts.tsx** - Contact CRM with search/filter (already exists, verify working)
3. **AdminPipeline.tsx** - Sales pipeline with deal stages (already exists, verify)
4. **AdminCourses.tsx** - Course catalog CRUD management
5. **AdminFunnel.tsx** - Funnel analytics with conversion charts
6. **AdminEligibilityStats.tsx** - Eligibility quiz analytics
7. **AdminHr.tsx** - HR/employee management
8. **AdminStudents.tsx** - Comprehensive student management (~500+ lines, filterable, with status badges)
9. **AdminUsers.tsx** - User/role management (already exists, verify)
10. **AdminContracts.tsx** - Agent contract management (uses contracts table)
11. **AdminCampuses.tsx** - Campus management (promote from tab to page)
12. **AdminAppointments.tsx** - Appointment management (promote from tab to page)
13. **AdminSMS.tsx** - SMS sending and logging (promote from tab, uses sms_logs table)
14. **AdminAbandonedCarts.tsx** - Abandoned cart recovery (promote from tab)
15. **AdminEmailSequences.tsx** - Email drip sequence builder (uses email_sequences table)
16. **AdminEmailTemplates.tsx** - Email template editor (already exists, verify)
17. **AdminStudentCRM.tsx** - Full student CRM with deals, touchpoints, documents

Update App.tsx admin routing to use nested routes with DashboardLayout.

---

### PHASE 6: Student E.D.U. Journey System (27 pages)

This is the BIGGEST missing feature. The E.D.U. Method (Evaluate → Deliver → Unlock) with 11 tracked steps.

Create **src/hooks/useEduJourney.ts**:
```typescript
// Hook that reads/writes edu_journey table via Supabase
// Returns: currentPhase, currentStep, stepsCompleted, updateStep(), completeStep()
// Phases: evaluate (steps 1-4), deliver (steps 5-8), unlock (steps 9-11)
```

Create **src/hooks/useGamification.ts**:
```typescript
// Hook that reads/writes gamification table via Supabase
// Returns: points, level, streak, badges, addPoints(), awardBadge()
```

Create **src/components/edu/EDUJourneyTracker.tsx**:
- Visual 3-phase progress tracker
- 11 steps with completion status (completed/in_progress/pending)
- Each step links to its page route
- Phase progress bar with animated badges

Create **src/components/edu/EDUPhaseBadge.tsx**:
- Badge component for E/D/U phases
- Colors: blue-cyan (Evaluate), orange-purple (Deliver), green-lime (Unlock)
- Pulse animation on active phase, checkmark on completed

Create **src/components/gamification/GamificationBar.tsx**:
- Shows level, points, streak, badges
- Dark gradient card with orange accents

Create student pages in **src/pages/student/**:

#### Core Student Pages
1. **EDUDashboard.tsx** - Main student dashboard (~800+ lines)
   - E.D.U. pipeline visualization with 3 phases
   - 11 tracked steps with progress
   - GamificationBar
   - Next-step guidance
   - Consultation booking
   - Messages inbox
   - Quick stats
   - AI-recommended courses

2. **StudentDocuments.tsx** - Document upload/management (uses documents table + Supabase Storage)
3. **StudentFinance.tsx** - Finance application tracking
4. **StudentPreparation.tsx** - Exam preparation guide
5. **StudentProfile.tsx** - Profile management
6. **StudentReferral.tsx** - Referral program (uses referrals table)
7. **StudentMessages.tsx** - Admin-student messaging (uses messages table)
8. **SkillScanner.tsx** - AI skill assessment (calls skill-scanner edge function)
9. **IkigaiApply.tsx** - Apply based on Ikigai quiz results

#### E.D.U. Pipeline Steps (src/pages/student/edu/)
10. **EligibilityCheck.tsx** - Step E1: Eligibility verification form
11. **CourseMatch.tsx** - Step E2: AI course matching based on Ikigai + eligibility
12. **EduPlan.tsx** - Step E3: Personalized education plan generation
13. **TestPrep.tsx** - Step E4: Test preparation hub
14. **Documents.tsx** - Step D1: Document collection/upload
15. **CVBuilder.tsx** - Step D2: AI-assisted CV/personal statement builder
16. **DocumentChecks.tsx** - Step D3/D4: University response tracking
17. **StudentFinanceEdu.tsx** - Step U1: SFE application guidance
18. **Bonuses.tsx** - Step U2: Platform bonuses/rewards
19. **FreedomCircle.tsx** - Step U3/U4: Alumni community

#### Test Prep (src/pages/student/test-prep/)
20. **AIPersonalizedTest.tsx** - AI-generated personalized practice tests (uses test_prep_attempts table)
21. **WrittenTest.tsx** - Written admission test practice
22. **InterviewPractice.tsx** - Mock interview with AI
23. **TestResults.tsx** - Test results and analysis

#### Freedom Launchpad Wizard (src/pages/wizard/)
24. **FreedomLaunchpad.tsx** - Post-graduation career launchpad hub
25. **GigJobBuilder.tsx** - Gig/freelance job listing builder
(DefineYourPath, OfferBuilder, ProfileBuilder, OutreachGenerator, FreedomPlanExport already exist)

---

### PHASE 7: Agent Dashboard Expansion (10 pages)

Create agent pages in **src/pages/agent/**:

1. **AgentDashboard.tsx** - Command center with:
   - Commission tiers display (Starter £500, Pro £750, Elite £1000)
   - Milestone bonuses (25=Dubai, 100=BMW, 200=Franchise)
   - KPI cards (total referrals, enrolled, earnings)
   - Student referral form
   - Membership perks

2. **AgentStudents.tsx** - Agent's referred students list with status tracking
3. **AgentCommissions.tsx** - Commission tracking and payout history (already exists, verify)
4. **AgentLeaderboard.tsx** - Agent rankings (already exists, verify)
5. **AgentMaterials.tsx** - Sales materials and resources download center
6. **AgentVipCourses.tsx** - VIP/premium course access
7. **AgentCourseIncome.tsx** - Income breakdown by course
8. **AgentCourseVideouriAI.tsx** - AI-generated course video content
9. **AgentCourseInfo.tsx** - Course information for pitching
10. **AgentProfile.tsx** - Profile settings (already exists, verify)

---

### PHASE 8: Missing Components

Create these components:

1. **src/components/ErrorBoundary.tsx** - React error boundary with retry
2. **src/components/AIChatBox.tsx** - Reusable AI chat interface
   - Props: messages[], onSend callback, loading state, suggestedPrompts[]
   - Markdown rendering for assistant messages
   - Auto-scroll, Enter to send, Shift+Enter for newline

3. **src/components/EligibilityFlowWidget.tsx** - 4-step eligibility wizard (~800 lines)
   - Step 1: Eligibility (financing history, qualification, personal info, age checks)
   - Step 2: Course selection (filtered by eligibility level)
   - Step 3: Campus selection
   - Step 4: Account creation + apply
   - Saves lead to contacts table at step 1
   - Romanian translations inline
   - GTM dataLayer events

4. **src/components/GlobalSocialProof.tsx** - Rotating social proof notifications
   - 15 Romanian names with UK cities
   - 6 action types (applied, enrolled, received financing, etc.)
   - Shows every 25-50 seconds, auto-hides after 5 seconds
   - Fixed bottom-left, slide-up animation
   - Dismissible

5. **src/components/ExitIntentPopupGlobal.tsx** - Exit-intent modal
   - Desktop: triggered by mouse leaving viewport top
   - Mobile: triggered by 45s inactivity
   - Shows once per session (sessionStorage)
   - CTA to /ikigai
   - 3 benefit bullets, trust signals

6. **src/components/SEO.tsx** - Comprehensive SEO component
   - Uses react-helmet-async (add to dependencies)
   - Title, description, canonical, hreflang (en/ro), Open Graph, Twitter Cards
   - JSON-LD structured data helpers: organization, FAQ, product, local business, article
   - Breadcrumbs support

7. **src/components/WebinarAnnouncementBanner.tsx** - Fixed top banner for IKIGAI webinar promotion
8. **src/components/NewsletterForm.tsx** - Email capture with 4 variants (dark/light/inline/card)
   - GDPR checkbox, UTM capture, saves to newsletter_subscribers table
9. **src/components/VideoTestimonialsCarousel.tsx** - 16 YouTube video testimonials
   - Domain filtering (All/Business/Technology/Health/Construction)
   - Stats bar (6288+ students, 31M+ funding, 4.8/5 Trustpilot, 94% success)
10. **src/components/AgentContractModal.tsx** - Agent contract signing modal
11. **src/components/ApplyToCourseModal.tsx** - Course application modal
12. **src/components/HomeEligibilityForm.tsx** - Simplified home page eligibility form
13. **src/components/HomeFAQ.tsx** - FAQ accordion for homepage
14. **src/components/Map.tsx** - Campus map component
15. **src/components/ReferralCard.tsx** - Referral display card
16. **src/components/UniversityPartnersBanner.tsx** - Partner logos banner
17. **src/components/WaitlistForm.tsx** - Waitlist signup form
18. **src/components/NoIndexMeta.tsx** - noindex meta for internal pages

#### Webinar Components (src/components/webinar/)
19. **AnimatedCounter.tsx** - IntersectionObserver count-up animation
20. **CountdownTimer.tsx** - Countdown display
21. **IkigaiCountdownTimer.tsx** - Evergreen 48h countdown (localStorage anchor)
22. **IkigaiSignupForm.tsx** - Webinar signup form (saves to contacts table)
23. **ScrollReveal.tsx** - Scroll-triggered animations
24. **SocialProofToast.tsx** - Webinar-specific social proof toasts
25. **UrgencyBanner.tsx** - "Spots remaining" fake scarcity banner
26. **TestimonialsCarousel.tsx** - Dark-themed testimonials for webinar pages
27. **ExitIntentPopup.tsx** - Webinar-specific exit intent
28. **ExitIntentPopupAgents.tsx** - Agent-focused exit intent

---

### PHASE 9: i18n Expansion

Currently there's a single `src/i18n/translations.ts`. Expand to 4 languages matching highticket:

Create:
- **src/i18n/locales/ro.ts** - Romanian (primary)
- **src/i18n/locales/en.ts** - English
- **src/i18n/locales/hu.ts** - Hungarian
- **src/i18n/locales/pl.ts** - Polish

Update **src/i18n/LanguageContext.tsx** to support 4 languages with localStorage persistence.

---

### PHASE 10: Data & Lib Files

1. **src/lib/googleAnalytics.ts** - GA4 initialization and event helpers
2. **src/lib/metaPixel.ts** - Meta Pixel initialization and event helpers (Lead, Purchase, etc.)
3. **src/lib/session.ts** - Session token management (localStorage)
4. **src/lib/blogArticles.ts** - Blog article data/content
5. **src/lib/professionalCourses.ts** - Professional courses data
6. **src/lib/adminNav.tsx** - Admin sidebar navigation config
7. **src/lib/ceoNav.tsx** - CEO sidebar navigation config

---

### PHASE 11: App.tsx Final Update

Wrap the app with additional providers:
```tsx
import { HelmetProvider } from "react-helmet-async";
import ErrorBoundary from "@/components/ErrorBoundary";

// In the App component, add:
// - ErrorBoundary as outermost wrapper
// - HelmetProvider
// - Conditionally show/hide Header/Footer for dashboard pages
// - Show GlobalSocialProof and ExitIntentPopupGlobal on public (non-dashboard) pages
// - Show WebinarAnnouncementBanner on non-webinar public pages
```

Add these dependencies to package.json:
```
react-helmet-async
```

---

## IMPORTANT RULES

1. **Use Supabase, NOT tRPC** - All data fetching should use `supabase.from('table').select()` with @tanstack/react-query. No tRPC.
2. **Use react-router-dom, NOT wouter** - Keep the existing router.
3. **Use Tailwind v3 syntax** - No v4 features. Use `className` not CSS-in-JS.
4. **Keep shadcn/ui pattern** - Use existing ui components from src/components/ui/
5. **Use Supabase Auth** - Not custom JWT. The AuthContext already handles this.
6. **Edge Functions for server logic** - Any server-side logic goes in supabase/functions/
7. **Romanian as primary language** - UI text should be in Romanian by default with i18n support.
8. **Match highticket's visual design** - Warm tones (#E67E22 orange, #1a252f dark), serif headings for marketing pages. Dark navy (#0a1628) + gold (#d4a843) for CEO dashboard.
9. **Lazy load all pages** - Use React.lazy() + Suspense for code splitting on every page.
10. **Mobile-first responsive design** - All pages must work on mobile.

## Priority Order

If you can't do everything at once, prioritize:
1. Phase 1 (Database) → Phase 2 (Auth) → Phase 6 (Student E.D.U.) → Phase 4 (CEO) → Phase 5 (Admin) → Phase 7 (Agent) → Phase 3 (Public pages) → Phase 8 (Components) → Phase 9-11 (Polish)

Start now. Work through each phase systematically, committing after each phase.
