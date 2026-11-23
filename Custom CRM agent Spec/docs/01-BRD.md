# Weekly Business Reports from the CRM, Made Automatically
## Business Requirements Document (internal reference)

Version 0.4 draft · 23 Nov 2026 · For review

We want AI agents that fit into the Custom CRM our team built. When someone clicks a button in the CRM, they produce that week's PowerPoint deck and Google Sheet with no manual work, both showing the same numbers. There is no schedule; the agents run only when asked. A fixed line of AI agents does the work, each owning one step, and we can bring in new agents, keep the ones that work, and change the order of steps without rebuilding. The two of us, Vivek and Claude, are building this together; no one else is involved yet. This document is our working reference. It says what we need and why, in plain words, and is not a technical design: no libraries, hosting or prices here.

**Confidence scores:** every section ends with a score from 0 to 100. 90–100 means it comes straight from the requirement. 60–89 means it is a reasonable reading that needs a yes from you. Below 60 means it is a guess to be confirmed or replaced. Nothing in this document is confirmed with the CRM yet.

---

## 1. The problem today

Every week someone builds the business report by hand. It takes hours, and the result is often late or slightly wrong.

1. Open the CRM and pull out the week's numbers.
2. Copy them into a document and a slide deck.
3. Check them, fix mistakes, and send them around.

Each week's report looks a little different, so numbers are hard to compare. By the time leadership reads it, the data is already a few days old.

*Confidence: 60 / 100. We have not seen the current manual process. This is the usual pain; the real steps and hours need confirming.*

## 2. What we want to build

Agents inside our CRM that make the weekly report for us whenever we ask.

1. The system reads the week's data from the CRM.
2. It works out the numbers (new deals, wins, pipeline, activity).
3. It compares them with last week and spots what changed.
4. It writes a plain-English summary.
5. It builds two outputs: a slide deck and a Google Sheet.
6. It checks the deck and the sheet agree with each other.
7. It sends them to the right people.

Each step is owned by an agent (see section 3). The agents do the judgement work: spotting what went up or down, what is worth highlighting, and writing the summary. For anything exact, an agent calls a code tool: pulling data, adding up numbers, building the files.

One rule: the AI never invents a number. Every number in the report comes from a calculation the software did.

*Confidence: 85 / 100. The flow follows the requirement. The exact split between AI work and normal software is our proposal.*

## 3. Agents we can bring and keep

The system is an agentic workflow: a fixed line of agents, each owning one step, running in a set order. Every number comes from a code tool the agent calls, never from the agent's own head. We can add, swap or remove an agent without rebuilding the rest.

| Agent | Its one job | What it gets | What it gives | If it fails |
|---|---|---|---|---|
| Orchestrator | Runs the steps in order, passes files between agents, records status | The "Generate weekly report" click from our CRM, and the workflow list | A finished run, or a clear stop with the reason | Stops the run, alerts a named person, allows a re-run from the last good step |
| CRM Data | Reads this week's data from our CRM and freezes a copy | Read-only CRM access, week start and end | One frozen weekly snapshot of the data | Retries, then stops the run; nothing downstream runs on partial data |
| Metrics | Checks the data and calculates every number with code | The frozen snapshot, the agreed list of numbers | A numbers file: this week, last week, change | Flags bad data; blocks the run if a check fails hard, warns if minor |
| Analyst | Reads the numbers and finds what matters | The numbers file | Highlights, lowlights, trends, risks, each pointing at a number | Output checked against the numbers file; retried if it cites a number that is not there |
| Writer | Turns the analysis into plain-English slide text and a one-slide summary | Analyst output and the numbers file | Slide titles, takeaway lines, summary text | Same check as Analyst; a human can edit before send |
| PPT Builder | Fills the slide template with numbers, charts and the Writer's text | Numbers file, Writer text, slide template | The PowerPoint deck | Stops and reports the missing item; no half-built deck is sent |
| Sheets Builder | Adds this week's numbers to the Google Sheet and keeps the history | Numbers file, the sheet link | The updated Google Sheet | Re-running never duplicates a week; stops on a permission error |
| QA | Checks deck and sheet agree with the numbers file and read sensibly | Deck, sheet, numbers file | Pass, pass with warnings, or fail with reasons | A fail stops delivery and alerts a person |
| Distribution | Sends the deck and the sheet link to the right people | QA pass, recipient list | Delivery receipts and a short run summary | Retries, then alerts; logs who did and did not receive |

