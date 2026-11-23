# Weekly CRM Reporting Agents: Functional Specification

Version 0.1 draft · 23 Nov 2026 · Vivek and Claude

When one of us clicks "Generate weekly report" inside our Custom CRM, nine agents run in a fixed order. They read the CRM, calculate the numbers with code, write about them, build a PowerPoint deck and update a Google Sheet, check both, and send them. There is no schedule or timer: the agents run only when asked, and each report covers one week of data. This spec says what each step takes in, does and gives out, and what happens when something breaks. It follows the BRD and is written by the two of us, Vivek and Claude.

The live version with diagrams is the Claude Doc "SPEC: Weekly CRM Reporting Agents". This file is a text copy.

---

## How to read this spec

Every section ends with a confidence score from 0 to 100, the same scale as the BRD.

| Score | Meaning |
|---|---|
| 90–100 | Comes straight from the BRD or is a hard rule we chose |
| 60–89 | A sensible design choice that still needs our yes |
| Below 60 | A guess; it depends on something we have not checked yet, usually the CRM schema |

One safety rule runs through every step: an agent never writes a number it made up. Every number in the deck and the sheet comes from the Metrics step's numbers file. The QA step checks this before anything is sent.

*Confidence: 95 / 100. The scale and the rule were agreed in the BRD.*

## Overview

```
            ┌──────────────────────── Orchestrator ────────────────────────┐
            │ Runs the steps in order, moves files, checks each output, logs│
            │                                                               │
            │  1 Start run → 2 CRM Data → 3 Metrics → 4 Analyst             │
            │  (code)        (code)       (code)       (AI + check)         │
            │                                              │                │
            │       ┌──────────────────────────────────────┘                │
            │       ▼          ┌→ 6 PPT Builder ─┐                          │
            │  5 Writer ───────┤   (code)        ├→ 8 QA ──→ 9 Distribution │
            │  (AI + check)    └→ 7 Sheets Builder┘  (code,    (code; after │
            │                       (code)            then AI)  optional OK)│
            │                                     fail: stop, alert us      │
            └───────────────────────────────────────────────────────────────┘
```

Steps 6 and 7 run side by side once the Writer is done; QA waits for both and is the only gate to delivery. Steps 1, 2, 3, 6, 7 and 9 are code; steps 4 and 5 are AI with a code check; QA is code first, then AI.

*Confidence: 80 / 100. This is the agent line from the BRD. Whether Analyst and Writer merge is still open.*

## Steps 1 to 3: start, read, calculate

Each step is owned by one agent. "Code" means a normal program the agent calls as a tool; "AI" means the agent's own judgement.

### Step 1. Start the run (Orchestrator)

| | |
|---|---|
| Gets | A "Generate weekly report" request from inside our CRM, with the week chosen (last full week by default); the workflow list of steps |
| Does | 1. Takes the chosen week: Monday 00:00 to Sunday 23:59 in our time zone. 2. Gives the run an ID, for example 2026-W48. 3. Checks no other run for that week is going. 4. Creates the run folder and the run log. |
| Gives | run_config file (week start, week end, run ID, time zone, recipients) |
| AI or code | Code |
| Done when | The run folder exists and the run is marked started in the log |
| If it fails | A second run for the same week is refused; the first one keeps going |

### Step 2. Read the CRM (CRM Data agent)

| | |
|---|---|
| Gets | run_config; read-only CRM access |
| Does | 1. Tests the connection. 2. Pulls deals, leads, accounts, activities, users, stages and targets for the week, plus the current state of every open deal. 3. Saves everything as one frozen weekly snapshot. 4. Writes a short count sheet: how many records of each kind it got. |
| Gives | The weekly snapshot and the count sheet |
| AI or code | Code |
| Done when | Every kind of data is saved and the counts are recorded |
| If it fails | Retries three times with a wait between. Still failing: the run stops. Nothing later runs on half the data. |

### Step 3. Check and calculate (Metrics agent)

| | |
|---|---|
| Gets | The weekly snapshot; last week's snapshot; the agreed list of numbers (the metric list) |
| Does | 1. Runs data checks: missing values, duplicates, deals with no owner, dates in the future, stage names we do not recognise. 2. Sorts each problem as block or warn. 3. Calculates every number on the metric list with code, for this week and last week, and the change. 4. Keeps 13 weeks of history for trends. |
| Gives | The numbers file and the data-quality report |
| AI or code | Code only. No AI touches any number. |
| Done when | Every metric on the list has a value or a stated reason it has none |
| If it fails | A block problem (for example no deals at all) stops the run. A warn problem goes into the deck's data note and the sheet's data-quality tab. |

*Confidence: 70 / 100. The shape of these steps is solid. How step 2 reads our CRM (database query, internal API or export) and which data checks matter depend on our schema, which we have not mapped yet.*

