
This document explains every node in the workflow. Use it during setup and interview prep.

---

## Node Map

```
Watch New Submissions
    └── Normalize Submission Data
            └── Build GitHub Raw URL
                    ├── Fetch GitHub README ────────┐
                    └── Fetch GitHub Repo Metadata ──┘
                                                      └── Merge GitHub Data
                                                              └── Route Video URL
                                                                      └── Fetch Video Info
                                                                              └── Build Eval Prompts
                                                                                      └── Evaluate GitHub (Groq)
                                                                                              └── Evaluate Assignment Explanation (Groq)
                                                                                                      └── Final Holistic Evaluation (Groq)
                                                                                                              └── Parse and Merge Scores
                                                                                                                      └── Log to Evaluation Sheet
                                                                                                                              └── Route by Verdict (IF)
                                                                                                                                      ├── Send Recruiter Alert (true)
                                                                                                                                      └── Acknowledge Candidate (false + true)
```

---

## Node 1 — Watch New Submissions

**Type:** Google Sheets Trigger

**What it does:** Polls the `Raw Submissions` tab every 5 minutes. When Google Form writes a new row (a candidate submits), this node fires and starts the workflow.

**Why it exists:** This is the intake solution. Instead of parsing unstructured emails, all submissions come through a structured Google Form that auto-populates the sheet. Zero manual intervention.

**Configure:**
- Credential: Google Sheets OAuth2
- Spreadsheet: your `Eubrics Candidate Evaluations` sheet
- Sheet name: `Raw Submissions`
- Event: Row Added
- Poll interval: Every 5 minutes

**Output:** Raw row object with all form field values

---

## Node 2 — Normalize Submission Data

**Type:** Set

**What it does:** Renames the raw Google Form column headers (which include spaces and slashes) into clean camelCase variable names used throughout the rest of the workflow.

**Why it exists:** Google Form column names like `"Loom / YouTube Demo Video URL"` are impossible to reference cleanly in expressions. This node creates `videoUrl`, `githubUrl`, etc. once, so all downstream nodes use clean names.

**Output:** `candidateName`, `email`, `college`, `githubUrl`, `videoUrl`, `resumeDriveUrl`, `docsUrl`, `explanation`, `submittedAt`, `rowNumber`

---

## Node 3 — Build GitHub Raw URL

**Type:** Code (JavaScript)

**What it does:** Converts `https://github.com/USER/REPO` into two URLs:
- `githubRawUrl`: points to the raw README.md text
- `githubApiUrl`: points to the GitHub REST API for repo metadata

**Why it exists:** GitHub doesn't serve README content at the standard URL — you need the `raw.githubusercontent.com` URL. The GitHub API URL is different again. One code node handles both transformations cleanly.

---

## Nodes 4 & 5 — Fetch GitHub README + Fetch GitHub Repo Metadata

**Type:** HTTP Request (parallel)

**What they do:** Run simultaneously. Node 4 fetches the raw README.md text. Node 5 hits the GitHub API for metadata (stars, language, topics, description, last updated).

**Why parallel:** Saves time — both requests run at the same time instead of sequentially.

**Key setting:** `continueOnFail: true` — private repos or typo URLs won't crash the workflow.

---

## Node 6 — Merge GitHub Data

**Type:** Merge (by position)

**What it does:** Combines the two parallel branches back into one item containing both README content and repo metadata.

**Why it exists:** n8n parallel branches produce separate items. This node merges them so the evaluation nodes have everything in one place.

---

## Node 7 — Route Video URL

**Type:** Code (JavaScript)

**What it does:** Detects whether the video URL is YouTube or Loom. Builds the appropriate oEmbed API URL to fetch the video's title and description.

**Why it exists:** YouTube and Loom have different oEmbed endpoints. This router handles both without branching the workflow.

---

## Node 8 — Fetch Video Info

**Type:** HTTP Request

