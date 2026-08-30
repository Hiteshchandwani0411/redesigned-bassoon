# Dynamic Skill Assessment Engine Architecture

The **Dynamic Skill Assessment Engine** combines passive code profiling, pre-computed adaptive testing, and strict anti-cheat mechanisms. This blueprint outlines the MERN-stack architecture and presentation narrative for the platform's core assessment workflow.

---

## 1. Architectural Flowchart

```text
[ Industry Requirement ] ──────┐ (Pre-computation)
                               ▼
                        [ LLM Microservice ] ───> Generates JSON Decision Tree
                               │
[ Student Registration ]       ▼
        │               [ MongoDB DB ] (Stores Assessment Trees)
        ▼                      │
[ Link GitHub & LeetCode ]     │
(OAuth 2.0 - Read Access)      │
        │                      │
        ▼                      ▼
[ Redis Job Queue ] <──── [ Node/Express Server ] ───> [ WebRTC/Canvas API ]
(Handles API limits)           │                       (Live Snapshot Proctoring)
        │                      │
        ▼                      ▼
[ 3rd Party APIs ]        [ React Frontend ]
(GitHub, LeetCode)        (The Assessment UI)
        │                      │
        │                      ├──> 1. Fetches Node 1 from DB
        │                      ├──> 2. Starts Server-Side Timestamp
        │                      ├──> 3. Tracks Page Visibility (Tab switch)
        │                      └──> 4. Renders Next Node instantly based on answer
        │
        └──────────────────────┐
                               ▼
                    [ Final Weighted Scoring ]
         (50% GitHub Code + 20% LeetCode Logic + 30% Assessment Theory)
                               │
                               ▼
                   [ Verified Digital Portfolio ]
                   (Calculates Exact Skill Gap Array)

```

---

## 2. Technical Implementation Roadmap

### Phase A: Passive Profiling (Background Verification)

1. **OAuth Integration:** Utilizes GitHub OAuth to acquire read-only access tokens, resolving access limitations for private repositories.
2. **Asynchronous Job Queuing:** Pushes profile sync tasks to a Redis queue (`BullMQ`) to manage rate limits and maintain system throughput during high-concurrency registration windows.
3. **Repository Parsing Engine:** Scans authenticated user repositories to detect framework implementations (e.g., Express routing, template rendering, nested MongoDB queries) and populates baseline skill tags automatically. Queries competitive programming APIs to log problem-solving streaks and verified logical capabilities.

### Phase B: Active Assessment (Adaptive Testing)

1. **State Initialization:** The React client fetches the root question node from the pre-generated JSON tree in MongoDB.
2. **Server-Side Verification:** Node.js logs a `fetched_at` timestamp per question, applying strict time limits (e.g., 45 seconds) to prevent external lookup.
3. **Proctoring Hooks:**
* Uses the `document.visibilityState` API in React to flag tab switches or window minimization events, triggering instant assessment termination upon violation.
* Utilizes a lightweight Canvas API script to capture random webcam frames for identity verification.


4. **Zero-Latency State Routing:** Upon selecting an answer, the client reads the associated `next_node_id` from local state and renders the subsequent question node instantly.

### Phase C: Gap Calculation

At assessment completion, the server evaluates candidate tags against target job requirements:

$$\text{Skill Gap} = \text{Target Requirements} \setminus \text{Verified Student Tags}$$

*Example:*

* **Target:** `["JWT", "Redux", "Docker"]`
* **Verified:** `["JWT"]`
* **Calculated Gap:** `["Redux", "Docker"]`

The calculated gap updates the candidate profile and enables targeted application access.

---

## 3. Pitch Script

> *"Judges, current skill assessments on platforms like Superset are fundamentally broken. They rely on static, easily manipulated questionnaires. Our platform introduces a highly secure, three-tier Verification Engine.*
> *First, we eliminate self-reporting. When a student registers, they authenticate via OAuth. Our backend utilizes Redis queues to asynchronously parse their GitHub and problem-solving profiles, scanning for actual code implementation—like complex database schemas or consistent algorithmic streaks—even within private repositories.*
> *Second, we execute the Active Assessment. To eliminate LLM latency, our GenAI pipeline pre-computes dynamic decision-trees based on live industry demands. The student experiences a zero-latency, adaptive test that adjusts difficulty based on their real-time answers.*
> *Finally, to guarantee integrity in the AI era, we implemented strict server-side timestamp validation and browser-visibility hooks. A student mathematically does not have the time to copy a prompt into a background LLM without failing the test. We provide HR with undeniably verified, multi-dimensional talent, not just a PDF resume."*