# Justification of the Harness Design

## 1. What the system may do

This system is restricted to a single, narrow operational purpose: preliminary screening of adult scholarship applications (`purpose == "scholarship_screening"`) for the synthetic Tulpar Scholarship Fund. It produces an advisory triage outcome (`shortlist`, `reject`, `escalate`, or `refuse`) using `Qwen/Qwen2.5-3B-Instruct` wrapped inside a deterministic Python harness (`Harness` in `lab06_core.py`). It is **not** authorised to issue unreviewed final rejections on ambiguous, high-merit, or anomalous applications, nor to repurpose applicant records for any other decision (such as admissions, credit, or marketing).

Each active control in `HARNESS` enforces a concrete boundary in code:

- **`validate_input=True`**: Calls `validate_profile(profile)` before any model inference to enforce data types and valid domains (`gpa` in `0.0–4.0`, `exam_score` in `0–140`, `household_income_bracket` in `1–5`, `achievements` in `0–5`, `age` in `16–40`, boolean `consent_to_automated_processing`).
- **`refuse=True`**: Runs `refusal_reason(profile, purpose)` and `my_extra_refusal(profile, purpose)` prior to inference, halting disallowed requests with `outcome="refuse"` and `score=None`.
- **`blind=("name", "university")`**: Strips `name` and `university` via `render_for_model(profile, withheld=BLIND)` so the LLM never receives language/gender-coded name cues or university-tier prestige signals.
- **`log_path="decisions.jsonl"`**: Appends a structured JSON record for every screening, escalation, and refusal event.
- **`escalate=True` with `borderline=(45, 60)` and `my_extra_escalation`**: Diverts uncertain, unparseable, contradictory, high-merit/high-need rejection, or bias-tainted model outputs to human reviewers (`escalated_to_human=True`).

## 2. What it must refuse

The harness enforces hard refusal boundaries **before** invoking `ScreeningAgent`:

1. **Purpose outside scope (`purpose_not_allowed`)**: Any request whose `purpose` is not in `ALLOWED_PURPOSES = {"scholarship_screening"}` is refused immediately.
2. **Minors (`minor`)**: Applicants with `age < 18` are refused automated screening and must follow a separate human-led workflow with legal-guardian consent.
3. **Missing consent (`no_consent`)**: Applicants whose `consent_to_automated_processing` is not explicitly `True` are refused.
4. **Invalid or out-of-range inputs (`invalid_input`)**: Malformed types or impossible values (e.g., negative GPA or out-of-bounds exam scores) are refused rather than silently coerced.
5. **Adversarial or malformed free text (`prompt_injection_suspected`, `empty_free_text`, `oversized_free_text`)**: `my_extra_refusal` inspects the applicant-supplied free-text fields (`name`, `university`, `program`) for control characters/newlines, excessive length (>120 characters), empty strings, and prompt-injection patterns (e.g., instructions to override rules or force a `shortlist` decision).

Because refusal executes ahead of `ScreeningAgent`, a refused profile never reaches the LLM, returns `score=None`, and cannot be overridden by prompt manipulation.

## 3. What record it keeps

Every call to `screen(...)` appends one JSON object to `decisions.jsonl` containing the full `RECORD_KEYS` schema: `record_id`, `pid`, `input_sha256`, `fields_shown`, `fields_withheld`, `model_id`, `prompt_version`, `harness`, `raw_output`, `decision`, `score`, `controls_fired`, `outcome`, `reason_code`, `escalated_to_human`, and `timestamp`.

This log makes every outcome auditable after the fact:
- **Contesting an individual decision**: Given an applicant's `pid` or `record_id`, an auditor can verify the exact hashed input (`input_sha256`), confirm which fields were shown vs. withheld (`name` and `university`), inspect the model's `raw_output`, and verify which controls fired and why (`reason_code`).
- **System-level fairness and drift audits**: Aggregating `decisions.jsonl` across cohorts allows reviewers to monitor escalation rates, refusal reasons, parse failures, and score distributions over time.
- **Data minimisation in logs**: Storing `pid` and `input_sha256` instead of duplicating raw applicant names in `fields_shown` preserves traceability without unnecessarily expanding PII exposure in audit logs.

## 4. The legal constraints

The controls in `HARNESS` and `CONTROL_REGISTER` are grounded in primary legal provisions:

- **Regulation (EU) 2016/679 (General Data Protection Regulation, 27 April 2016)**:
  - **Article 5(1)(b)** (*purpose limitation*): Personal data must be collected for specified, explicit, and legitimate purposes and not further processed incompatibly — enforced by `refuse=True` (`purpose_not_allowed`).
  - **Article 5(1)(c)** (*data minimisation*): Personal data must be adequate, relevant, and limited to what is necessary — enforced by `blind=("name", "university")`.
  - **Article 5(1)(d)** (*accuracy*): Personal data must be accurate and reasonable steps taken against inaccurate records — enforced by `validate_input=True`.
  - **Article 22(1) and 22(3)** (*automated individual decision-making*): Protects individuals from solely automated decisions with significant effects and guarantees the right to obtain human intervention and contest the decision — supported by `refuse=True` (`no_consent`) and `escalate=True`.