How the agents work together:

1. Each agent gets its input as a file from the previous step and writes its output as a file for the next. No agent talks to another agent directly; the Orchestrator moves the files.
2. A step is done only when its output file exists and passes a format check. Then the Orchestrator starts the next step.
3. Approval point: after QA, a person can review before Distribution runs. This can be switched off later if trust is high.
4. Every run is logged: what each agent received, did, returned, and how long it took.
5. Agents are kept as a library. The same CRM Data or QA agent can be reused in another project.
6. The order of steps is a written list. Adding a step (for example, a Translation agent) is adding one line and one agent.

Why a fixed order and not free-running agents: the report must look the same each time it is made. A fixed order with a QA agent gives consistency; it is not about saving cost.

*Confidence: 80 / 100. Agents as the theme, one job each, fixed order and a QA step are what the requirement asks for. The exact nine agents are our proposal; some may merge (for example Analyst and Writer) during design.*

## 4. The two outputs of each weekly report

Both must show the same numbers. The written weekly summary lives inside the deck (a summary slide) and in the sheet (a summary tab).

| Output | What it is | Best for |
|---|---|---|
| PowerPoint deck | 8–10 slides: a one-slide written summary of the week, then charts with one-line takeaways | Presenting and skimming |
| Google Spreadsheet | The numbers behind the deck, with history building up for every week we report on | Digging into the data |

*Confidence: 90 / 100. PPT and Google Sheets are the two outputs you named. Slide count and tab layout are our proposal.*

## 5. What we need from the CRM

Our own team built the CRM, so we can map each row below to exact fields ourselves from our schema. This lists the kind of information needed; the exact field mapping is an appendix we fill in before design.

| Kind of information | Why we need it |
|---|---|
| Deals: stage, value, owner, open date, close date, won or lost | Pipeline, revenue, win rate |
| Leads: when created, where from, what happened to them | New business coming in |
| Customers / accounts | New vs existing business |
| Activities: calls, meetings, emails logged, by whom and when | How active the team is |
| Sales team list: who belongs to which team or region | Per-person and per-team views |
| Stage names and what they mean | So the report uses the right words |
| Targets / quotas (may live outside the CRM) | Actual vs target |
| Reasons deals were lost (if recorded) | Why we are losing |

We also need to read the data the same way each time and keep a copy of every week we report on, so the report can say "up 12% from last week".

*Confidence: 65 / 100. These are the kinds of data a sales CRM normally holds, and we own the CRM so we can check quickly. Not yet checked against our schema.*

## 6. The numbers we will report

This is a starting list. We finalise the list together before design. Every number is shown next to last week's number.

| Group | Numbers |
|---|---|
| Pipeline | New leads this week; new deals this week; total open pipeline value by stage; deals that moved forward or backward |
| Results | Deals won (count and value); deals lost (count and value); win rate; average deal size; actual vs target |
| Speed | Average time to close a deal; deals stuck in the same stage too long |
| Activity | Calls, meetings and emails per person |
| Team | Results by person and by team or region |
| Warnings | Deals past their expected close date; deals with no activity for a while |

*Confidence: 50 / 100. Standard sales numbers. None are confirmed as wanted, and all depend on the CRM holding the data in section 5.*

## 7. How a report is made

There is no schedule, timer or cron job. A report is made only when someone asks for it from inside our CRM.

