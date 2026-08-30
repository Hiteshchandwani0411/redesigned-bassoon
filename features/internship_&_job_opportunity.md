# Intelligent Internship & Job Marketplace Architecture

The **Internship and Job Marketplace** converts traditional transactional application boards into a closed-loop, pre-filtered talent pipeline. Utilizing vector-based skill matching, atomic database operations, and real-time state synchronization, the marketplace ensures recruiters receive only verified, role-compatible candidates.

---

## 1. End-to-End System Workflow

```text
[ Phase 1: HR Posting ] ────> [ MongoDB ] (Stores Job & Vector Schemas)
                                  │
                                  ▼
[ Phase 2: AI Matching ] ───> [ Python Microservice ] ───> Calculates Cosine Similarity Score (%)
                                  │
                                  ▼
[ Phase 3: Student Feed ] ──> [ React Client ] ───────────> Dynamic Gating Logic:
                                                            ├── Match Score >= 70%: Unlocks "Apply" (Atomic Transaction)
                                                            └── Match Score < 70%: Unlocks "View Skill Gap & Roadmap"
                                  │
                                  ▼
[ Phase 4: Recruitment ] ───> [ HR Kanban ATS ] ───────────> Real-Time Socket Event ──> Updates TPO & Student Dashboards

```

---

## 2. Technical Phase Breakdown

### Phase 1: Industry Job Creation

* **Authentication:** Recruiters authenticate via JWT-secured routes.
* **Structured Skill Inputs:** Job roles do not rely solely on unstructured text descriptions. Form interfaces mandate selecting discrete micro-skill tags from a validated taxonomy.
* **Database Schema:**

```json
{
  "job_id": "job_505",
  "company": "TechCorp",
  "role": "Backend Intern",
  "required_vectors": ["express-js", "mongodb-aggregation", "rest-api"],
  "applicants_queue": [],
  "status": "Active"
}

```

### Phase 2: Recommendation & Vector Matching Engine

* **Execution:** On job creation, the Node.js core server pushes `required_vectors` to the Python microservice.
* **Similarity Calculation:** The microservice executes Cosine Similarity comparisons between job requirements and verified candidate skill vectors stored in MongoDB.
* **Caching:** Resulting compatibility scores are cached in Redis to populate real-time candidate and job feeds instantly.

### Phase 3: Candidate Application & Algorithmic Gating

* **Personalized Feed:** Candidates view opportunities pre-sorted by their specific match percentage.
* **Algorithmic Application Gate:**
* **Score < Threshold (e.g., 70%):** The "Apply" action is disabled. The client renders a "View Skill Gap & Roadmap" CTA redirecting the candidate to targeted skill development modules.
* **Score $\ge$ Threshold:** Unlocks application capability.


* **Atomic Transaction Execution:** Submitting an application triggers a two-way atomic transaction in MongoDB:
1. Appends `student_id` to the job's `applicants_queue`.
2. Appends `job_id` to the candidate's `applied_opportunities` array with status `Applied`.



### Phase 4: Applicant Tracking System (ATS) & Real-Time Sync

* **HR Candidate Pipeline:** Applicants are displayed on a Kanban board categorized by status (`Applied`, `Shortlisted`, `Interview`, `Hired`) and sorted by verified proof-of-work scores.
* **Drag-and-Drop State Sync:** State transitions trigger WebSockets / SSE events, broadcasting real-time status updates directly to both candidate and institutional TPO dashboards.

---

## 3. Recommended Frontend Integration for ATS

For the HR Kanban Board UI, **`@hello-pangea/dnd`** (the actively maintained community fork of `react-beautiful-dnd`) or **`@dnd-kit/core`** are the recommended libraries to handle drag-and-drop state updates efficiently within React components.

---

## 4. Pitch Script

> *"Judges, most internship portals act like unorganized notice boards where students spam resumes to hundreds of companies, wasting HR's time. We engineered a closed-loop, intelligent marketplace.*
> *Step 1: When an industry posts an opportunity, they define mandatory micro-skill vectors. Step 2: Our Python microservice instantly calculates the Cosine Similarity between this job requirement and our students' verified digital portfolios. Step 3: The student receives a highly personalized feed. To completely eliminate application spam, the 'Apply' button is algorithmically gated. If a student's match score is below 70%, they cannot apply; they are redirected to a customized learning roadmap to earn those missing skills first.*
> *Finally, when a qualified student applies, our Express backend handles the application as a secure, two-way transaction, updating both the student's tracker and the HR's internal Applicant Tracking System in real-time. We deliver exactly what companies want: a pre-filtered, verified talent pipeline."*