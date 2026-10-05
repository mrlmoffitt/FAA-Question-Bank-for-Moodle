# Assessment Quiz — Specification

Status: draft for implementation · Last revised 2026-10-05 (rev. 3)

This document specifies the **assessment** version of the quiz web app: scored assignments and in-class tests drawn from the FAA question bank. It is a separate Apps Script project from the **practice quiz** and shares no code or deployment with it. Each project has its own copy of the shared logic (question formatting, figure links, grading), so a change to one never affects the other.

---

## 1. Platform and access

- Standalone Google Apps Script project, deployed as a web app.
  - **Execute as:** the teacher (owner).
  - **Who has access:** anyone in the school's Google Workspace domain.
- Students must be signed into a school account. The script identifies each student by their account email; there is no name or ID field to fill in.
- Students have no access to the question Sheet, the results spreadsheets, or the script source. They receive only the HTML the script returns.
- Set the project's time zone (Project Settings) to the school's time zone. All dates and times in configs and results use it.

## 2. Data sources

### 2.1 Question bank (Google Sheet)

Read fresh from the Sheet on every request, so corrections take effect immediately.

| Column | Content |
|---|---|
| A | Question # (permanent, unique ID) |
| B | Question text |
| C, D, E | Option A, B, C text |
| F | Correct answer (A, B, or C) |
| G | Category, as a `/`-separated path (e.g. `Preflight preparation/Weather`) |
| H, I, J | Reserved for per-option feedback on incorrect answers (§9). Ignored for now. |
| K+ | Ignored |

Rows missing column A, B, F, or G are ignored.

**Editing questions that are already in use:** fix wording freely; it shows up on the next load or review. To fix a wrong answer key, change column F and then regrade (§7). Do not swap the text between columns C, D, and E for a question that has been used. Saved answers and choice orders refer to those columns by letter, so a swap would silently change what students' recorded answers mean.

**Category tree rule:** questions belong only to *leaf* categories. A category either has subcategories or has questions, never both. The admin page's config check (§7) reports any violations it finds.

**Allowed markup in question and option text** (all Moodle-compatible):

- `<br>` (any case, `<br/>`, `<br />`) starts a new, indented line.
- `<sup>…</sup>` and `<sub>…</sub>` render as superscript and subscript.
- `&nbsp;` is a non-breaking space. Runs of it are used for table column offsets.
- Everything else is shown as literal text.

### 2.2 Figure references (`figureReferences.txt` in the repo)

One `label:path` entry per line:

```
Figure 9:Figures/Figure09.jpg
the Chart Supplement Legend:Figures/ChartSupplementLegendBW.pdf
```

- `Figure n` labels match "Figure n" or "Figure 0n" anywhere in the question text.
- Other labels match their exact wording, ignoring capitalization and extra spaces. The longest matching phrase wins.
- Paths are relative to the repo's raw base URL. A path starting with `http(s)://` is used as-is.
- Matches become links that open in a new tab. The file is cached for 10 minutes.

### 2.3 Assignment configs (repo folder `Assignments/`)

One text file per assignment, e.g. `Assignments/unit3-test.txt`. See §3. Configs contain nothing secret; keys live only in the script (§6).

## 3. Assignment config

### 3.1 Invocation

```
<web app URL>?a=unit3-test
```

The script loads `Assignments/<a>.txt` from a fixed base URL set in the code. The `a` value may contain only letters, digits, `-` and `_`. The script never accepts a full URL or path from the link, so students cannot point it at a config of their own.

Configs are cached for 5 minutes, so an edit can take that long to reach students.

### 3.2 Format

One setting per line, `name: value`. Blank lines and lines starting with `//` are ignored. Names are case-insensitive.

```
title:         Unit 3 – Weather Systems Quiz
R3:            Preflight preparation/Weather
R2:            Preflight preparation/Human factors
R:             Post-flight procedures, emergency and night operations
#112
#245
shuffle:       yes
attempts:      3
score:         best
feedback:      missed
feedback-when: released
time:          30
open:          2026-10-12 08:00
close:         2026-10-19 23:59
key:           required
```

### 3.3 Question selection

| Line | Meaning |
|---|---|
| `Rn: category` | Draw *n* random questions from that category. `R:` alone means 1. |
| `#nnn` | Include question *nnn* (column A). |

- Lines may appear in any order and any number of times. Each attempt is assembled from all of them.
- **Category matching:** a category name matches itself and every category beneath it. `R3: Preflight preparation` draws from all of `Preflight preparation/…`. Matching ignores capitalization and extra spaces.
- **No repeats within an attempt:** a question added with `#nnn` is excluded from random draws, and overlapping `R` lines never draw the same question twice.
- If a category has fewer unused questions than requested, the assignment is considered misconfigured (§3.5).

