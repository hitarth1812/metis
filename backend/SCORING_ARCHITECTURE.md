# Scoring Metrics Through LangGraph DAG: Complete Architecture

> **Status**: Production Ready | **Version**: 1.0 | **Last Updated**: March 2026

## Table of Contents

- [Overview](#overview)
- [Architecture Overview](#architecture-overview)
- [Model 1: METIS-CORE Engine](#model-1-metis-core-engine-resume-analysis)
- [Model 2: Assessment Evaluator](#model-2-assessment-evaluator-skill-testing)
- [Model 3: LangGraph Scoring Pipeline](#model-3-langgraph-scoring-pipeline-main-dag)
  - [Graph Structure](#graph-structure)
  - [State Schema](#state-schema-data-flowing-through-dag)
  - [Node 1: Weighted Score](#node-1-weighted-score-calculation)
  - [Node 2: Integrity Check](#node-2-integrity-check)
  - [Node 3: Final Score](#node-3-final-score-calculation)
  - [Node 4: Shortlist & Ranking](#node-4-shortlist--ranking)
- [Complete Data Flow Example](#complete-data-flow-example)
- [Integration with Backend API](#integration-with-backend-api)
- [Configuration & Tuning](#configuration--tuning)
- [Error Handling](#error-handling)
- [Reporting & Explainability](#reporting--explainability)
- [Troubleshooting Guide](#troubleshooting-guide)
- [Technical Implementation Details](#technical-implementation-details)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

The METIS hiring platform uses a **three-model architecture** integrated through a **LangGraph 1.0.8 DAG** to provide comprehensive candidate evaluation:

1. **Model 1 (METIS-CORE)**: Resume parsing and job description analysis
2. **Model 2 (Assessment Evaluator)**: Candidate skill assessment and performance analysis
3. **Model 3 (LangGraph Scoring Pipeline)**: Final scoring orchestration and leaderboard ranking

The three models work in sequence, with Model 3's LangGraph DAG orchestrating the final scoring process using outputs from Models 1 and 2.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      INPUT SOURCES                          │
├─────────────────────────────────────────────────────────────┤
│  Job Description (JD)  │  Candidate Resume  │  Assessment   │
│   (structured text)    │   (text file)      │   (questions  │
│                        │                    │    & answers) │
└────────────┬───────────────────────────────┬────────────────┘
             │                               │
    ┌────────▼───────────┐         ┌────────▼────────────────┐
    │   MODEL 1          │         │    MODEL 2             │
    │  METIS-CORE        │         │  Assessment Evaluator  │
    │                    │         │                        │
    │ JD Parsing         │         │ Resume Analysis        │
    │ Skill Weights (0-1)│         │ Assessment Scoring     │
    │ Skill Importance   │         │ Skill Scores (0-100)   │
    │                    │         │ Resume Claims          │
    └────────┬───────────┘         └────────┬────────────────┘
             │                              │
             └──────────────┬───────────────┘
                            │
              ┌─────────────▼────────────────┐
              │   MODEL 3: LANGGRAPH DAG    │
              │  Scoring Pipeline (Main)   │
              └─────────────┬────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
    ┌───▼────┐         ┌────▼────┐      ┌──────▼─────┐
    │Weighted│         │Integrity│      │   Final    │
    │ Score  │────────▶│  Check  │─────▶│   Score    │
    └────────┘         └────────┘      ├────────────┤
                                        │ Shortlist  │
                                        │ Ranking    │
                                        │ Leaderboard│
                                        └────────────┘
```

---

## Model 1: METIS-CORE Engine (Resume Analysis)

### Purpose
Provides comprehensive resume evaluation using evidence-based scoring.

### Scoring Categories (100 points total)

| Category | Max Points | Description |
|----------|-----------|-------------|
| Skill Evidence Quality | 30 | Technical depth, certifications, relevant keywords |
| Project & Work Authenticity | 25 | Real-world projects, company recognition, impact claims |
| Professional Signal Strength | 15 | Leadership, initiatives, career growth trajectory |
| Impact & Outcomes | 15 | Quantifiable results, metrics, business impact |
| Resume Integrity & ATS Risk | 15 | Consistency, red flags, ATS compatibility |

### Model 1 Output Schema

```python
{
    "model": "metis_core_v1",
    "overall_score": 65,                    # 0-100
    "section_scores": {
        "skill_evidence": 24,               # 0-30
        "project_authenticity": 19,         # 0-25
        "professional_signals": 12,         # 0-15
        "impact_outcomes": 10,              # 0-15
        "resume_integrity": 12              # 0-15 (penalised)
    },
    "strength_signals": [
        "Strong AWS experience",
        "Leadership in 3 projects",
        "Quantified impact metrics"
    ],
    "risk_signals": [
        "Job hopping every 1-2 years",
        "Potential role inflation",
        "Limited quantified outcomes"
    ],
    "ats_flags": [
        "Missing LinkedIn profile",
        "No GitHub link present"
    ],
    "confidence_level": "high",
    "final_reasoning": "Solid candidate with good technical depth..."
}
```

### Model 1 Integration Point
Provides **skill weights** extracted from the job description:
```python
"skill_weights": [
    {"skill": "Python", "weight": 0.30, "importance": 9},
    {"skill": "AWS", "weight": 0.25, "importance": 8},
    {"skill": "SQL", "weight": 0.20, "importance": 7},
    {"skill": "Leadership", "weight": 0.15, "importance": 6},
    {"skill": "Communication", "weight": 0.10, "importance": 5}
]
```

---

## Model 2: Assessment Evaluator (Skill Testing)

### Purpose
Evaluates candidate performance on structured skill assessments and validates resume claims.

### Skill Assessment Output Schema

```python
"skill_scores": [
    {
        "skill": "Python",
        "score": 78,                        # 0-100
        "questions_attempted": 10,
        "correct_answers": 8,
        "avg_difficulty": 6.5,              # 1-10 scale
        "time_spent_seconds": 480
    },
    {
        "skill": "AWS",
        "score": 85,
        "questions_attempted": 10,
        "correct_answers": 9,
        "avg_difficulty": 7.2
    }
]
```

### Resume Claims (for Integrity Checking)

```python
"resume_claims": [
    {
        "skill": "Python",
        "claimed_level": "Expert",          # Expert|Advanced|Intermediate|Beginner
        "years_experience": 7
    },
    {
        "skill": "AWS",
        "claimed_level": "Advanced",
        "years_experience": 4
    },
    {
        "skill": "Leadership",
        "claimed_level": "Expert",
        "years_experience": 6
    }
]
```

### Model 2 Processing Pipeline
1. **Question Parsing**: Extract skill-based questions from assessment
2. **Answer Evaluation**: Grade answers using AI + heuristics
3. **Score Calculation**: Compute skill scores (0-100) based on correctness and difficulty
4. **Resume Extraction**: Parse resume to identify skill claims and experience levels
5. **Validation Link**: Map assessment scores to resume claims for integrity checking

---

## Model 3: LangGraph Scoring Pipeline (Main DAG)

### Graph Structure

The core LangGraph workflow is a **linear DAG** with 4 nodes:

```
START
  │
  ▼
┌──────────────────────┐
│  weighted_score_node │
│  (Σ score × weight)  │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ integrity_check_node │
│ (resume vs actual)   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  final_score_node    │
│ (weighted × integrity)
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  shortlist_node      │
│ (rank & advance)     │
└──────────┬───────────┘
           │
           ▼
          END
```

### State Schema (Data Flowing Through DAG)

```python
class ScoringState(TypedDict):
    # Input fields (from Models 1 & 2)
    candidate_id: str
    candidate_name: str
    job_id: str
    job_title: str
    
    # From Model 1 (JD parsing)
    skill_weights: List[SkillWeight]
    
    # From Model 2 (Assessment)
    skill_scores: List[SkillScore]
    resume_claims: List[ResumeClaim]
    
    # Node 1 Output: Weighted Score
    weighted_score: float  # 0-100
    skill_contributions: List[Dict]
    
    # Node 2 Output: Integrity Check
    integrity_score: float  # 0-100
    consistency_flags: List[ConsistencyFlag]
    
    # Node 3 Output: Final Score
    final_score: float
    
    # Node 4 Output: Shortlist Status
    rank: Optional[int]
    shortlist_status: Literal['round_1', 'round_2', 'rejected']
    
    # Metadata
    processing_errors: List[str]
```

---

## Node 1: Weighted Score Calculation

### Purpose
Calculate how well a candidate performs on the skills required by the job.

### Formula
```
Weighted Score = Σ(skill_score_i × skill_weight_i)
```

Where:
- `skill_score_i` = Candidate's assessment score for skill i (0-100)
- `skill_weight_i` = Importance weight of skill i from JD (0-1)

### Example Calculation

**Inputs:**

Model 2 Skill Scores:
```
Python: 78, AWS: 85, SQL: 72, Leadership: 65, Communication: 80
```

Model 1 Skill Weights:
```
Python: 0.30, AWS: 0.25, SQL: 0.20, Leadership: 0.15, Communication: 0.10
```

**Calculation:**
```
Weighted Score = (78 × 0.30) + (85 × 0.25) + (72 × 0.20) + (65 × 0.15) + (80 × 0.10)
               = 23.4 + 21.25 + 14.4 + 9.75 + 8.0
               = 76.8
```

### Node 1 Output

```python
{
    "weighted_score": 76.8,
    "skill_contributions": [
        {
            "skill": "Python",
            "score": 78,
            "weight": 0.30,
            "contribution": 23.4,
            "percentage_of_total": 30.5
        },
        {
            "skill": "AWS",
            "score": 85,
            "weight": 0.25,
            "contribution": 21.25,
            "percentage_of_total": 27.7
        },
        # ... other skills
    ]
}
```

### Implementation Details

- **Missing Skills**: If a skill is in the JD weights but not in the assessment scores, it's treated as 0
- **Extra Skills**: If candidate has skills not in the JD, they are ignored
- **Normalization**: Weights are normalized to sum to 1.0 if needed
- **Range**: Always clamped to 0-100

---

## Node 2: Integrity Check

### Purpose
Detect resume-assessment mismatches (e.g., claims "Expert" but scored 45).

### Proficiency Score Thresholds

Expected minimum assessment scores for claimed levels:

```python
PROFICIENCY_THRESHOLDS = {
    'Expert': 80,           # Claims expertise but scored < 80 = RED FLAG
    'Advanced': 70,         # Claims advanced but scored < 70
    'Intermediate': 50,     # Claims intermediate but scored < 50
    'Beginner': 30          # Claims beginner but scored < 30
}
```

### Severity Classification

```python
SEVERITY_THRESHOLDS = {
    'high': 40,     # Discrepancy > 40 points (e.g., claimed Expert, scored 35)
    'medium': 20,   # Discrepancy 20-40 points
    'low': 10       # Discrepancy 10-20 points
}
```

### Penalty Calculation

```
Penalty = (Discrepancy / 100) × Severity_Weight
```

- High severity: 10 points penalty per comparison
- Medium severity: 5 points penalty per comparison
- Low severity: 2 points penalty per comparison

### Example Calculation

**Inputs:**

Candidate Resume Claims:
```
Python: Expert (claimed)
AWS: Advanced (claimed)
Leadership: Expert (claimed)
```

Assessment Scores:
```
Python: 78
AWS: 85
Leadership: 50
```

**Analysis:**
```
Python vs Expert:
  Expected: 80, Actual: 78
  Discrepancy: 2 points (LOW severity)
  Penalty: 2 points
  Status: ✓ ACCEPTABLE

AWS vs Advanced:
  Expected: 70, Actual: 85
  Discrepancy: -15 (over-performed - no penalty)
  Status: ✓ EXCELLENT

Leadership vs Expert:
  Expected: 80, Actual: 50
  Discrepancy: 30 points (MEDIUM severity)
  Penalty: 5 points
  Status: ⚠️ DISCREPANCY
```

**Final Integrity Score:**
```
Integrity Score = 100 - (2 + 5) = 93
```

### Node 2 Output

```python
{
    "integrity_score": 93.0,
    "consistency_flags": [
        {
            "skill": "Leadership",
            "claimed_level": "Expert",
            "actual_score": 50,
            "expected_score": 80,
            "discrepancy": 30,
            "severity": "medium"
        }
    ]
}
```

### Special Cases

- **No Resume Claims for Skill**: No check performed, no penalty
- **Skill Not Assessed**: Flagged as error but doesn't affect integrity score
- **Perfect Consistency**: Bonus +5 to integrity if all claims match performance

---

## Node 3: Final Score Calculation

### Purpose
Combine assessment performance with credibility (integrity).

### Formula
```
Final Score = Weighted Score × (Integrity Score / 100)
```

### Interpretation

The integrity score acts as a **credibility multiplier**:
- `Integrity = 100%`: Final Score = Weighted Score (full credit)
- `Integrity = 75%`: Final Score = Weighted Score × 0.75 (25% penalty)
- `Integrity = 50%`: Final Score = Weighted Score × 0.50 (50% penalty)

### Example Calculation

**Inputs from Previous Nodes:**
```
Weighted Score: 76.8
Integrity Score: 93.0
```

**Calculation:**
```
Final Score = 76.8 × (93.0 / 100)
            = 76.8 × 0.93
            = 71.4
```

**Interpretation:**
- Strong assessment performance (76.8) with minor resume discrepancies (93% integrity)
- Results in solid final score of 71.4

### Impact Analysis

```python
{
    "scores": {
        "weighted_score": 76.8,
        "integrity_score": 93.0,
        "final_score": 71.4,
    },
    "integrity_impact": {
        "points_lost": 5.4,          # (76.8 - 71.4)
        "percentage_lost": 7.0,      # (5.4 / 76.8) × 100
        "reason": "Resume-assessment mismatch in Leadership claim"
    },
    "score_breakdown": {
        "Python": 23.4,              # 78 × 0.30
        "AWS": 21.25,                # 85 × 0.25
        "SQL": 14.4,                 # 72 × 0.20
        "Leadership": 9.75,          # 65 × 0.15
        "Communication": 8.0         # 80 × 0.10
    }
}
```

---

## Node 4: Shortlist & Ranking

### Purpose
Determine advancement to next round and create leaderboard rankings.

### Shortlist Thresholds

When processing individual candidates (score-based):

```python
ROUND_2_SCORE_THRESHOLD = 85    # Final Score >= 85
ROUND_1_SCORE_THRESHOLD = 70    # Final Score >= 70
```

When batch processing (percentile-based):

```python
ROUND_2_PERCENTILE = 10         # Top 10% of candidates
ROUND_1_PERCENTILE = 30         # Top 30% of candidates
```

### Shortlist Status Decision

```python
if final_score >= 85:
    status = 'round_2'          # Advanced to final interview
elif final_score >= 70:
    status = 'round_1'          # Advanced to technical interview
else:
    status = 'rejected'         # Not moving forward
```

### Example Shortlisting

Scenario: 100 candidates total

**Batch Processing (Percentile-Based):**
```
Round 2 (Top 10%):  Top 10 candidates OR score >= 85
Round 1 (Top 30%):  Top 30 candidates OR score >= 70
Rejected:          Remaining 70 candidates
```

**Score-Based:**
```
Round 2:  All candidates with score >= 85
Round 1:  All candidates with score 70-84
Rejected: All candidates with score < 70
```

### Node 4 Output

```python
{
    "final_score": 71.4,
    "rank": 15,                 # Among all candidates
    "shortlist_status": "round_1",
    "advancement_note": "Qualified for Round 1 (Technical Interview)",
    "percentile": 85,           # Top 15%
}
```

---

## Complete Data Flow Example

### Scenario
Candidate "Alice Chen" applying for "Senior Backend Engineer"

### Model 1 Processing (JD & Resume Analysis)
```
INPUT: Job Description + Resume
OUTPUT: skill_weights
  Python: 0.30 (critical)
  AWS: 0.25 (critical)
  SQL: 0.20
  Leadership: 0.15
  Communication: 0.10
```

### Model 2 Processing (Assessment)
```
INPUT: Assessment Answers + Resume Claims
OUTPUT: 
  skill_scores:
    Python: 78
    AWS: 85
    SQL: 72
    Leadership: 65
    Communication: 80
  
  resume_claims:
    Python: Expert (7 years)
    AWS: Advanced (4 years)
    Leadership: Expert (6 years)
```

### Model 3 DAG Processing

**Node 1 (Weighted Score):**
```
weighted_score = (78 × 0.30) + (85 × 0.25) + (72 × 0.20) + (65 × 0.15) + (80 × 0.10)
               = 76.8
```

**Node 2 (Integrity Check):**
```
Python: 78 vs Expected 80 (Expert) → -2 discrepancy (low) → 0 penalty
AWS: 85 vs Expected 70 (Advanced) → +15 (over-performed) → 0 penalty
Leadership: 65 vs Expected 80 (Expert) → -15 discrepancy (medium) → 5 penalty
integrity_score = 100 - 5 = 95
```

**Node 3 (Final Score):**
```
final_score = 76.8 × (95 / 100) = 72.96
```

**Node 4 (Shortlist):**
```
final_score (72.96) >= 70 AND < 85
Status: ROUND_1 (Technical Interview)
Rank: 23 out of 150 total candidates
Percentile: 85th
```

### Final Leaderboard Entry
```json
{
    "candidate_id": "alice_chen_001",
    "candidate_name": "Alice Chen",
    "weighted_score": 76.8,
    "integrity_score": 95.0,
    "final_score": 72.96,
    "rank": 23,
    "shortlist_status": "round_1",
    "skill_breakdown": [
        {"skill": "Python", "score": 78, "weight": 0.30, "contribution": 23.4},
        {"skill": "AWS", "score": 85, "weight": 0.25, "contribution": 21.25},
        {"skill": "SQL", "score": 72, "weight": 0.20, "contribution": 14.4},
        {"skill": "Leadership", "score": 65, "weight": 0.15, "contribution": 9.75},
        {"skill": "Communication", "score": 80, "weight": 0.10, "contribution": 8.0}
    ],
    "has_consistency_issues": true,
    "consistency_flag": "Leadership claim (Expert) below expected performance (65 vs 80)"
}
```

---

## Integration with Backend API

### API Endpoints

**1. Run Full Pipeline (Models 1→2→3):**
```
POST /api/scoring/pipeline
{
    "candidate_id": "alice_chen_001",
    "resume_text": "...",
    "job_description": "...",
    "assessment_answers": {...}
}
```

**2. Generate Leaderboard (Batch Processing):**
```
POST /api/leaderboard/generate
{
    "job_id": "sr_backend_001",
    "candidate_scores": [...]
}
```

**3. Get Individual Score Breakdown:**
```
GET /api/scoring/{candidate_id}/{job_id}
```

---

## Configuration & Tuning

### Adjustable Parameters

```python
# Node 1: Weighted Score
normalize_unknown_weights = False  # How to handle missing weights

# Node 2: Integrity Check
PROFICIENCY_THRESHOLDS = {
    'Expert': 80,
    'Advanced': 70,
    'Intermediate': 50,
    'Beginner': 30
}

PENALTY_WEIGHTS = {
    'high': 10,
    'medium': 5,
    'low': 2
}

# Node 4: Shortlist
ROUND_1_SCORE_THRESHOLD = 70
ROUND_2_SCORE_THRESHOLD = 85
ROUND_1_PERCENTILE = 30
ROUND_2_PERCENTILE = 10
```

### Performance Considerations

- **Linear DAG**: Straightforward execution, no branching
- **Scalability**: Can process 1000s of candidates in batch mode
- **LangGraph Integration**: Enables future enhancements with conditional branching, loop, and dynamic node addition
- **State Management**: All calculations are deterministic and reproducible

---

## Error Handling

### Processing Errors Captured

1. **Missing Skills**: Skill in JD but not in assessment → Treated as 0, logged as error
2. **Weight Normalization**: Weights don't sum to 1.0 → Auto-normalized with warning
3. **Invalid Inputs**: Negative scores, missing fields → Validation at state entry
4. **Resume Parsing Failures**: Can't extract claims → Skips integrity check for that skill

### Error Log Example

```python
"processing_errors": [
    "Skill 'Docker' in JD weight map but not found in assessment scores",
    "Resume claim for 'React' has missing 'years_experience' field",
    "Skill weights sum to 0.95 instead of 1.0 - normalized"
]
```

---

## Reporting & Explainability

### Score Explanation Report

```
CANDIDATE EVALUATION REPORT
===========================

Candidate: Alice Chen
Job: Senior Backend Engineer
Evaluation Date: 2026-03-26

ASSESSMENT PERFORMANCE (Weighted Score: 76.8/100)
────────────────────────────────────────────────
  Python        78/100  ×  30% weight  =  23.40  (30.5% of score)
  AWS           85/100  ×  25% weight  =  21.25  (27.7% of score)
  SQL           72/100  ×  20% weight  =  14.40  (18.8% of score)
  Leadership    65/100  ×  15% weight  =   9.75  (12.7% of score)
  Communication 80/100  ×  10% weight  =   8.00  (10.4% of score)

CREDIBILITY CHECK (Integrity Score: 95/100)
────────────────────────────────────────
  ✓ Python: Claims Expert (80+), Scored 78 - ACCEPTABLE
  ✓ AWS: Claims Advanced (70+), Scored 85 - EXCELLENT
  ⚠ Leadership: Claims Expert (80+), Scored 65 - DISCREPANCY (-15)
  
  Impact: -5 points integrity penalty

FINAL SCORE: 72.96/100
──────────────────────
  Calculation: 76.8 × (95/100) = 72.96
  Decision: ADVANCED TO ROUND 1 (Technical Interview)
  Percentile: 85th (Rank 23 of 150)

KEY INSIGHTS
────────────
• Strong technical skills in core areas (Python, AWS)
• Leadership claim needs verification in technical interview
• Overall credible candidate with good assessment performance
```

---

## Troubleshooting Guide

### Issue: Final Score Much Lower Than Weighted Score

**Cause**: Low integrity score due to multiple resume discrepancies
**Solution**: 
1. Review consistency flags
2. Consider resume clarity or interview verification
3. Check if claims match actual experience

### Issue: Candidate Scores Vary Between Runs

**Cause**: Non-deterministic integrity checks or missing state values
**Solution**:
1. Ensure all resume claims are extracted
2. Verify skill_scores and skill_weights are complete
3. Check for floating-point precision issues

### Issue: Unexpected Shortlist Status

**Cause**: Score threshold configurations or batch vs individual processing
**Solution**:
1. Verify `ROUND_1_SCORE_THRESHOLD` and `ROUND_2_SCORE_THRESHOLD`
2. Use batch processing for percentile-based shortlisting
3. Check percentile cutoff calculations

---

## Technical Implementation Details

### LangGraph 1.0.8 Compatibility

- **StateGraph**: Main workflow container
- **TypedDict**: Strict typing for state schema
- **add_node()**: Register processing functions
- **add_edge()**: Define workflow transitions
- **compile()**: Generate executable graph

### Python Dependencies

```
langgraph==1.0.8
langchain>=0.1.0
groq  # For AI-powered scoring
pydantic  # State validation
```

### Execution Example

```python
from scoring_model.langgraph_model import create_scoring_graph
from scoring_model.state import ScoringState

# Initialize
graph = create_scoring_graph()

# Prepare input
input_state = ScoringState(
    candidate_id="alice_chen_001",
    candidate_name="Alice Chen",
    job_id="sr_backend_001",
    job_title="Senior Backend Engineer",
    skill_scores=[...],
    skill_weights=[...],
    resume_claims=[...]
)

# Execute
result = graph.invoke(input_state)

# Access results
print(f"Final Score: {result['final_score']}")
print(f"Status: {result['shortlist_status']}")
print(f"Consistency Issues: {result['consistency_flags']}")
```

---

## Summary

| Component | Responsibility | Input | Output |
|-----------|----------------|-------|--------|
| **Model 1** | Resume parsing & JD analysis | Job Description + Resume | Skill weights, categories |
| **Model 2** | Skill assessment & validation | Assessment answers + Resume claims | Skill scores, performance metrics |
| **Model 3** | Orchestration & final scoring | Outputs from Models 1 & 2 | Final score, rank, leaderboard |
| **Node 1** | Weighted scoring | Skill scores + weights | weighted_score (0-100) |
| **Node 2** | Integrity checking | Resume claims + actual scores | integrity_score (0-100) |
| **Node 3** | Final calculation | Weighted + integrity | final_score (0-100) |
| **Node 4** | Ranking & advancement | Final score | shortlist_status, rank |

This integrated approach ensures **fair, transparent, and explainable candidate evaluation** across all skill dimensions.

---

## Contributing

### Code Style & conventions
- Follow PEP 8 for Python code
- Use type hints in all functions
- Include docstrings for classes and methods
- Maintain test coverage above 80%

### To Contribute:

1. **Fork** the repository
2. **Create** a feature branch (`git checkout -b feature/scoring-improvement`)
3. **Make** your changes with clear commit messages
4. **Test** thoroughly before submitting
5. **Submit** a Pull Request with a detailed description

### Reporting Issues
- Use GitHub Issues for bug reports
- Include steps to reproduce
- Provide example inputs/outputs where applicable
- Specify the LangGraph version being used

---

## Related Documentation

- **Resume Parser**: See `backend/models/metis/resume_parser.py`
- **JD Parser**: See `backend/models/metis/jd_parser.py`
- **LangGraph Model**: See `backend/models/scoring/langgraph_model.py`
- **State Schema**: See `backend/models/scoring/state.py`
- **API Routes**: See `backend/routes/` for integration points

---

## References

- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [Python TypedDict](https://docs.python.org/3/library/typing.html#typing.TypedDict)
- [Pydantic Validation](https://docs.pydantic.dev/)

---

## License

This project is part of the METIS hiring platform. All rights reserved. Internal use only.

---

## Support & Questions

For questions or support regarding this architecture:
- Check the [Troubleshooting Guide](#troubleshooting-guide) section
- Review existing GitHub Issues
- Contact the METIS development team

**Last Updated**: March 2026  
**Maintained By**: METIS Engineering Team