## Steps 4 to 5: analyse and write

These are the two steps where AI does real work. Both may only use numbers that are already in the numbers file.

### Step 4. Analyse (Analyst agent)

| | |
|---|---|
| Gets | The numbers file; the data-quality report; the metric list with plain definitions |
| Does | 1. Finds the three biggest good changes and the three biggest bad ones. 2. Spots trends over the 13 weeks. 3. Flags risks: stuck deals, overdue deals, falling activity. 4. Writes a one-line headline for the week. 5. Tags every point with the metric it is about, for example "win_rate". |
| Gives | An insights file: headline, highlights, lowlights, trends, risks; each point names its metric |
| AI or code | AI, then a code check |
| Done when | The insights file passes the format check and every metric it names exists in the numbers file |
| If it fails | Wrong format or a metric that does not exist: asked again, up to two more times. Still wrong: the run stops for a person to look. |

### Step 5. Write (Writer agent)

| | |
|---|---|
| Gets | The insights file; the numbers file; the slide list |
| Does | 1. Writes the one-slide summary of the week in plain English. 2. Writes a title and a one-line takeaway for each chart slide. 3. Writes speaker notes. 4. Writes the text for the sheet's summary tab. 5. Writes numbers only as placeholders, for example {win_rate}; code fills in the real value later. |
| Gives | A text file with every piece of slide and summary text |
| AI or code | AI, then a code check |
| Done when | Every slide has its text, every placeholder points at a real metric, and there are no bare numbers typed by the AI |
| If it fails | Same as step 4: two more tries, then stop for a person |

Why placeholders: if the Writer types "win rate rose to 34%" it could be wrong. If it types "win rate rose to {win_rate}", code fills in the true value and cannot get it wrong.

*Confidence: 80 / 100. Placeholders plus a code check is a strong guard against invented numbers. Whether Analyst and Writer stay as two agents or merge into one is open.*

## Steps 6 to 9: build, update, check, deliver

Steps 6 and 7 run side by side. Step 8 waits for both.

### Step 6. Build the deck (PPT Builder agent)

| | |
|---|---|
| Gets | The numbers file; the Writer's text; our slide template |
| Does | 1. Opens the template. 2. Fills every placeholder with the real value from the numbers file. 3. Draws each chart from the numbers file with code. 4. Adds a small data note on the last slide if step 3 raised warnings. 5. Saves the deck as run-ID.pptx. |
| Gives | The PowerPoint deck |
| AI or code | Code |
| Done when | The file opens, has the planned number of slides, and no placeholder is left unfilled |
| If it fails | Stops and names the missing item. A half-built deck is never passed on. |

### Step 7. Update the sheet (Sheets Builder agent)

| | |
|---|---|
| Gets | The numbers file; the Writer's summary text; the link to our one weekly Google Sheet |
| Does | 1. Writes this week's row into the history tab, replacing it if the week already exists. 2. Refreshes the summary, pipeline, team and activity tabs. 3. Writes the data-quality notes into their tab. 4. Keeps formulas and colour rules in place. |
| Gives | The updated Google Sheet and its link |
| AI or code | Code |
| Done when | Each tab has the expected rows and the week appears exactly once in history |
| If it fails | Re-running is always safe: a week is never written twice. A permission or quota error stops the step and is reported. |

### Step 8. Check everything (QA agent)

| | |
|---|---|
| Gets | The deck; the sheet; the numbers file; the insights and text files |
| Does | 1. Code checks: every number in the deck and sheet matches the numbers file; the deck and the sheet agree; no empty slide or tab. 2. AI checks: the text reads sensibly, does not contradict itself, and says nothing the numbers do not support. 3. Gives a result. |
| Gives | A QA report: pass, pass with warnings, or fail, with reasons |
| AI or code | Code first, then AI |
| Done when | The QA report is written |
| If it fails | A fail stops the run before delivery and alerts us with the reasons |

### Step 9. Deliver (Distribution agent)

| | |
|---|---|
| Gets | A QA pass, or a pass with warnings that a person has approved; the recipient list |
| Does | 1. If approval is switched on, waits for one of us to approve. 2. Saves the deck to the shared Drive folder. 3. Sends the deck and the sheet link to each recipient. 4. Sends us a short run summary. |
| Gives | Delivery receipts and the run summary |
| AI or code | Code |
| Done when | Every recipient is marked sent or failed in the log |
| If it fails | Retries each failed send three times, then lists who did not receive it |

*Confidence: 75 / 100. The steps are clear. The slide count, the sheet's tabs, the delivery channel and whether approval is on are still open questions from the BRD.*

## Run status