### 3.4 Settings

| Setting | Values | Default | Meaning |
|---|---|---|---|
| `title` | text | the `a` value | Shown to students and used to name the results file. |
| `shuffle` | `yes` / `no` | `yes` | Randomize question order. Answer choices are always shuffled. |
| `attempts` | number / `unlimited` | `unlimited` | Maximum attempts per student. |
| `score` | `first` / `best` / `last` / `average` | `best` | Which attempt(s) count in the Summary tab (§5.2). |
| `feedback` | `none` / `missed` / `full` | `missed` | What a student sees when reviewing an attempt (§4.6). |
| `feedback-when` | `completed` / `close` / `released` | `released` | When feedback becomes visible (§4.6). |
| `time` | minutes | none | Time limit per attempt (§4.4). |
| `open` | `YYYY-MM-DD HH:MM` | none (open now) | Attempts cannot start before this time. |
| `close` | `YYYY-MM-DD HH:MM` | none (never closes) | Attempts cannot start after this time, and in-progress attempts end at it. |
| `key` | `required` | none | Students must enter the access key to start an attempt (§6). |

If `open` and `close` are both omitted, the assignment is simply open.

`feedback-when: close` with no `close` date behaves like `released`.

### 3.5 Misconfiguration

If the config is missing or invalid, students see "This assignment isn't available right now. Please tell your teacher." Examples include an unknown setting, an unknown category or `#ID`, too few questions for an `R` line, `key: required` with no key set, and `feedback: full` with `feedback-when: completed` when more than one attempt is allowed (students would see the answers before retaking). The specific problem is shown only on the admin page.

## 4. Student experience

### 4.1 Landing page

Opening the assignment link shows:

- the assignment title and "Signed in as *student@school*";
- open/close times, the time limit, and the attempt allowance, where set;
- the student's previous attempts, each with its date, score (once feedback is visible), and a **Review** button when feedback is available;
- a **Start attempt** button (or **Resume attempt**, §4.3), with a key field if required. The button is replaced by an explanation when no attempt is possible: not yet open, closed, or no attempts left.

### 4.2 Taking an attempt

- The landing controls are hidden while an attempt is in progress.
- With an attempt limit, the header reads "You are on attempt #5/9". Without one, it reads "Attempt #5".
- With a time limit, a countdown is shown.
- Each selection is saved to the server as it is made (autosave).
- **Submit** first asks for confirmation: "This completes your attempt and will grade your answers. Continue?" On confirming, the attempt is graded. Unanswered questions count as wrong, with no separate warning. (An automatic submission at the time limit skips the confirmation.) The student then returns to the landing page, which shows feedback if `feedback-when` allows it.

### 4.3 Reloading and resuming

- An attempt is tied to the student's email, not the browser tab. Reloading, closing the tab, or switching computers and returning resumes the same attempt. It keeps the same questions, the same answer-choice order, and the autosaved answers.
- Resuming never draws new questions and never counts as a new attempt.
- If the assignment requires a key, resuming requires the **current** key (§6).
- Question and option text are re-read from the Sheet on every load. Corrections made after the attempt started therefore appear on resume and in later reviews.
- A student can have at most one unsubmitted attempt per assignment.

### 4.4 Time limits and closing

- An attempt's time limit counts only time the attempt is **active**. Time while it is suspended (§4.5) doesn't count. Its deadline is the earlier of *now + remaining time* and the `close` time, recalculated whenever the attempt starts or is unsuspended.
- The server enforces the deadline. The page submits automatically when the countdown reaches zero, and the server accepts a submission up to 60 seconds past the deadline to allow for network delay.
- If an active attempt passes its deadline without being submitted (tab closed, computer off), its autosaved answers are graded automatically. This happens when the student next opens the assignment, when the admin page is opened, or within 15 minutes via a time-driven trigger.
- A teacher can also end in-progress attempts from the admin page (§7), for example at the end of class.
- A suspended attempt never runs out of time, but it is still ended by the `close` time.
- Results record how each attempt ended: `submitted`, `time limit`, `closed`, or `teacher`.
- An open attempt page checks in with the server every 30 seconds, as well as on every autosave. If the attempt has ended (time limit, close, or teacher), the page shows "This attempt has ended" and returns to the landing page. If it has been suspended, the page shows the paused screen (§4.5). Either way, every answer selected so far has already been saved.

### 4.5 Suspending and continuing later

