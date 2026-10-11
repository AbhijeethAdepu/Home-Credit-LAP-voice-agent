# Home Credit LAP Voice Agent

An AI voice agent that phones existing Home Credit customers about a pre-approved **Loan Against Property (LAP)** offer of up to **₹75 lakh**, runs a short preliminary eligibility check, and hands suitable customers to a senior loan expert. Every call is automatically evaluated and logged through an n8n workflow.

> **Scope:** This is preliminary screening only. The agent never approves a loan and never quotes interest rates, fees, or timelines. Final decisions rest with the loan expert.

---

## Table of Contents

1. [Highlights](#highlights)
2. [Architecture](#architecture)
3. [Call Flow](#call-flow)
4. [Eligibility Rules](#eligibility-rules)
5. [Exit Paths](#exit-paths)
6. [Prompt Design](#prompt-design)
7. [Voice and Turn-Taking Tuning](#voice-and-turn-taking-tuning)
8. [Call Evaluation Workflow (n8n)](#call-evaluation-workflow-n8n)
9. [Test Scenarios](#test-scenarios)
10. [Setup](#setup)
11. [Repository Structure](#repository-structure)
12. [Limitations](#limitations)
13. [Evidence](#evidence)

---

## Highlights

- **Natural, professional voice conversation** with one question per turn and a consistent tone from greeting to close.
- **Identity verification first.** No offer or financial detail is shared until the right person is confirmed.
- **Seven-item eligibility checklist**, asked in a fixed order. Answers volunteered early or out of order are captured and never re-asked.
- **Instant disqualification** when a mandatory rule clearly fails, with a brief, neutral explanation.
- **Existing-loan and EMI-reduction detection** at any point in the call, routed to a loan-transfer specialist.
- **Loan cap handling.** Requests above ₹75 lakh get a counter-offer instead of a rejection.
- **Self-corrections handled.** The latest clear answer wins ("residential... actually agricultural").
- **Automated evaluation and logging** of every call into Google Sheets.

## Architecture

```
Customer phone
      │
      ▼
┌──────────────┐   call ended event   ┌────────────────────────┐
│  Retell AI   │ ───────────────────► │  n8n webhook           │
│  voice agent │                      │  Home Credit LAP -     │
│              │                      │  Retell Results        │
└──────────────┘                      └───────────┬────────────┘
                                                  │
                                    ┌─────────────▼─────────────┐
                                    │ Python Code node          │
                                    │ transcript parser +       │
                                    │ rule-based evaluator      │
                                    └─────────────┬─────────────┘
                                                  │
                                          ┌───────▼────────┐
                                          │ Google Sheets  │
                                          │ call log       │
                                          └────────────────┘
```

| Component | Role |
|---|---|
| **Retell AI** | Voice agent, speech handling, and conversation flow |
| **n8n** | Receives call-result webhooks and runs the evaluation |
| **Python (n8n Code node)** | Parses the transcript and checks it against the rules |
| **Google Sheets** | Stores call logs and evaluation results |

## Call Flow

1. **Verify identity.** "Hello, may I speak with the customer?" Nothing about the offer is revealed until confirmed.
2. **Check availability.** The agent introduces itself and asks for two minutes.
3. **Present the offer** and ask consent to begin a brief eligibility check.
4. **Collect seven checklist items** in order:

   | # | Item | Question (paraphrased) |
   |---|---|---|
   | 1 | Property type | What type of property is it? |
   | 2 | Ownership | Sole name or jointly owned? |
   | 3 | Original documents | Are the originals available for verification? |
   | 4 | Loan amount | How much loan are you looking for? |
   | 5 | Occupation and income mode | Salaried or self-employed, and does income come through a bank? |
   | 6 | Market value | Approximate current market value? |
   | 7 | Tenure | For how many years? |

5. **Handoff.** Only when all seven items are answered and every rule passes, the agent states that the customer meets the basic criteria and that a senior loan expert will follow up. It never says "approved" or "guaranteed".

After every customer reply the agent silently extracts all facts, checks for exit conditions, applies the rules, and asks only the earliest unanswered item.

## Eligibility Rules

| Criteria | Rule | Outcome if failed |
|---|---|---|
| Property type | Residential, commercial, or industrial | Agricultural → **Disqualified** |
| Ownership | Sole or joint | Never disqualified; unclear answers are clarified |
| Property documents | Originals available for verification | Photocopies only, lost, or unavailable → **Disqualified** |
| Loan amount | Up to ₹75 lakh | Above cap → counter-offer, not rejection |
| Occupation | Salaried or self-employed | Not a disqualifier |
| Income mode | Must mainly come through a bank | Cash only or mainly cash → **Disqualified** |
| Market value | Customer's estimate recorded | No minimum value |
| Tenure | 3 to 15 years, inclusive | Outside the range → **Disqualified** |
| Existing loan on this property | Detected at any point | Routed to **loan-transfer specialist** |

The agent never invents extra criteria, and approximate answers ("around", "roughly", "I think it's about") are accepted for amounts and tenure.

## Exit Paths

| Situation | Agent behavior |
|---|---|
| **Disqualified** | Stops asking questions, gives one neutral reason, closes politely. No exceptions or persuasion. |
| **Loan above ₹75 lakh** | States the maximum and asks whether to proceed with ₹75 lakh. Yes → continue. No → polite close, logged as `CUSTOMER_DECLINED`. |
| **Existing loan / EMI reduction** | Stops the checklist and routes to the loan-transfer specialist. |
| **Busy at any point** | No pitch. Asks for a callback time, confirms it, closes. |
| **Wrong person** | Asks for a callback time. Discloses nothing about the offer. |
| **Not interested / do not call** | Apologises briefly and ends the call without persuasion. |
| **Silence or poor audio** | Asks to repeat once, then closes after two failed attempts. |

Customers can interrupt, ask questions, or go off topic. The agent answers briefly from verified product context only, defers rates, fees, and timelines to the loan expert, and returns to the next unanswered item.

## Prompt Design

The prompt is split into two parts so each rule lives in exactly one place:

| Part | Contains |
|---|---|
| **Global Prompt** | Identity, verified product context, professional tone, turn-taking, speech and pronunciation rules, language, safety guardrails |
| **Welcome Node Prompt** | Call flow, checklist, eligibility rules, exit flows, turn procedure, examples |

Key design decisions:

- **Fixed tone.** Plain, level sentences, no exclamation marks, rare acknowledgments, one "thank you" per closing, and one-sentence closings.
- **Strict turn-taking.** The agent never sends two messages in a row, waits for unfinished answers, treats split replies as one answer, and stays silent after its closing line.
- **Spoken-friendly output.** Amounts and years are spoken in words, and "Loan Against Property" is never abbreviated.
- **No guessing.** Vague answers are clarified, and values the agent itself suggested are not treated as customer-confirmed.
- **Safety.** The agent never asks for OTP, PIN, passwords, card numbers, Aadhaar, PAN, or bank account numbers. It never invents rates, fees, or approval conditions.
- **Honest identity.** If asked, it says it is an AI assistant from Home Credit.

## Voice and Turn-Taking Tuning

Some behaviors are agent settings rather than prompt text. Settings used in this project:

| Setting | Approach |
|---|---|
| Responsiveness / response delay | Lowered so the agent waits for the customer to finish |
| Interruption sensitivity | Medium to low, so background noise and one-word replies do not trigger early replies |
| Backchanneling | Off, to avoid filler that sounds like cutting in |
| Silence reminder | Longer wait (about 8 to 10 seconds) |
| Voice | One fixed English (India) voice, speed slightly below default |
| Call end | The closing line triggers the end-call function, so the call ends right after it |

Exact setting names vary by platform version.

## Call Evaluation Workflow (n8n)

**Workflow:** `Home Credit LAP - Retell Results`

The workflow receives the call-ended event through a webhook, passes the transcript to a Python Code node, and appends the result to Google Sheets.

**What the evaluator does**

- Splits the transcript into customer and agent turns (handles different speaker labels, line endings, and Retell's `transcript_object` format).
- Uses agent text only for context, such as the question just asked, and never lets it set a customer value.
- Classifies each amount as **loan amount** or **property value** using cue words ("need" versus "worth") and the agent's last question, so a property value of one crore is never mistaken for a loan request.
- Reads the loan-cap decision only from the reply right after the agent's cap offer.
- Returns a result status, missing checklist items, and detected flags.

**Outputs**

| Field | Examples |
|---|---|
| Status | `REVIEW_REQUIRED`, `DISQUALIFIED`, `CUSTOMER_DECLINED`, `INCOMPLETE` |
| Flags | `LOAN_AMOUNT_ABOVE_LIMIT`, `TENURE_OUT_OF_RANGE`, `TRANSCRIPT_PARSE_ERROR`, `NO_CUSTOMER_UTTERANCES` |
| Missing items | Checklist items the call never covered |

Calls that pass all rules are marked for human review rather than auto-approved. The evaluator is version `v4` and lives in `evaluator/`.

## Test Scenarios

Each scenario was run as a recorded call using a written script.

| # | Scenario | Expected outcome |
|---|---|---|
| 1 | Eligible customer | All seven items collected, handoff to loan expert |
| 2 | Agricultural property | Immediate disqualification |
| 3 | Cash-based income | Immediate disqualification |
| 4 | Original documents unavailable (photocopies only) | Immediate disqualification |
| 5 | Existing loan or EMI reduction | Routed to loan-transfer specialist |
| 6 | Loan above ₹75 lakh, customer accepts cap | Continues, then handoff |
| 7 | Loan above ₹75 lakh, customer declines cap | Polite close, `CUSTOMER_DECLINED` |
| 8 | Tenure outside 3 to 15 years | Immediate disqualification |
| 9 | Busy customer and callback request | Callback time recorded, no pitch |
| 10 | Wrong person answers | No disclosure, callback requested |
| 11 | Self-correction mid-call | Latest clear answer is used |
| 12 | Answers volunteered out of order | Captured and not re-asked |
| 13 | Interruptions and off-topic questions | Brief answer, returns to checklist |
| 14 | Incomplete call or early hang-up | Logged as `INCOMPLETE` |

Detailed results, notes, and transcripts are in `test-results/` and the evidence folder below.

## Setup

1. **Retell AI**
   - Create a voice agent with an English (India) voice.
   - Paste the Global Prompt into the agent's global prompt field and the Welcome Node Prompt into the welcome node.
   - Set the begin message to: `Hello, may I speak with <customer name>?`
   - Apply the tuning settings above and connect the end-call function to the closing lines.
   - Point the call-ended webhook to your n8n webhook URL.
2. **n8n**
   - Import `workflows/home-credit-lap-retell-results.json`.
   - Paste `evaluator/evaluator_v4.py` into the Python Code node.
   - Confirm the transcript field name matches your webhook payload.
   - Connect your Google Sheets credentials and choose the target sheet.
3. **Test**
   - Run a short web call using one of the scripts in `test-scenarios/`.
   - Check the transcript in Retell, then confirm a new row in Google Sheets.

## Repository Structure

```text
Home-Credit-LAP-voice-agent/
├── README.md
├── .gitignore
├── System_Prompt/            # Global Prompt and Welcome Node Prompt
├── Voice_Agent_platform/     # Platform and agent configuration summary
├── Workflow/                 # n8n workflow export and evaluation logic
├── Test_Scenarios/           # Written call scripts for each test case
├── Call_Recordings/          # Audio recordings of the test calls
├── Transcripts/              # Transcripts of the test calls
└── Brief_Test_Results/       # Summary of test outcomes
```

## Limitations

- The agent performs preliminary screening, not final loan approval.
- Speech recognition errors can change how an answer is interpreted, so ambiguous or incomplete calls need manual review.
- The evaluator is keyword-based. It does not parse Hindi or Hinglish spoken numbers (such as "pachhattar lakh") or phrases like "half a crore" and "three and a half years".
- A range like "70 to 80 lakh" is read using only the last number that has a unit.
- A loan cap decision is read only from the reply right after the offer, so a later change of mind is not captured.
- "Mostly bank, some cash" is treated as unclear in the evaluator, even though the live agent asks a clarifying question.
- The agent speaks English only.
- The workflow depends on correctly configured webhook and Google Sheets credentials.

## Evidence

- **Call recordings, transcripts, and test results:** [https://drive.google.com/drive/folders/1pFCSsu_E0h5FYKze45LoHOIZZFkdtHn0?usp=sharing]
- **Demo video :** [https://drive.google.com/drive/folders/1qg_eb7ADiMGZkyjlKo95mSV-UcV8g-4F?usp=sharing]
