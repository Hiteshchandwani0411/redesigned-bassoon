# Proposed System Architecture & Tech Stack

For an SIH hackathon, the ideal architecture balances **rapid 36-hour prototyping** with **production-level scalability**. A **Modular Monolith Architecture** with a dedicated Python microservice for AI/ML matching provides a clear path to execution without over-engineering.

---

## 1. System Architecture Blueprint

```
                     ┌─────────────────────────────────────────┐
                     │          Frontend Client (Web)          │
                     │       React.js / Tailwind CSS / Vite     │
                     └────────────────────┬────────────────────┘
                                          │  REST APIs / WebSockets
                                          ▼
                     ┌─────────────────────────────────────────┐
                     │           API Gateway / Middleware       │
                     │          JWT Auth + RBAC Middleware     │
                     └────────────────────┬────────────────────┘
                                          │
        ┌─────────────────────────────────┼─────────────────────────────────┐
        ▼                                 ▼                                 ▼
┌───────────────┐                 ┌───────────────┐                 ┌───────────────┐
│ Student & TPO │                 │ Skill & Job   │                 │ Faculty & FDP │
│   Module      │                 │  Engine       │                 │ Booking Module│
└───────┬───────┘                 └───────┬───────┘                 └───────┬───────┘
        │                                 │                                 │
        └─────────────────────────────────┼─────────────────────────────────┘
                                          │
                        ┌─────────────────┴─────────────────┐
                        ▼                                   ▼
          ┌───────────────────────────┐       ┌───────────────────────────┐
          │   Primary Database        │       │   Python Recommendation   │
          │  MongoDB / PostgreSQL     │◄─────►│        Microservice       │
          │ (Profiles, Jobs, Logs)    │       │ (FastAPI + NLP Matching)  │
          └───────────────────────────┘       └───────────────────────────┘

```

---

## 2. Tech Stack Overview

| Layer | Technology | Key Purpose in the Portal |
| --- | --- | --- |
| **Frontend Framework** | **React.js (Vite)** + **Tailwind CSS** | Fast UI development with isolated dashboard components for Student, TPO, Industry, and Faculty roles. |
| **Core Backend** | **Node.js** + **Express.js** | Handles API routing, Role-Based Access Control (RBAC), authentication, and CRUD operations. |
| **AI / Matching Engine** | **Python (FastAPI)** | Microservice dedicated to parsing text and running similarity algorithms between student profiles and job descriptions. |
| **Primary Database** | **MongoDB** (or PostgreSQL) | Stores flexible user profiles, nested micro-skill matrices, and dynamic job postings. |
| **Cache & Queue** | **Redis** | Caches high-frequency search requests and queues background processing for job applications. |
| **Analytics & UI Charts** | **Recharts** / **Chart.js** | Visualizes placement readiness and skill metrics on institutional TPO dashboards. |
| **Media / File Storage** | **Cloudinary** / **AWS S3** | Stores student resumes, certificates, and completion reports securely. |

---

## 3. Core Technical Features

### Role-Based Access Control (RBAC)

* **User Scopes:** `STUDENT`, `FACULTY`, `INDUSTRY_HR`, and `INSTITUTION_TPO`.
* **Implementation:** Standard JWT authentication with role-checking middleware to restrict sensitive actions (e.g., job posting restricted to `INDUSTRY_HR`, FDP booking to `FACULTY`).

### AI Skill-Matching Engine

Calculates a **Skill Compatibility Index (%)** using text parsing and vector matching:

1. **Extraction:** Extracts target skills from job descriptions via keyword parsing.
2. **Vectorization:** Converts candidate skills, project tags, and assessment scores into matching vectors.
3. **Similarity Score:** Computes vector alignment using standard cosine distance:

$$\text{Similarity Score} = \frac{\vec{A} \cdot \vec{B}}{\Vert{}\vec{A}\Vert{} \Vert{}\vec{B}\Vert{}}$$

### Verified External Profiles

* Integrates directly with third-party APIs (such as GitHub or competitive programming platforms) to fetch public activity, verified stats, and badges automatically—reducing unverified resume claims.

### Faculty Slot Booking Engine

* Implements transaction locks and status tracking (`AVAILABLE`, `RESERVED`, `COMPLETED`) to allow faculty members to reserve limited industry slots or FDPs without schedule collisions.

---

## 4. Key Advantages for Hackathon Execution

* **Development Speed:** Node.js speeds up core CRUD development, leaving time to refine user interactions.
* **Separation of Concerns:** Offloading match algorithms to a specialized Python service cleanly isolates analytical logic from standard web traffic.
* **Demonstrated Scalability:** The separation between I/O-heavy API routing (Node.js) and CPU-bound analytical tasks (Python) demonstrates real-world software design principles to judges.