For students whose extended time is split across sessions, for example half the test in class and the rest during a free period in a testing room.

- A teacher suspends individual attempts from the admin page (§7). Students cannot suspend their own attempts.
- Within 30 seconds, the student's open page replaces the questions with: "Your attempt is paused. Your answers are saved. Enter the current key to continue." The countdown stops.
- To continue, on any device, the student opens the assignment link and enters the **current** key. The attempt resumes with the same questions, choice order, and saved answers, and with the time remaining when it was suspended.
- Typical end-of-class sequence:
  1. Suspend the students who will finish later.
  2. **End all** remaining in-progress attempts.
  3. Change the key.
  4. Give the new key to whoever supervises the later session.
- An attempt can be suspended and continued any number of times. Suspension sends no email; the receipt is sent when the attempt finally ends.

### 4.6 Feedback and review

`feedback` controls *what* a review shows:

| Mode | Review shows |
|---|---|
| `none` | Score only. |
| `missed` | Score; every question with the student's answer, missed questions marked. |
| `full` | As `missed`, plus the correct answer for each missed question. |

`feedback-when` controls *when* it becomes visible:

| Value | Visible |
|---|---|
| `completed` | As soon as the attempt is submitted. |
| `close` | After the assignment's `close` time. |
| `released` | After the teacher releases it on the admin page (§7). |

- Until feedback is visible, the landing page lists the attempt as "Submitted — feedback not yet available", with no score.
- Students review any time afterward by returning to the same assignment link and choosing **Review** on an attempt. A review shows the attempt's questions in the order they were taken, with the same choice order.
- Reviews reflect the most recent grading, including any regrade (§7). Question wording comes from the current Sheet.
- Questions excluded from grading (§7) are shown in reviews as "Not graded".

### 4.7 Email receipts

Whenever an attempt ends, however it ends, the student receives a receipt email. This includes a submission, a time limit, the assignment closing, and a teacher ending it. Students should be taught to treat the receipt as their proof of submission.

- **From:** the account the script runs as, with the assignment title as the sender name. Replies go to that account.
- **Subject:** `Receipt: <title> – attempt #n`
- **Body:**
  - the assignment title;
  - the attempt number;
  - when the attempt started and ended;
  - how it ended;
  - how many questions were answered out of the total;
  - a short receipt code (the first characters of the attempt token);
  - the assignment link for reviewing.
- **Score:** included only if feedback is already visible under `feedback-when`. Otherwise the email says the score will be available when feedback is released.
- Attempts that are finalized late are emailed at the moment they are finalized. This covers a tab closed before the time limit, which is graded on next access or by the 15-minute trigger.
- Regrades and exclusions do not send email.
- A failed email never blocks grading. The failure is recorded in the Attempts tab, and the admin page shows the count.

## 5. Results spreadsheet

One spreadsheet per assignment, named after its title, created in a configured Drive folder on the first submission. The script stores its ID and reuses it; a script lock prevents duplicates when the first submissions arrive simultaneously. Archive the folder at the end of the semester.

### 5.1 `Results` tab — one row per finished attempt

| # | Column | Example |
|---|---|---|
| 1 | Submit timestamp | 2026-10-14 10:42:13 |
| 2 | Email | student@school.org |
| 3 | Score | 7 |
| 4 | Out of | 8 |
| 5 | Percent | 87.5 |
| 6 | Attempt # | 2 |
| 7 | Started | 2026-10-14 10:21:50 |
| 8 | Active time (min) | 20.4 (excludes time suspended) |
| 9 | Ended by | submitted / time limit / closed / teacher |
| 10 | Question IDs (as served) | 112, 245, 703, … |
| 11 | Answers (original letters) | 112:B, 245:—, 703:C, … |
| 12 | Missed IDs | 245, 811 |
| 13 | Regraded at | 2026-10-16 15:02:40 (blank if never regraded) |
| 14 | Score before regrade | 6 (blank if never regraded) |
| 15 | Excluded IDs | 811 (questions in this attempt not counted; blank if none) |

Score and Out of count only graded questions. An attempt of 8 questions with one excluded is scored out of 7.

Answers are recorded as column-F letters, so they compare directly with the Sheet regardless of display order. `—` means unanswered.

### 5.2 `Summary` tab — one row per student

Rewritten after every finished attempt. Columns: Email, Attempts used, Score, Out of, Percent, Last submitted. The Score columns follow the config's `score` rule: `first`, `best`, `last`, or `average` (average of percents).

### 5.3 `Attempts` tab — internal state