```
Running (one state at a time, recorded in the run log)
  Started → Reading CRM → Calculating → Analysing → Writing
  → Building → QA check → Awaiting OK (only if approval is on) → Sending → Done

Any step fails → Stopped (alert sent, files kept) → re-run from the last good step
```

At any moment a run is in one state, and the run log records every change with the time. "Building" covers steps 6 and 7 running side by side. A stopped run keeps every file it made, so a re-run picks up where it broke instead of starting over.

*Confidence: 85 / 100. The states follow the nine steps one to one. Whether the approval state is used depends on the approval question.*

## How a run starts, inside our CRM

There is no cron job, timer or schedule. The agents sit behind our Custom CRM and run only when someone asks.

| Rule | What it means |
|---|---|
| Trigger | A "Generate weekly report" button in our CRM. Clicking it sends one request to the Orchestrator. |
| Which week | The person picks a week. The default is the last full week, Monday to Sunday. |
| Data cut-off | Sunday 23:59 of the chosen week. Anything changed after that is not in this report. |
| Status in the CRM | While it runs, the CRM page shows the current state (see Run status). When done, it shows links to the deck and the sheet. |
| Who can run it | Only CRM users we allow; the CRM's own login decides who sees the button. |
| Time limit | A run should finish within 60 minutes. Past that, it stops and alerts us. |
| One run per week at a time | The run ID is the week, for example 2026-W48. A second click for the same week while it runs is refused. |
| Re-run | Clicking again for a finished or stopped week re-runs it from the last step that finished, using the saved files, and never duplicates anything in the sheet. |
| Past weeks | Any past week can be chosen, as long as its snapshot was saved or the CRM still holds that week's data. |

*Confidence: 75 / 100. On-demand from inside the CRM is what we decided. Where the button sits in the CRM, and whether the CRM calls the agents directly or through a small service, is a design choice for the architecture.*

## Permissions

Each part of the system gets only the access it needs, and nothing more.

| Who or what | Can do | Cannot do |
|---|---|---|
| CRM Data agent | Read the CRM, using its own read-only login | Change, add or delete anything in the CRM |
| Sheets Builder agent | Edit our one weekly Google Sheet, through a robot Google account | Open any other file in our Google Drive |
| Distribution agent | Save to one Drive folder; send from one sending address | Send to anyone not on the recipient list |
| Analyst, Writer, QA agents | Read the run's files | Reach the CRM, Google or email directly |
| Vivek and Claude | Start a run, approve delivery, change settings and templates | — |

Every run, approval and settings change is written to a log with who did it and when.

*Confidence: 80 / 100. Least access per agent is a firm rule. The exact Google and email setup depends on our Workspace, still to confirm.*

## Data security

- **Passwords and keys** live in a secrets store. They never appear in code, in a prompt to an agent, or in a log.
- **The AI sees summaries, not customers.** The Analyst, Writer and QA agents get the numbers file and totals, not customer names or contact details. Per-person sales figures are the one exception, because the team view needs them.
- **Only the columns we need** are read from the CRM. Anything not on the data list is not pulled.
- **Weekly copies** of the CRM data are kept for a set time, then deleted. How long is still to decide.
- **Everything travels encrypted** and is stored encrypted.
- **Logs** record what happened, never passwords or customer details.

*Confidence: 75 / 100. These are firm rules. The retention period and whether per-person figures may go to the AI are still to confirm.*

## When something fails

The golden rule: never send a wrong or half-finished report. If in doubt, stop and tell us.

| What goes wrong | How we notice | What the system does | What we do |
|---|---|---|---|
| CRM cannot be reached | Connection test fails | Retries 3 times, then stops | Fix access, re-run |
| Only part of the data came back | Counts look wrong vs last week | Stops before calculating | Check the CRM, re-run |
| Data check finds a block problem | Metrics step | Stops; writes the data-quality report | Fix the data, re-run |
| Data check finds a small problem | Metrics step | Carries on; adds a note to the deck and sheet | Nothing, or fix later |
| A calculation crashes | Metrics step error | Stops | Fix the metric, re-run |
| AI is slow or busy | No answer within the time limit | Waits and tries again up to 3 times | Re-run if still failing |
| AI gives the wrong format or an unknown metric | Code check after steps 4 and 5 | Asks again up to 2 more times, then stops | Look at the AI output, adjust its instructions |
| A number in the deck or sheet does not match | QA step | Fails QA; nothing is sent | Find the cause, re-run |
| Google refuses (permission or limit) | Sheets or Drive error | Waits and tries again, then stops | Fix access, re-run |
| Email or message not delivered | Send error | Retries 3 times; lists who missed it | Resend to those people |

A re-run always starts from the last step that finished cleanly, using the files already saved. Nothing earlier is redone.

*Confidence: 85 / 100. Failure handling covers every step. The retry counts and time limits are starting values we can tune.*

## The files passed between agents