**What it does:** Calls the oEmbed URL to get the video's title, author name, and thumbnail. Free, no API key required.

**Why it exists:** We can't watch the video, but the title often reveals what the candidate built. A title like "AI Sales Roleplay Bot — Architecture Walkthrough" tells us a lot about the candidate's communication style and project scope.

---

## Node 9 — Build Eval Prompts

**Type:** Code (JavaScript)

**What it does:** Consolidates all fetched data from previous nodes into one clean object: README text, repo metadata fields, video title, and the candidate's explanation — all ready to inject into Groq prompts.

**Why it exists:** The Merge node and video fetch nodes store data in different shapes. This node flattens everything into named fields that can be directly referenced in prompt templates with `{{ $json.fieldName }}`.

---

## Node 10 — Evaluate GitHub

**Type:** HTTP Request → Groq API

**What it does:** Sends the README content and repo metadata to Groq (Llama 3 70B) with a structured prompt. Returns JSON scores for: readme quality, code structure, documentation, automation relevance, innovation, completeness.

**Credential:** Groq HTTP Header Auth (Authorization: Bearer YOUR_KEY)

**Model:** `llama3-70b-8192`

**Why separate from other evaluations:** Single-responsibility. This prompt focuses only on GitHub. A combined prompt would produce worse scores because the model's attention is split.

---

## Node 11 — Evaluate Assignment Explanation

**Type:** HTTP Request → Groq API

**What it does:** Scores the candidate's written explanation and video title for: communication quality, clarity, technical depth, architecture thinking.

**Why after GitHub eval:** The explanation and video require different evaluation criteria. Keeping it separate lets us tune the prompt independently.

---

## Node 12 — Final Holistic Evaluation

**Type:** HTTP Request → Groq API

**What it does:** The synthesiser. Receives ALL prior scores as input context. Produces the master evaluation: overall score (0–10), all 9 dimension subscores, verdict (Strong Hire / Hire / Maybe / Reject), top strengths, top weaknesses, recommendations, hiring summary, and interview focus areas.

**Why a third call:** The final verdict should be informed by everything we know. By passing all prior scores as context, the model reasons holistically rather than scoring in isolation.

---

## Node 13 — Parse and Merge Scores

**Type:** Code (JavaScript)

**What it does:**
1. Extracts the text response from each Groq API call's `choices[0].message.content`
2. Strips any markdown backticks (Groq sometimes wraps JSON in them)
3. Parses each JSON safely (try/catch — never crashes)
4. Merges all three score objects + base candidate data into one flat object

**Why it exists:** Groq returns JSON inside an API response envelope. This node unwraps it, cleans it, and makes it sheet-ready.

---

## Node 14 — Log to Evaluation Sheet

**Type:** Google Sheets → Append

**What it does:** Appends one new row to the `Evaluations` tab with all 22 columns filled: dates, candidate info, all scores, verdict, strengths, weaknesses, recommendations, summary.

**Configure:**
- Spreadsheet ID: copy from your Sheet URL
- Sheet name: `Evaluations`
- All column mappings are pre-configured in `workflow_export.json`

---

## Node 15 — Route by Verdict

**Type:** IF

**What it does:** Checks if `verdict` contains "Hire" (matches both "Strong Hire" and "Hire"). Routes true → recruiter alert. False → candidate acknowledgement only. True also flows to acknowledgement after alert.

---

## Node 16 — Send Recruiter Alert

**Type:** Gmail

**What it does:** Sends a rich HTML email to the recruiter with the full score breakdown, strengths, weaknesses, summary, and interview focus areas. Only fires for Strong Hire and Hire verdicts.

**Configure:** Replace `RECRUITER_EMAIL_HERE` with the actual email address.

---

## Node 17 — Acknowledge Candidate

**Type:** Gmail

**What it does:** Sends a polite confirmation email to the candidate. Fires for all verdicts — every candidate gets an acknowledgement.

