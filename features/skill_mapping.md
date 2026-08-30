# AI-Driven Skill Mapping & Recommendation Architecture

Standard string-matching queries (`$setIntersection` or exact tag matches) fail to capture semantic context. If an industry post requires **Node.js** and a course is tagged **Express.js Development**, basic string filters report a 0% match despite their close conceptual relationship.

To solve this, a Python microservice utilizes pre-trained sentence embeddings to evaluate semantic similarity ("knowledge distance") between candidates and opportunities.

---

## 1. Vector Matching Engine Logic

```
   Student Missing Skills                 Course / Role Skill Tags
["JWT Auth", "Docker", ...]              ["OAuth", "API Security", ...]
             │                                       │
             ▼                                       ▼
    Sentence-BERT Embeddings               Sentence-BERT Embeddings
   (Vector: s1, s2, ...)                  (Vector: c1, c2, ...)
             │                                       │
             └───────────────────┬───────────────────┘
                                 │
                                 ▼
                     Cosine Similarity Score
                       cos(θ) = (A·B) / (||A|| ||B||)
                                 │
                                 ▼
                   Hard Eligibility Filtering
                    (Level & Prerequisites)
                                 │
                                 ▼
                    Ranked Recommendations

```

### Execution Flow

1. **Individual Skill Vectorization:** The Node.js backend passes the student’s missing skills (e.g., `["JWT Authentication", "Docker"]`) to the Python API. Each skill is embedded independently ($\vec{s}_1, \vec{s}_2, ...$) using a pre-trained **Sentence-BERT** model (`all-MiniLM-L6-v2` or `all-mpnet-base-v2`).
2. **Precomputed Target Vectors:** Target items (courses, jobs, industries) are assigned concise skill tags during creation. These tags are embedded once and stored in the database: $\vec{c}_1, \vec{c}_2, ...$
3. **Similarity Calculation & Aggregation:** The microservice calculates the cosine distance between individual missing skill vectors and item tag vectors:

$$\text{Similarity}(\vec{s}, \vec{c}) = \frac{\vec{s} \cdot \vec{c}}{\Vert\vec{s}\Vert \, \Vert\vec{c}\Vert}$$

4. **Filtering & Ranking:** Hard eligibility rules (e.g., prerequisites, experience level) are applied to filter out unsuitable items before ordering the results by score.

---

## 2. Multi-Target Reusability

A single embedding-and-similarity pipeline serves three separate problem statement requirements:

| Recommendation Target | Input A (Student Side) | Input B (Target Side) |
| --- | --- | --- |
| **Skill Development Programs** | Missing Skill Gap ($\vec{s}_{\text{gap}}$) | Course Skill Tags ($\vec{c}_{\text{tags}}$) |
| **Relevant Job Roles** | Current Verified Skills ($\vec{s}_{\text{skills}}$) | Job Required Skills ($\vec{j}_{\text{req}}$) |
| **Target Industries** | Current Verified Skills ($\vec{s}_{\text{skills}}$) | Industry Skill Demand Profile ($\vec{i}_{\text{profile}}$) |

---

## 3. Microservice API Interface

### Request Contract

`POST /api/v1/skill-match`

```json
{
  "student_gap": ["JWT Authentication", "Docker"],
  "target_type": "course",
  "student_level": "intermediate",
  "top_k": 5
}

```

### Response Contract

```json
{
  "status": "success",
  "recommendations": [
    {
      "id": "C104",
      "title": "Secure API Architectures",
      "score": 0.87,
      "matched_on": "JWT Authentication"
    },
    {
      "id": "C209",
      "title": "Container Fundamentals",
      "score": 0.81,
      "matched_on": "Docker"
    }
  ]
}

```

---

## 4. Key Demo Pitch Summary

> "Standard database queries fail at semantic matching—a query for 'NoSQL' won't match a 'MongoDB' course on text alone. Our platform uses a dedicated Python microservice with Sentence-BERT embeddings to convert student skill gaps and course tags into a shared vector space. By calculating cosine similarity and applying prerequisite filters, the engine maps relevant courses, job roles, and target industries through a single unified AI component without needing custom model training."