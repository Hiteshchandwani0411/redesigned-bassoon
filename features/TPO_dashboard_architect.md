# Institutional Analytics & TPO Dashboard Architecture

The **Institutional Analytics & TPO Dashboard** transforms the platform from a individual utility into an enterprise-grade B2B SaaS solution for academic institutions. By replacing reactive reporting with proactive, predictive tracking and employing optimized data-processing strategies, the dashboard empowers Training and Placement Officers (TPOs) and Deans to identify skill bottlenecks across entire student cohorts long before recruitment drives begin.

---

## 1. System & Data Architecture Flowchart

```text
[ Raw Student Data ] ──────> [ Node.js Streams (csv-parser) ] ───> Bulk Ingestion (MongoDB)
(GitHub, Tests, ERP)                                                  │
                                                                      ▼
[ Scheduled Nightly Cron ] ──> Executes Aggregation Pipeline ───> [ Materialized Views ]
(node-cron @ 02:00 AM)                                                │
                                                                      ▼
[ TPO Login / Request ] ────> Fetches Pre-Calculated Summary ───> [ Redis Caching Layer ]
                                                                      │
                                                                      ▼
                                                              [ React TPO Dashboard ]
                                                              ├── Predictive Skill Heatmap
                                                              ├── At-Risk Cohort Alerts
                                                              └── Role-Gated Access (RBAC)

```

---

## 2. Technical Challenges & Architectural Defenses

### Challenge 1: High Concurrency & Aggregation Overload

* **The Risk:** Calculating real-time skill matrix aggregations, assessment performance, and activity metrics across thousands of active students on demand will cause backend API timeouts and application server crashes.
* **Architectural Defense:** **Materialized Views + Redis Caching.**
* **Implementation:** A background cron worker (`node-cron`) runs during low-traffic hours (e.g., 2:00 AM) to process complex `$lookup` and `$group` aggregations across the student database.
* The computed results are saved into a dedicated `DashboardStats` collection and cached in **Redis**.
* When a TPO logs in, the backend serves the pre-calculated summary instantly with sub-50ms latency.



### Challenge 2: Reactive "Post-Mortem" Reporting

* **The Risk:** Traditional institutional portals display placement failures after recruitment seasons conclude, making timely intervention impossible.
* **Architectural Defense:** **Predictive "At-Risk" Tracking Pipeline.**
* **Implementation:** An analytical routine checks candidate skill growth velocity against incoming industry job vectors.
* If a scheduled campus drive requires specific skills (e.g., MongoDB Aggregations or Docker) and a cohort demonstrates insufficient verified progress within a 30-day window, the system flags the group in the TPO’s **At-Risk Cohort Pipeline**.
* TPOs can immediately initiate targeted Faculty Development Programs (FDPs) or industry workshops to resolve the gap before interview dates.



### Challenge 3: Ingestion Friction for Large Datasets

* **The Risk:** Ingesting multi-megabyte CSV exports containing thousands of student records can exhaust server memory (RAM) and cause system crashes.
* **Architectural Defense:** **Streamed Memory-Efficient Ingestion Pipeline.**
* **Implementation:** Bulk uploads use Node.js `Streams` (`csv-parser`) to read, parse, and commit incoming CSV files in bounded memory chunks rather than buffering the entire payload into RAM at once.



### Challenge 4: Data Isolation & Role-Based Security

* **The Risk:** Unrestricted visibility into student salary packages, confidential feedback, or cross-departmental evaluations violates data privacy requirements.
* **Architectural Defense:** **Role-Based Access Control (RBAC) Middleware.**
* **Implementation:** JSON Web Tokens (JWT) store user roles (`SuperAdmin_TPO`, `Department_HOD`, `Faculty_Mentor`). Access to backend aggregation routes is enforced at the middleware layer to strictly partition domain visibility.



---

## 3. Data Scoping & Role Access Matrix

| Role | Scope of Access | Dashboard Capabilities |
| --- | --- | --- |
| **SuperAdmin / TPO** | Institution-wide | Full access to institutional heatmaps, hiring forecasts, macro skill gaps, and bulk workshop scheduling. |
| **Department HOD** | Branch-specific (e.g., CSE) | View department cohort velocity, skill distribution, and branch-specific placement rates. |
| **Faculty Mentor** | Assigned Mentees ($\sim 20$ students) | Detailed individual progress tracking, action-item assignment, and 1-on-1 mentorship logs. |

---

## 4. Pitch Script

> *"Judges, current university ERPs are 'Descriptive'—they act as digital filing cabinets that tell TPOs who got placed and who didn't after the fact. Our platform is 'Predictive'.*
> *Because processing live data for thousands of students is incredibly resource-heavy, our backend utilizes automated Cron Jobs to generate materialized views every night. This allows the dashboard to instantly render a 'Live Skill Heatmap' without crashing. More importantly, we engineered an 'At-Risk' detection algorithm. If our system detects that 40% of the 3rd-year cohort is lacking the verified API integration skills required by an upcoming recruiter, it flags this gap to the TPO months in advance. The TPO can then proactively deploy targeted industry workshops through our platform to bridge that exact gap. We are shifting institutional analytics from a post-mortem report to a proactive intervention system."*