Agents never talk to each other. Each one reads files and writes a file, and every file is kept in the run's folder. That is how a re-run can start halfway, and how we can see exactly what each agent did.

| File | Written by | Read by | What is in it |
|---|---|---|---|
| run_config | Orchestrator | Every agent | Week start and end, run ID, time zone, recipients |
| Weekly snapshot | CRM Data | Metrics | The frozen CRM data for the week |
| Count sheet | CRM Data | Metrics, QA | How many records of each kind were read |
| Numbers file | Metrics | Analyst, Writer, PPT Builder, Sheets Builder, QA | Every metric: ID, name, this week, last week, change, 13-week history |
| Data-quality report | Metrics | Analyst, PPT Builder, Sheets Builder | Each problem found, block or warn |
| Insights file | Analyst | Writer, QA | Headline, highlights, lowlights, trends, risks, each tagged with a metric ID |
| Text file | Writer | PPT Builder, Sheets Builder, QA | Slide titles, takeaways, summary, speaker notes, with number placeholders |
| Deck | PPT Builder | QA, Distribution | The finished PowerPoint |
| Sheet update record | Sheets Builder | QA, Distribution | Sheet link and what was written where |
| QA report | QA | Distribution, us | Pass, pass with warnings, or fail, with reasons |
| Run log | Orchestrator | Us | Every step: start, end, result, tries, time taken, AI cost |

A step counts as done only when its file exists and passes a format check. The Orchestrator then starts the next step.

*Confidence: 85 / 100. This file chain follows directly from the BRD's rule that agents pass files, not chat. Exact file formats are a design detail for the low-level design.*

## Settings we can change without code

These live in plain files we edit and keep under version control. Changing one does not mean rebuilding an agent.

| Setting | Example of a change |
|---|---|
| Workflow list | Add a Translation step after Writer; switch the approval step on or off |
| Metric list | Add "average discount"; remove a number nobody reads. Each metric has an ID, a plain definition, a formula and a test. |
| Data checks | Make "deal with no owner" a block instead of a warn |
| Slide template | New branding, a new slide, a different chart |
| Sheet layout | A new tab, a new column |
| Recipients | Add or remove a person; change email to Slack |
| CRM button | Rename the button, or limit which CRM users can see it |
| Agent instructions | Reword what the Analyst looks for; stored with each agent in the agent library |

Adding a new metric means: one entry in the metric list, plus one test that checks its value on sample data.

*Confidence: 80 / 100. Keeping these as editable settings is what makes agents and the workflow reusable, as the BRD asks. Which ones need code changes in practice will show in the build.*

## How we test it

1. **Sample data.** We make a small fake CRM week where we know every right answer by hand.
2. **Each metric.** A test checks each number against the hand-worked answer.
3. **Each agent alone.** Feed it a known input file, check its output file.
4. **The AI agents.** Run Analyst and Writer on the sample many times. Count how often they cite a metric that does not exist or type a bare number. Target: zero after the code check.
5. **Whole run.** Run all nine steps on the sample and compare the deck and sheet to what we expect.
6. **Failure drills.** Break things on purpose: cut the CRM connection, corrupt a file, revoke Google access. Check the system stops and alerts as section "When something fails" says.
7. **Re-run.** Run the same week twice. The sheet must still show the week once.
8. **Shadow weeks.** Run it on real data for two to four weeks alongside the manual report, and compare before we rely on it.

*Confidence: 85 / 100. Standard testing for a pipeline like this. The number of shadow weeks is our choice to make.*

## Open questions for the spec

These come on top of the BRD's questions, and each one changes a step above.

1. Step 2: what is the read-only way into our CRM — database query, internal API or export?
2. Step 2: does our CRM keep a history of stage changes? If not, "deals that moved" only works from the second week we run.
3. Step 3: which data checks should block the run, and which should only warn?
4. Steps 4 and 5: keep Analyst and Writer as two agents, or merge them into one?
5. Step 6: how many slides, and do we have a slide template to start from?
6. Step 7: one Google Sheet forever, or a new sheet each year?
7. Step 9: is the approval step on at the start? Who approves when one of us is away?
8. Step 9: email, Slack, Teams, or more than one?
9. Where does the system run: our own server or the cloud? (This decides how our CRM calls the agents.)
10. Security: may per-person sales figures go to the AI, or only team totals?

*Confidence: 90 / 100. These are the gaps we know about.*

## What comes next

1. Answer the open questions here and in the BRD.
2. Map our CRM fields to the data list (BRD section 5).
3. Write the architecture: the high-level design (how the parts fit, where it runs) and the low-level design (each agent's instructions, tools and file formats).
4. Build in Claude Code, one agent at a time, starting with CRM Data and Metrics on sample data.

*Confidence: 90 / 100. This is the order we agreed in the BRD.*
