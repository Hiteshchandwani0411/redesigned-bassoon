# Dynamic Skill Assessment Engine Architecture

The **Dynamic Skill Assessment Engine** combines passive code profiling, pre-computed adaptive testing, and strict anti-cheat mechanisms. This blueprint outlines the MERN-stack architecture and presentation narrative for the platform's core assessment workflow.

---

[ Admin / Industry Partner ]                    [ Student Registration ]
        │ defines role → subtopic list                     │
        ▼                                                  ▼
[ LLM Microservice ]                          [ Link GitHub/LeetCode? ] (optional)
   generates BOUNDED tree per subtopic                     │
   (depth=3, branch=2) per role                    ┌────yes┴─────no─────┐
        │                                          ▼                    ▼
[ Human Review / Approval ]              [ Redis Job Queue ]      [ profile_linked:
   admin edits & approves                  (BullMQ, rate-limited)    false flag set ]
        │                                          │                    │
        ▼                                          ▼                    │
[ MongoDB: assessment_trees ]            [ 3rd-Party APIs ]             │
  versioned, immutable, per subtopic       (GitHub, LeetCode)           │
        │                                          │                    │
        │                                          ▼                    │
        │                              [ Repo Signal Extractor ]        │
        │                               - dependency scan (regex)       │
        │                               - jwt.verify() usage check      │
        │                               - commit recency                │
        │                               → per-subtopic START TIER       │
        │                                 (Medium, or Medium-Hard       │
        │                                  if strong evidence)          │
        │                                          │                    │
        └──────────────────┬───────────────────────┴────────────────────┘
                            ▼
              [ Node/Express Server — SESSION OWNER ]
                  (server holds current tier, timing,
                   next-question logic — client never decides)
                            │
                ┌───────────┼────────────────────┐
                ▼           ▼                     ▼
      [ React Frontend ] [ Soft-Skill Module ]  [ Tab-Visibility Logger ]
       renders question   fixed rubric-scored     (soft signal only —
       server sends ONLY   questionnaire            logs count/duration,
       current node,        (Communication,          flags for HUMAN
       never full tree      Teamwork, etc.)          review, no auto-fail)
            │                    │                     │
            ▼                    │                     │
  [ Staircase Adaptive Loop ]    │                     │
   per subtopic:                 │                     │
   correct → tier UP             │                     │
   wrong   → tier DOWN           │                     │
   stabilize 2x @ tier → LOCK    │                     │
   max 4 Qs/subtopic             │                     │
            │                    │                     │
            └─────────┬──────────┘                     │
                       ▼                               │
        [ Per-Subtopic Proficiency Bands ]             │
         {Node.js: Advanced, DB: Beginner, ...}        │
                       │                               │
                       ▼                               ▼
         [ Weighted Technical Score ]          [ proctorFlags stored
          RENORMALIZED across only               alongside, informational ]
          available signals (github/
          leetcode/assessment) — no
          fixed 50/20/30 if a signal
          is missing
                       │
                       ▼
         [ Python Embedding Microservice ]
          Sentence-BERT + Cosine Similarity
          (same engine as Skill Mapping /
           Course Matching — reused, not
           duplicated set-subtraction)
                       │
                       ▼
         [ Skill Gap Array — BY SEVERITY ]
          { skill, required_tier, achieved_tier, severity }
          not just present/absent
                       │
                       ▼
         [ Verified Digital Portfolio ]
          versioned, expiresAt: +6 months
          → feeds Course/Job/Industry
            recommendation engine (shared contract)

---

## 2. Technical Implementation Roadmap

### Phase O: Passive Profiling (Background Verification)

1. **OAuth Integration:** Utilizes GitHub OAuth to acquire read-only access tokens, resolving access limitations for private repositories.
2. **Asynchronous Job Queuing:** Pushes profile sync tasks to a Redis queue (`BullMQ`) to manage rate limits and maintain system throughput during high-concurrency registration windows.
3. **Repository Parsing Engine:** Scans authenticated user repositories to detect framework implementations (e.g., Express routing, template rendering, nested MongoDB queries) and populates baseline skill tags automatically. Queries competitive programming APIs to log problem-solving streaks and verified logical capabilities.

Here's how I'd actually build this — as a working system, not a pitch. I'll go phase by phase, and at each point I'll flag *why* I'm deviating from the original doc where it matters.

---

---

## Phase A — Registration & Passive Profiling

```
Student registers
   → account created in MongoDB (users collection)
   → student prompted: "Link GitHub" / "Link LeetCode" (optional, not blocking)
   → OAuth redirect (GitHub: read-only repo scope)
   → on callback: store encrypted access token, enqueue sync job
```

**Job queue design (this part of the original doc is genuinely right):**

