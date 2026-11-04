---
title: Clean-Room Implementation Questions & Answers
section: 00-questions
version: 0.1.0
status: ratified
circumvention: false
---

# Clean-Room Implementation Q&A Log

This document serves as the formal bidirectional communication channel between independent clean-room software implementers and specification authors.

When developing software against `spec/`, implementers will inevitably encounter edge cases, ambiguities, or gaps in the specification. This log provides a controlled, audited mechanism for resolving those questions without risking intellectual property contamination.

---

## 1. Clean-Room Inquiry Principles & Workflow

```
+---------------------------+
|   Clean Implementer       |
|  - Reads spec/            |
|  - Encounters gap/issue   |
+---------------------------+
              |
              | 1. Logs question (Status: Open)
              v
+---------------------------+
|     spec/questions.md     |
+---------------------------+
              |
              | 2. Protocol research & verification
              v
+-------------------------------+
|     Specification Team        |
|  - Consults hardware/traces   |
|  - Synthesizes clean fix      |
+---------------------------+
              |
              | 3. Edits spec/*.md & updates spec/questions.md
              v
+------------------------------------------------------+
|  Specification Update (spec/NN-*.md version bump)    |
|  + Answer logged in spec/questions.md (Answered)      |
+------------------------------------------------------+
```

### 1.1 Submission Rules for Implementers

1. **Grounded in Specification Text:**
   Questions MUST reference specific sections, tables, or requirements within `spec/`.
2. **Strictly Black-Box Framing:**
   Questions MUST be framed in terms of observed external behavior, data formats, state transitions, or protocol requirements.
3. **No Prohibited Inquiries:**
   Implementers MUST NOT ask:
   - "How did the vendor implement this function?"
   - "What are the decompiler variable or function names?"
   - "Can you provide the cryptographic key or certificate for this feature?"
   Inquiries requesting proprietary vendor assets or source code will be marked `Rejected`.

### 1.2 Resolution Rules for Specification Authors

1. **Spec Update First:**
   Clarifications MUST NOT be communicated via out-of-band private chat, email, or unrecorded verbal exchanges. The primary answer to any valid question is an edit and version bump to the relevant `spec/*.md` document.
2. **Clean-Room Answer Synthesis:**
   Answers recorded in this file MUST be expressed using RFC 2119 normative terms, clean descriptive names, and observable protocol facts. Answers MUST NOT contain decompiler output, vendor symbols, or proprietary code snippets.
3. **Traceability:**
   Each resolved entry MUST cite the specific commit hash or document version where the specification was amended.

---

## 2. Question Entry Schema

Each question submitted to this log MUST use the following template:

```markdown
### `Q-###`: [Brief Summary of Issue]

- **Date:** YYYY-MM-DD
- **Requester:** [Implementer Identifier]
- **Spec Section:** [`spec/NN-section.md`](NN-section.md)
- **Status:** Open | Answered | Rejected
- **Question:**
  > [Detailed description of the ambiguity, missing field behavior, or error handling question.]
- **Answer:**
  [Concise clean explanation of the resolution. If Rejected, explanation of why inquiry violates clean-room guidelines.]
- **Resolved In:** [Spec Section version / commit SHA]
```

### Field Definitions

| Field | Type | Description |
|---|---|---|
| **Question ID** | `string` | Sequential identifier prefixed with `Q-` (e.g., `Q-001`, `Q-002`). |
| **Date** | `ISO 8601 Date` | Date on which the question was logged (`YYYY-MM-DD`). |
| **Requester** | `string` | Name or handle of the clean-room implementer posing the question. |
| **Spec Section** | `string / link` | Link to the relevant specification file under `spec/`. |
| **Status** | `enum` | Current state of the question: <br>• `Open`: Awaiting research investigation.<br>• `Answered`: Resolved via a clean-room specification update.<br>• `Rejected`: Question violates clean-room rules or is out of scope. |
| **Question** | `markdown block` | Clear statement of the problem, referencing specific lines or tables. |
| **Answer** | `markdown text` | Clean-room synthesis explaining the behavior or rationale. |
| **Resolved In** | `string` | Target spec section version and/or git commit SHA that incorporated the fix. |

---

## 3. Question Index

<!--
Log active questions below. Keep the index sorted by ID.
-->

| ID | Date | Section | Requester | Status | Summary |
|---|---|---|---|---|---|
| *(none yet)* | — | — | — | — | No questions logged yet. |

---

## 4. Question Log

*(No questions currently active. When implementers encounter ambiguities or gaps in `spec/`, follow the submission template below to log inquiries.)*

---

### Non-Normative Example Entry (Illustration Only)

```markdown
### `Q-000`: [Example Question Title] (Non-Normative)

- **Date:** 2026-10-23
- **Requester:** ExampleImplementer
- **Spec Section:** `spec/20-upload-manifest.md`
- **Status:** Answered
- **Question:**
  > Does the manifest parser tolerate unknown key-value pairs?
- **Answer:**
  [Clean specification clarification explaining observable controller behavior without citing decompiler output.]
- **Resolved In:** `spec/20-upload-manifest.md` v0.1.1 (commit abc1234)
```

## 5. Question Submission Template

<!--
To log a new question, increment the Question ID, add a row to the Question Index table above, and append your question below using this template:

### `Q-###`: [Short Title]

- **Date:** 2026-MM-DD
- **Requester:** [Your Name / Handle]
- **Spec Section:** [`spec/NN-section.md`](NN-section.md)
- **Status:** Open
- **Question:**
  > [Enter your specific question regarding the specification text or behavior here.]
- **Answer:**
  *Pending investigation by research team.*
- **Resolved In:** *Unresolved*
-->