Holds each attempt's token, email, start time, deadline, served questions and choice order, autosaved answers, and status. The script uses it to resume, enforce deadlines, and build reviews. It is not intended for hand editing.

## 6. Access keys

- `key: required` in the config means students must enter the key to **start** an attempt. They must enter the **current** key again to **resume** one after a reload, on another device, or after a suspension (§4.5).
- The key itself is set on that assignment's admin page (`?a=unit3-test&admin`) and stored in the script's private Script Properties, per assignment. It never appears in the repo, the page source, or the results.
- Matching ignores capitalization and leading or trailing spaces.
- The teacher shares the key in class and can change it at any time, for example between class periods or at the end of an in-class test. After a change, nobody can start or resume with the old key. A page that is already open keeps working until its attempt is ended or suspended, so to stop students who walk out with a page still open, end or suspend their attempts from the admin page (§7).
- A wrong key gets "That key isn't correct". Nothing is served until the key is accepted.

## 7. Admin page

```
<web app URL>?a=unit3-test&admin
```

**Who can open it:** anyone with edit access to the question Sheet. The script, running as its owner, checks the signed-in account against the Sheet's editor list (cached for 5 minutes). Granting or removing someone's edit access to the Sheet therefore also grants or removes admin access, with no separate list to maintain. Anyone else who adds `&admin` simply gets the normal student view. Because the project is standalone, Sheet editors do not see or edit the script itself.

All admin actions apply to the assignment named in the link. The page provides:

- **Config check:** the parsed config, the questions available for each `R` line, and any errors (§3.5) or category-tree violations (§2.1).
- **Key:** set, change, or clear the access key.
- **Feedback release:** release or withdraw feedback when `feedback-when: released`.
- **Results:** a link to the results spreadsheet.
- **In-progress attempts:** each unfinished attempt, with its status (active or suspended), start time, active time used, and time remaining. Actions on selected attempts:
  - **Suspend selected** pauses those attempts (§4.5).
  - **End selected** or **End all** grades the attempts immediately from their autosaved answers (Ended by: `teacher`) and sends receipts. End all includes suspended attempts unless they are deselected.

  Typical use at the end of an in-class test: suspend the extended-time students, end everyone else, then change the key (§4.5).
- **Finalize now:** grade any expired attempts immediately (§4.4).
- **Regrade:** re-score every finished attempt for this assignment against the current column F. A preview first lists the attempts whose scores would change. On confirmation, the script:
  - updates Score, Percent, and Missed IDs in the Results tab;
  - fills in Regraded at, and records Score before regrade the first time an attempt's score changes;
  - rewrites the Summary tab;
  - makes reviews show the corrected grading.

  Questions since deleted from the Sheet keep their original grading. Regrading never changes which questions an attempt contained or the answers recorded.
- **Exclude from grading:** for a question that can't be fixed by correcting the answer key. Enter its question # and see a preview of the attempts affected; on confirmation:
  - the question no longer counts for anyone in this assignment. It is removed from both Score and Out of in every finished attempt, listed in Excluded IDs, and the Summary tab is rewritten;
  - attempts still in progress show it but won't count it;
  - new attempts don't serve it. A random draw takes another question from the same `R` line instead; a `#nnn` question is simply dropped;
  - reviews show it as "Not graded".

  An exclusion can be undone with **Include again**, which regrades accordingly. Exclusions apply only to this assignment. To retire a question from every assignment, fix it or delete it in the Sheet, which also excludes it from future draws.
- **Email problems:** the number of receipts that failed to send, if any.

## 8. Security notes

- Answer keys never leave the server. Pages receive question text and shuffled choices tagged with original letters, never column F.
- Submissions are accepted only for a valid, unsubmitted attempt token belonging to the signed-in student. Only the question IDs recorded for that attempt are graded.
- Time limits, open/close times, attempt limits, and keys are all enforced on the server, not just in the page.
- An attempt can continue anywhere a page is still open until it is ended or suspended. Changing the key stops new starts and resumes, but only ending or suspending the attempt stops an already-open page, within about 30 seconds.
- Not covered: the script cannot stop students from using notes, other tabs, or each other during an attempt. In-class assessments still depend on supervision.

## 9. Out of scope (for now)

- Resetting an individual student's attempt. Technical problems are handled ad hoc from the Results tab.
- Feedback for incorrect answers on individual questions. This will use columns H, I, and J of the question Sheet.
- Different time limits for individual students. Extended time is handled with a generous `time` setting plus suspending and ending attempts individually (§4.5, §7).
- Partial credit, question types other than three-option multiple choice, and syncing with a gradebook or LMS.