```javascript
// producer — on OAuth callback
await syncQueue.add('github-sync', { userId, provider: 'github' }, {
  attempts: 3,
  backoff: { type: 'exponential', delay: 5000 },
  removeOnComplete: true,
});
```

Why BullMQ + Redis here specifically: GitHub's API rate limit is 5,000 req/hr per token — fine per-user, but if 500 students register during a college's onboarding window, you need queued, rate-aware, retryable workers, not synchronous calls blocking the registration request. That's a correct architectural instinct from the original doc; I'd keep it as-is.

**What the parser actually extracts** — and here I'd scope it down from "detects framework implementations via repository parsing" to something achievable in a hackathon:

- Repo language breakdown (GitHub API gives this directly — no parsing needed)
- Dependency manifests (`package.json`, `requirements.txt`, `pom.xml`) — regex/JSON-parse for known framework names (Express, Django, Spring) — cheap, reliable, no AST needed
- Commit frequency/recency (signal of active vs. dormant profile)
- LeetCode: no official public API exists — I'd use a known community GraphQL endpoint pattern (fragile, can break) and explicitly flag this as an external dependency risk in your docs, with a manual "paste your LeetCode profile stats" fallback if the sync fails.

**Critical addition the original doc misses: what happens if a student has no GitHub/LeetCode at all?** Set a `profile_linked: boolean` flag per source. This flag is what makes Phase E's scoring actually work for everyone — flagging it now because it's a data-model decision, not just a scoring-time patch.

---

## Phase B — Building the Question Bank (offline, admin-time)

This runs **once**, when an industry partner or admin defines a role template — not per student, not per session.

```
Admin defines target role: "Backend Developer" → skill list: [Node.js, MongoDB, REST APIs, Auth]
   ↓
LLM generates a BOUNDED tree: depth = 3, branching = 2
   (~2^3 = 8 leaf paths per skill — not thousands)
   ↓
Human review step (admin approves/edits generated questions before publishing)
   ↓
Stored in MongoDB as a versioned, immutable document
```

```javascript
// assessment_trees collection
{
  _id: "tree_backend_dev_v1",
  role: "Backend Developer",
  skillTags: ["Node.js", "MongoDB", "REST APIs", "Auth"],
  nodes: {
    "root": { question: "...", options: [...], skillTag: "Node.js", difficulty: 2,
              next: { "correct": "n2a", "incorrect": "n2b" } },
    "n2a": { ... },
    // depth capped at 3, branching capped at 2 — deliberate, stated limit
  },
  version: 1,
  approvedBy: "admin_id",
  createdAt: ...
}
```

Depth/branching caps aren't a limitation to hide — they're a design decision you state out loud: *"our adaptive engine is a bounded decision tree, not full Item Response Theory CAT — appropriate for a 15-minute assessment, and it means zero LLM calls happen during the live test."* That sentence, said plainly, is more credible to a judge than an unbounded claim that won't hold up under a follow-up question.

---

## Phase C — The Soft-Skill Module (this was missing entirely, and it's required by the PS)

Doesn't need adaptivity — a fixed, short, scenario-based questionnaire scored against a rubric:

```javascript
// soft_skill_questions collection
{
  competency: "Communication",
  scenario: "A teammate disagrees with your approach mid-sprint. You...",
  options: [
    { text: "Explain reasoning, ask for their concerns", score: 5 },
    { text: "Insist on your approach since deadline is close", score: 2 },
    { text: "Avoid the conflict, quietly redo it their way", score: 1 },
  ]
}
```

Map 4–5 competencies (Communication, Teamwork, Adaptability, Problem-Solving Approach) to the PS's *"technical and soft skills"* line directly. This is deliberately simple — deterministic, rubric-scored, explainable to a judge in one sentence, and it closes the actual gap in the PS requirement.

---

## Phase D — The Live Assessment Runtime

```
POST /assessment/start  { studentId, treeId }
   → server creates session doc: { sessionId, treeId, currentNodeId: "root",
                                    startedAt, answers: [], deadlineAt: now+45s }
   → server returns ONLY the current node's question+options (never the tree)

Client renders question, shows a countdown mirroring deadlineAt (display only)

POST /assessment/answer  { sessionId, nodeId, selectedOptionId }
   → server checks: now <= session.deadlineAt (+ small grace, e.g. 2s network buffer)
       → if late: mark node as "timeout", treat as incorrect, proceed anyway
   → server looks up next node from the TREE (not from client), using
       tree.nodes[nodeId].next[correct ? "correct" : "incorrect"]
   → server appends { nodeId, selectedOptionId, correct, latencyMs } to session.answers
   → server updates session.currentNodeId, new deadlineAt
   → returns next question
```