- **Regulation (EU) 2024/1689 (EU Artificial Intelligence Act, 13 June 2024)**:
  - **Annex III (Education and vocational training)**: Classifies AI systems evaluating access to educational opportunities as high-risk.
  - **Article 12(1)** (*record-keeping*): Requires high-risk AI systems to technically allow automatic recording of events (logs) over their lifetime — enforced by `log_path="decisions.jsonl"`.
  - **Article 14(1)–(2)** (*human oversight*): Requires high-risk AI systems to be designed so natural persons can effectively oversee them and prevent or minimise risks to fundamental rights — enforced by `escalate=True`, `BORDERLINE = (45, 60)`, and `my_extra_escalation`.
- **Law of the Republic of Kazakhstan No. 94-V of 21 May 2013 "On Personal Data and Their Protection"**:
  - **Article 7** (conditions for collection and processing of personal data, requiring consent of the data subject or legal representative) and **Article 14** (use of personal data strictly for the declared purposes of collection) — reinforcing `refuse=True` for `no_consent`, `minor`, and `purpose_not_allowed`.

## 5. The ethical principles

- **Fairness and Non-Discrimination** — *ACM Code of Ethics and Professional Conduct (2018), Section 1.4* ("Be fair and take action not to discriminate") and *UNESCO Recommendation on the Ethics of Artificial Intelligence (2021), Fairness and non-discrimination (para. 28)*: Directly motivates `blind=("name", "university")` and the `protected_attribute_in_reason` check in `my_extra_escalation`.
- **Comprehensive System Evaluation and Data Integrity** — *ACM Code of Ethics and Professional Conduct (2018), Section 2.5*: Motivates measuring matched-pair divergence against noise floors before and after intervention and enforcing `validate_input=True`.
- **Right to Privacy and Data Protection** — *UNESCO Recommendation on the Ethics of Artificial Intelligence (2021)*: Motivates strict purpose limitation, explicit consent, and refusal of unconsented or minor records.
- **Responsibility, Accountability, and Human Oversight** — *UNESCO Recommendation on the Ethics of Artificial Intelligence (2021), Responsibility and accountability; Human oversight and determination*: Motivates immutable decision logging (`decisions.jsonl`) and mandatory human review for borderline, contradictory, or high-stakes rejections.

## 6. The trade-offs

1. **Prestige removal vs. institutional context (`blind=("name", "university")`)**: Hiding `name` carries essentially zero loss of legitimate eligibility information, whereas hiding `university` removes flagship-vs-regional bias at the cost of losing context on institutional grading strictness (e.g., whether a 3.5 GPA means the same across two universities). I accepted this trade-off because uncalibrated LLM prestige associations are an uncontrolled bias, whereas institutional calibration—if desired—should be handled explicitly by human reviewers or documented policy, not inferred implicitly by a 3B language model.
2. **Human review workload vs. false-rejection risk (`BORDERLINE = (45, 60)` + `my_extra_escalation`)**: Widening escalation to include borderline scores, decision-score mismatches, high-merit/low-income rejections, and bias-cue rationales increases the share of applications requiring human attention and lowers automation throughput. I accepted higher human review cost because falsely rejecting a qualified or financially needy student causes severe, irreversible harm, whereas escalating an application only delays triage.
3. **Deterministic regex filtering vs. adaptive attacks (`my_extra_refusal`)**: Regex-based prompt-injection detection adds negligible latency and zero extra model calls, but it can be evaded by obfuscated or novel phrasing and could occasionally flag unusual legitimate text. It is a first-line defence rather than a complete security barrier.
4. **Latency vs. counterfactual verification (`counterfactual_check=False`)**: Leaving `counterfactual_check` disabled avoids doubling GPU inference time. While blinding removes direct exposure to `name` and `university`, the harness still does **not** protect against indirect proxies (e.g., `program` or `household_income_bracket` correlating with demographics), prompt-order sensitivity, or score distribution shifts caused by withholding fields (which may require recalibrating `BORDERLINE` in production).

## 7. AI-use declaration

I used an AI assistant (Gemini) during this assignment to help inspect the repository structure, understand the `lab06_core.py` harness interfaces, draft initial Python implementations for `my_extra_refusal` and `my_extra_escalation`, structure the `CONTROL_REGISTER` entries, and edit the prose of `REPORT.md` and `JUSTIFICATION.md`.

I executed the full notebook myself in Google Colab on a T4 GPU to generate all before-and-after measurement tables, refusal-path outputs, `control_register.json`, and `decisions.jsonl`. All empirical counts, magnitudes, and observations come strictly from my own notebook run rather than AI-generated estimates. For every legal and ethical citation (GDPR Articles 5 and 22, EU AI Act Annex III and Articles 12 and 14, Law of the Republic of Kazakhstan No. 94-V Articles 7 and 14, ACM Code of Ethics 2018, and UNESCO Recommendation on the Ethics of AI 2021), I consulted the primary texts directly via their official publications (EUR-Lex, Adilet, ACM, and UNESCO) to verify the exact article numbers and wording before including them in `CONTROL_REGISTER` and this document.