| Step | What happens |
|---|---|
| 1 | Someone clicks "Generate weekly report" in our CRM and picks a week (last full week by default). |
| 2 | The agents run: read the CRM, calculate, write, build the deck and the sheet, check them. |
| 3 | The CRM page shows progress, then links to the deck and the sheet. |
| 4 | Optional: one of us reviews and approves before it is sent. |
| 5 | The deck and the sheet link go to the agreed people. |

If something goes wrong (CRM data looks broken, a check fails), the agents stop, send nothing, and tell us what failed. Clicking again re-runs it once the problem is fixed.

*Confidence: 75 / 100. On-demand from inside the CRM is what we decided. Where the button sits, who may click it and whether approval is on are still open.*

## 8. What success looks like

- The report is ready within an hour of clicking the button, without anyone building it by hand.
- The numbers match the CRM exactly.
- The deck and the sheet agree with each other.
- The people who need the report get it without asking.
- The team gets its hours back.
- After a few weeks, the reviewer rarely needs to change anything.

*Confidence: 75 / 100. These follow from the goal of automatic weekly reporting. Exact targets (time saved, on-time rate) need numbers from you.*

## 9. What is not included (for now)

- Changing or fixing the CRM itself.
- Live, real-time dashboards.
- Forecasting or predictions.
- Reports for departments other than sales, unless the business asks.
- Schedules, timers, and daily or monthly reports. The agents run only when asked, and each report covers one week.

*Confidence: 70 / 100. Nothing here was asked for, so we left it out. Tell us if any of it should be in.*

## 10. What we are assuming (please confirm or correct)

| # | Assumption |
|---|---|
| A1 | The report is mainly about sales and pipeline. |
| A2 | The week runs Monday to Sunday. |
| A3 | One currency and one time zone. |
| A4 | Because our team owns the CRM, read-only access and a field list are available in-house. |
| A5 | The company uses Google Workspace (Drive, Sheets, Gmail). |
| A6 | Sending summarised numbers to the Claude AI service is allowed. The AI does not need raw customer details. |
| A7 | Targets, if any, can be given to us in a simple sheet. |

*Confidence: 45 / 100. By definition these are unconfirmed. Each one changes the design if wrong.*

## 11. Questions we need answered before design

Sorted into three groups so the blocking ones are clear.

**Group 1: decide together first (blocking; design cannot start without these)**
1. Which numbers matter most? Confirm or cut the list in section 6.
2. Who receives the deck and the sheet, and by which channel (email, Slack, Teams)?
3. Should a person approve before sending, or should it go out automatically?
4. Where in our CRM should the "Generate weekly report" button sit, and who may click it?
5. Is there an existing slide template or branding to match?
6. Who will use the deck and the sheet once it works?
7. Who should be alerted when a run fails?
8. Any rule against sending summarised numbers to an AI service?
9. Should the sheet keep history forever, or for a set number of weeks?

**Group 2: answer ourselves from our CRM (we own it)**
1. Which tables and fields hold deals, leads, accounts, activities, users and stages?
2. What is the best read-only way to pull a weekly snapshot (database query, internal API or export)?
3. Does the CRM keep history of stage changes, or only the current state?
4. How are deleted or merged records handled?
5. Roughly how many deals, leads and users are there?
6. Which currency and time zone does the CRM store?
7. Where are targets kept, if anywhere?

**Group 3: decide later (during design, not now)**
1. Where the system runs (our server or cloud).
2. How long raw weekly copies are kept.
3. Which Claude model each agent uses.
4. Whether the CRM calls the agents directly or through a small service.

*Confidence: 90 / 100. These are the gaps we know we have. Answers are needed before the technical design starts.*

## 12. What comes next

1. Decide the Group 1 questions together.
2. Answer the Group 2 questions from our CRM schema and add the field-mapping appendix.
3. Write the Functional Specification: step by step, inputs and outputs for each agent.
4. Write the architecture (high-level and low-level design).
5. Build, one agent at a time, in Claude Code.

*Confidence: 90 / 100. This is the order we agreed. No one outside the two of us is involved yet.*