**On tab-switch (`document.visibilityState`):** log it as a soft signal, don't hard-terminate. A hard-terminate rule is punishing legitimate behavior (glancing at a calculator app, a notification popup, a flaky window manager) as hard as actual cheating, and it's trivially bypassable anyway (second monitor). Better: log count + duration per switch, surface it to the recruiter/institution as a "reviewed by human" flag if it crosses a threshold (e.g., 3+ switches or any switch >30s) — a signal for a human to look at, not an automatic verdict.

**Webcam capture:** I'd cut it from the MVP outright, per what I flagged earlier — it needs a baseline reference photo to mean anything, real consent flow under DPDP Act 2023, and a retention/deletion policy, none of which is a good use of hackathon time relative to what it buys you.

---

## Phase E — Scoring, With the Fallback the Original Design Was Missing

```javascript
function computeTechnicalScore(student) {
  const weights = { github: 0.5, leetcode: 0.2, assessment: 0.3 };
  const available = {};
  if (student.profileLinked.github)   available.github = student.githubScore;
  if (student.profileLinked.leetcode) available.leetcode = student.leetcodeScore;
  available.assessment = student.assessmentTheoryScore; // always available

  // renormalize weights over only the signals actually present
  const usedWeights = Object.keys(available).reduce((sum, k) => sum + weights[k], 0);
  const finalScore = Object.entries(available)
    .reduce((sum, [k, v]) => sum + v * (weights[k] / usedWeights), 0);

  return { finalScore, basis: Object.keys(available) }; // transparency: show what it's based on
}
```

This one change — renormalizing instead of hardcoding 50/20/30 — is what makes the feature work for a first-year student with no GitHub, instead of silently scoring them on 30% of the intended signal. Also return `basis` so the UI can honestly show *"scored on: Assessment Theory only"* rather than pretending it's a full multi-signal score.

Soft-skill score is computed separately, same session, stored alongside.

---

## Phase F — Gap Calculation (fixed: embeddings, not set subtraction)

```javascript
// call the SAME Python matching microservice from the Skill Mapping feature
async function computeSkillGap(verifiedTags, targetRequirements) {
  const gaps = [];
  for (const target of targetRequirements) {
    const bestMatch = await Promise.all(
      verifiedTags.map(tag => cosineSimilarity(embed(tag), embed(target)))
    );
    const maxSim = Math.max(...bestMatch, 0);
    if (maxSim < 0.6) gaps.push(target);   // threshold, tunable
  }
  return gaps;
}
```

`"Express.js"` verified against `"Node.js"` required now correctly resolves as *not* a gap, because it reuses the same embedding engine as the course-matching feature — one matching primitive, three call sites (course recommendations, job-role matching, and now skill-gap detection). That's a stronger, more consistent architecture than three different pieces of gap logic.

---

## Phase G — Writing to the Verified Digital Portfolio

```javascript
// portfolio document, versioned per assessment attempt
{
  studentId, treeVersion: "tree_backend_dev_v1",
  technicalScore, technicalBasis: ["github", "assessment"],
  softSkillScores: { communication: 4.2, teamwork: 3.8, ... },
  verifiedTags: ["JWT", "Express.js", "REST APIs"],
  skillGap: ["Docker", "Redux"],
  proctorFlags: { tabSwitches: 1, flaggedForReview: false },
  completedAt, expiresAt: completedAt + 6 months  // re-assessment cadence
}
```

The `expiresAt` field matters for a reason the original doc never addressed: a skill profile from month 1 of a 4-year degree shouldn't be treated as current forever — build in a re-assessment trigger, even if it's just "recommend re-take after 6 months" for the MVP.

---

## The full flow, end to end

```
1. Register → optionally link GitHub/LeetCode (OAuth) → jobs queued in Redis
2. Background workers pull profile data, tag profileLinked flags, extract baseline skill tags
3. Student selects a target role → server loads pre-built (admin-approved) decision tree for that role
4. Server starts session, serves root question with a server-owned deadline
5. Client answers → server validates timing → server (not client) resolves next node → repeat
6. Student separately completes the fixed soft-skill scenario questionnaire
7. Server computes: technical score (weighted, renormalized for missing signals) 
                   + soft-skill score (rubric-based)
8. Server calls the shared embedding microservice to compute skill gap vs. target role
9. Final verified profile + gap array written to the portfolio, versioned and timestamped
10. Downstream: course/job-role/industry recommendations consume this same skillGap array
```

That last line is the payoff worth saying out loud in your pitch: **the assessment engine's output plugs directly into the same recommendation engine we already built** — one data contract (`verifiedTags` + `skillGap`) feeding both features, which is exactly the kind of "one core product, integrated modules" story judges respond to, rather than a pile of disconnected AI demos.