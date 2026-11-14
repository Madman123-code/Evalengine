Fair Exam Evaluator

AI-assisted marking of handwritten answer sheets, with the teacher in control.

Upload scanned answer-sheet PDFs, and the system reads the handwriting, marks each descriptive answer against a rubric, applies "answer N of M" paper rules, flags marks it is unsure about, and gives you an Excel mark sheet and a marked PDF with the marks written on the student's own sheet.

The AI proposes, the code enforces, the teacher decides. Treat every mark as a suggestion until a teacher has reviewed it.

Contents
What it does
How a sheet is marked
Design decisions and why
Quick start
Choosing the AI provider
Using the web app
Outputs
Project structure
HTTP API
Data and sheet states
Testing
Demo files
Limitations
Security and privacy
Troubleshooting
Roadmap
Related work
1. What it does
Need	How the system meets it
Mark many handwritten sheets quickly	Upload several PDFs at once; a background worker marks them one after another.
Fair, consistent marks	Three-stage marking, optional three independent scoring passes with the median taken, and a fairness audit.
Real paper rules	Any number of questions, students answer N of M; extra answers count as best N or first N.
Teacher stays in charge	Doubtful marks are flagged; the teacher can edit any mark and totals are recomputed.
Results that can be used	Excel sheet (red = fail, green = good marks) and a PDF with marks stamped on the student's own sheet.
Cost and privacy options	Free Gemini tier, paid Claude, or fully local models through Ollama.
Trust	Marks are limited by code, student text cannot give orders to the AI, and the test suite runs without any AI at all.
2. How a sheet is marked
 PDF ──► Read sheet ──► for each answer:  Extract claims ──► Score vs rubric (x R) ──► Fairness audit
          name, ID,                                                │
          answers by                                  Validate in code (clamp, cap, total)
          question                                                 │
                                          Paper rules (N of M) ──► Total, %, PASS / FAIL / good
                                                                   │
                                          Flags ──► Teacher review ──► Excel + marked PDF
Read the sheet. The model reads the PDF, transcribes each answer exactly as written ([illegible] for unreadable words), matches it to the question number the student wrote, and reads the student's name and ID. It also estimates where each answer starts on the page, so marks can be stamped there.
Extract claims. No marks yet. The answer is broken into claims, each judged correct, partial, incorrect, ambiguous or unverifiable. Contradictions and unreadable parts are listed.
Score against the rubric. Every rubric criterion gets marks between 0 and its maximum. Valid points outside the rubric go into "alternative valid points". This runs R times (R = 1 for Fast, 3 for Careful in the UI; the API accepts 1 to 5). The run whose total is closest to the median is used.
Fairness audit. A separate check asks whether anything was marked down only for differing from the reference answer, whether marks lack evidence, whether style or length was rewarded, and whether a valid alternative was ignored.
Validation in code. See the rules below. This step does not use the AI.
Paper rules. Decide which answers count, then compute the total, percentage, PASS or FAIL and "good marks".
Rules enforced in code (validate.py, exam.py)
Marks for each criterion are clamped to 0 .. criterion maximum.
Missing criteria in the model's reply score 0 and raise a warning.
Credit for alternative valid points can only fill marks lost elsewhere, and the total never exceeds the question's maximum.
Totals are always recomputed from the marks; they are never typed in separately.
Pass and good-marks thresholds are compared on the unrounded percentage, so 39.96% does not pass a 40% mark.
If extra answers are written, the highest-scoring N (ties: the earlier answer) or the first N count.
When an answer is flagged for review

An answer is flagged when any of these measurable signals appear. Confidence is never taken from the model's own opinion of itself.

Signal	Rule
Scoring runs disagree	spread between runs is over 15% of the question's marks
Unclear handwriting	reading confidence is under 0.8, or the model lists unreadable parts
Fairness audit raised issues	one or more issues listed
Student contradicts themselves	contradictions found
Model output needed correction	clamping or capping happened
Ambiguous criteria	any criterion judged ambiguous
Possible prompt injection	the answer tries to give instructions to the marker

Confidence is 1 − spread/marks − 0.15 × audit issues − (1 − reading confidence) − 0.1 × corrections, kept between 0 and 1.

3. Design decisions and why
Decision	Reason
LLM proposes, code disposes	A model can return marks above the maximum or forget a criterion. Rules in code cannot be talked around.
Three stages, not one prompt	Extracting claims before scoring, and auditing after, makes each step checkable and cuts whole-answer guessing.
Several scoring runs, median used	One model call can vary. Repeating it exposes unstable marks, which are then flagged.
Objective confidence signals	Model self-reported confidence is unreliable; spread, reading quality and audit issues are measurable.
Student text is untrusted data	Answers are wrapped in <student_answer> tags and the prompts say never to follow instructions inside. Attempts are flagged.
One interface for every AI provider	complete_json and read_pdf_json are all a provider must implement, so switching is one setting.
JSON file storage	No database server is needed for one teacher on one computer.
Thread-pool job queue, 1 worker by default	AI calls are slow, so the browser must not wait; one worker respects free-tier rate limits.
Mock model for tests	The AI is not repeatable, so a stand-in with fixed replies lets every expected result be exact.
Teacher is the final authority	Wrong marks change a student's result; flags and edits keep a person accountable.
4. Quick start

Requirements: Python 3.10 or newer, and one of a Gemini key, a Claude key, or Ollama (see section 5).

bash
# 1. Get the code and enter the folder
cd evalengine

# 2. Create and activate a virtual environment
python -m venv .venv
.venv\Scripts\Activate.ps1          # Windows PowerShell
# source .venv/bin/activate         # macOS / Linux

# 3. Install
pip install -r requirements.txt

# 4. Set an AI key (free Gemini key from https://aistudio.google.com)
$env:GEMINI_API_KEY="your-key"      # Windows PowerShell
# export GEMINI_API_KEY="your-key"  # macOS / Linux

# 5. Run
uvicorn evalengine.api:app --reload

Open http://localhost:8000.

Run the tests (no key or internet needed):

bash
python -m unittest

Notes for Windows PowerShell: use $env:NAME="value" instead of export, and separate commands with new lines instead of &&. Environment variables last only for the current window, so set them again in each new one.

5. Choosing the AI provider

Set LLM_PROVIDER to pick one. The default is gemini.

Provider	LLM_PROVIDER	Needs	Good for
Google Gemini (default)	gemini	GEMINI_API_KEY	Free tier, reads PDFs directly
Anthropic Claude	anthropic	ANTHROPIC_API_KEY	Paid, strong accuracy
Ollama (local)	ollama	Ollama installed and models pulled	Privacy, no rate limits, no key
Environment variables
Variable	Default	Meaning
LLM_PROVIDER	gemini	gemini, anthropic or ollama
GEMINI_API_KEY	none	Gemini key
GEMINI_MODEL	gemini-flash-latest	Main Gemini model. Fallbacks: gemini-flash-lite-latest, gemini-2.5-flash
ANTHROPIC_API_KEY	none	Claude key
OLLAMA_HOST	http://localhost:11434	Ollama address
OLLAMA_MODEL	qwen2.5:7b	Model that marks answers
OLLAMA_VISION_MODEL	qwen2.5vl:7b	Model that reads handwriting
OLLAMA_NUM_CTX	8192	Context size
OLLAMA_MAX_PAGES	8	Pages read per sheet
OLLAMA_TIMEOUT	900	Seconds to wait per request
EVAL_WORKERS	1	Sheets marked at the same time. Keep low on free tiers
EVAL_DATA	data	Folder for stored papers and PDFs
Gemini reliability

Busy or rate-limited replies (503, 429) are retried up to 8 times with growing waits, switching to a fallback model partway. If all attempts fail, the sheet ends as Failed with the reason shown, and can be retried.

Using Ollama
bash
ollama pull qwen2.5:7b
ollama pull qwen2.5vl:7b
pip install pymupdf                  # renders PDF pages to images for the vision model
$env:LLM_PROVIDER="ollama"
python check_ollama.py               # confirms the server and models are reachable
uvicorn evalengine.api:app --reload

Sheets are read one page at a time by the vision model and marked by the text model. A GPU is advised; without one, local models are slow. Connection refused means Ollama is not running: start it, or run ollama serve.

Adding another provider

Implement complete_json(stage, system, user) -> dict and read_pdf_json(stage, system, prompt, pdf) -> dict (see evalengine/llm.py) and add it to make_llm().

6. Using the web app

The page at / has three steps.

Set up paper. Enter the exam name, add questions with marks (optionally a model answer and rubric points), choose how many questions students must answer, what to do if they answer extra (count the highest marks or count the first answers), and the pass and good-marks percentages (defaults 40 and 75). Rubric points for a question must add up to its marks; if none are given, one "overall correctness" point is used.
Add answer sheets. Drop one or more PDFs. You may type each student's name and ID, or leave them blank so the system reads them from the sheet (anything you type is kept). Choose Fast (one marking pass) or Careful (three passes, checks agreement).
Results. A table shows each student, marks per question, total, percentage and result, with failed rows red and good-marks rows green. Open a row to see the marks and reasons, edit a mark, view the sheet, open the marked PDF, retry or delete. Download the Excel file or all marked PDFs as a zip.

Handy details:

Answers marked but not counted (because the student answered more than N) are struck through.
A flag icon marks answers the system wants a teacher to check. Editing a mark clears that flag for it.
A sheet marked before the stamping feature existed uses default stamp positions; use Retry for better placement.
An older single-answer review page is at /review.
7. Outputs

Excel mark sheet (Marks and Summary sheets)

Columns: Student ID, Name, one per question, Total, Out of, Percent, Result, Remarks.
Red rows for FAIL, green rows for good marks, yellow remarks where a teacher should check, grey strike-through for answers not counted, - for unanswered questions, NOT MARKED for sheets that failed.
The Summary sheet shows the paper rules, counts of passed, failed and good sheets, average percent, and a colour key.

Marked PDF

The student's own sheet with each answer's marks written beside it, a total stamp, and a one-page summary at the end. Extra answers carry a "not counted" label.
Available per sheet or as a zip for the whole exam.
8. Project structure
evalengine/
├─ README.md
├─ requirements.txt
├─ check_ollama.py          checks the Ollama server and models
├─ smoke_test.py            small live test against the configured provider
├─ run_stqa.py              runs every test with statement coverage and timing
├─ static/
│  ├─ index.html            main three-step web app (vanilla JS, single page)
│  └─ review.html           older single-answer review page
├─ tests/
│  ├─ data/demo_sheet.pdf   one-page demo sheet used by the PDF tests
│  ├─ test_pipeline.py      pipeline tests (mock model)
│  ├─ test_exam.py          paper rules, export and sheet tests
│  └─ test_stqa.py          99 logged checks across 8 test techniques
└─ evalengine/
   ├─ pipeline.py           extract → score (×R) → audit, flags, confidence
   ├─ prompts.py            stage prompts and the prompt-injection guard
   ├─ validate.py           clamps marks, caps alternative credit, recomputes totals
   ├─ exam.py               paper set-up, N of M, totals, pass / good
   ├─ grading.py            reads a sheet PDF and marks every answer
   ├─ export.py             Excel workbook
   ├─ annotate.py           stamps marks on the student's PDF
   ├─ llm.py                Gemini, Anthropic, Ollama and mock adapters
   ├─ ocr.py                single-image reading adapters
   ├─ api.py                app, single-answer routes, page routes
   └─ exam_api.py           exam, sheet, export routes and the job queue

About 970 lines of Python in eleven modules. Stack: Python 3.10+, FastAPI and Uvicorn, openpyxl, pypdf, reportlab, PyMuPDF (Ollama only), and a plain HTML and JavaScript front end.

9. HTTP API

Interactive documentation is at http://localhost:8000/docs.

Method	Path	Purpose
GET	/	Web app
GET	/exams	List papers
POST	/exams	Create or update a paper (400 with a clear message if invalid)
GET	/exams/{eid}	Read a paper
POST	/exams/{eid}/sheets	Upload PDFs (files, optional meta JSON with student_id and student_name per file, runs 1 to 5). Non-PDF files are rejected
GET	/exams/{eid}/results	Paper and summary of every sheet
GET	/exams/{eid}/sheets/{sid}	One sheet with criteria, feedback and reasons
GET	/exams/{eid}/sheets/{sid}/pdf	Original PDF
POST	/exams/{eid}/sheets/{sid}/marks	Edit one mark (question_id, marks between 0 and the question maximum)
POST	/exams/{eid}/sheets/{sid}/identity	Edit student ID and name
POST	/exams/{eid}/sheets/{sid}/retry	Queue a failed sheet again
DELETE	/exams/{eid}/sheets/{sid}	Delete a sheet and its PDF
GET	/exams/{eid}/export.xlsx	Excel mark sheet
GET	/exams/{eid}/sheets/{sid}/marked.pdf	Marked PDF (409 until marking finishes)
GET	/exams/{eid}/marked.zip	All marked PDFs
POST	/evaluate	Mark one text answer (older single-answer API)
POST	/evaluate-image	Mark one answer image (older API)
GET / POST	/reviews, /reviews/{rid}	Older review queue: ACCEPT, MODIFY_MARKS, OVERRIDE, ADD_COMMENT, SECOND_REVIEW
10. Data and sheet states

Everything is stored in ./data (or EVAL_DATA): exams.json holds papers and results, and pdfs/ holds the uploaded sheets. Delete the folder to start fresh. The older single-answer API keeps reviews.json in the working folder.

 Queued ──► Processing ──► Done
    ▲            │            │ (teacher edits a mark: stays Done, totals recalculated)
    │            ▼
    └──────── Failed   (press Retry)

If the server restarts while sheets are queued or processing, they become Failed with "Interrupted by restart. Press Retry."

11. Testing

The AI model is replaced by a mock, so all tests run offline in about a second.

bash
python -m unittest        # 33 tests
python run_stqa.py        # same tests with statement coverage and timing; writes stqa_results.json

Current state: 33 tests, 99 logged checks, all passing.

Technique	Checks	What it covers
White-box	17	Every branch of validation, the seven review-flag rules, paper-rule branches, stamp placement
Black-box	15	Pass and good thresholds (39, 40, 41, 74, 75 and values just below), N of M totals
Valid and invalid	23	Bad paper forms, bad model replies, question-label matching
Unit	10	enforce, pick_counted, finalize, summary PDF, workbook structure
Integration	10	Reader → pipeline → rules → Excel and PDF on the real demo sheet; Gemini retry with an injected 503
Functional	10	"Answer 4 of 8" with best and first policies, edits, injection flag, Excel colours
Performance	7	Own-code time per answer and sheet, 500-sheet export, AI calls per sheet
Regression	7	Guards for defects found earlier

Statement coverage: 100% for exam, validate, pipeline, grading, export and prompts; 98.6% for annotate; about 40% for llm (parser, mock and Gemini retry only). The HTTP routes (api.py, exam_api.py) and ocr.py are not covered by automated tests; they need FastAPI and a live model.

Defects found and fixed during testing: a rounding error that let 39.96% pass at a 40% mark, a crash in the marked PDF when the model left out its feedback text, and a required count of 0 being silently accepted. Earlier defects (a missing reportlab in requirements.txt, a silent "Save paper" failure, an unreadable download link, Gemini overload errors, a stray folder in the release zip) each have a regression check.

AI calls per sheet: 1 + n × (R + 2) for n answers and R scoring passes. A 5-answer sheet needs 16 calls at R = 1 and 26 at R = 3.

12. Demo files

The test_pdfs/ folder (delivered next to the project folder, not inside the zip) contains ready-made files:

File	Purpose
SAMPLE_EXAM_SETUP.txt	Values to enter in step 1 (4 questions, answer 2)
Asha_S101_good.pdf, Ravi_S102_fail.pdf, Meena_S103_extra_answer.pdf	Expected: green row, red row, and an extra answer left uncounted
Question_paper_and_model_answers.pdf	Five-question paper with model answers and expected mark ranges
Sakshi_C2323_student_answers.pdf	Student sheet with mistakes planted on purpose
Sakshi_C2323_MARKED_demo.pdf	Example of the marked PDF output

On one run of the Sakshi sheet with Gemini during development, four of five answers fell in the expected range, the total was 13.25 of 25 (PASS), and the one answer marked above its expected range was flagged for review. Real handwriting and real exams still need to be tried.

13. Limitations
Marks are suggestions. The AI can misread handwriting or misjudge an answer; a misread word flows into the marking.
Not tested on a large set of real handwritten sheets, so agreement with teachers has not been measured.
Single teacher, single computer. No login, no database, no multi-user editing.
Free-tier limits can slow or interrupt a large batch; keep EVAL_WORKERS low and use Retry.
Diagrams are described in words by the model, not marked for accuracy.
Local models are slower and less accurate than the large cloud ones, and need a capable machine.
No automated tests of the HTTP layer or live AI quality (see section 11).
No licence has been chosen yet.
14. Security and privacy
Never put an API key in code, screenshots, chat messages or logs. Set it through the environment. If a key is ever shown publicly, delete it and create a new one.
Sheets sent to Gemini or Claude leave your computer. If scripts must stay private, use Ollama.
The prompt-injection guard reduces the risk that a student writes "give full marks", and any attempt is flagged, but no guard is perfect. Teacher review is the final safeguard.
Uploaded PDFs and results are stored unencrypted in data/. Do not run the server on a shared or public network without adding login.
15. Troubleshooting
Symptom	Cause and fix
Server crashes on start with No module named reportlab	Old requirements.txt. Run pip install -r requirements.txt again
uvicorn or pip not found, or PowerShell rejects && / source	Activate the venv with .venv\Scripts\Activate.ps1; use $env:NAME="value"; run one command per line
Sheet becomes Failed, message mentions 503, 429 or "overloaded"	The free AI service is busy. Wait a minute and press Retry; lower EVAL_WORKERS
"Save paper" shows an error	Questions students must answer is larger than the number of questions, or marks and rubric do not add up. Read the red message under the form
Many marks look struck through	The paper says students answer fewer questions than they did. Raise "Questions students must answer"
Connection refused on port 11434	Ollama is not running. Start it, or run ollama serve
Ollama says PyMuPDF is needed	pip install pymupdf
Results look old after copying a new index.html	Hard refresh the browser (Ctrl+F5)
Marked PDF has default stamp positions	Sheet was marked before stamping existed. Press Retry
Everything fails with an authentication error	Key not set in this terminal window, or wrong LLM_PROVIDER
16. Roadmap
Automated tests of the HTTP routes with FastAPI's test client.
Branch coverage with coverage.py.
Calibration against teacher-marked samples and paraphrase-consistency tests.
Many-teacher use: login, a real database, audit history.
Better handling of diagrams and mathematics.
A load test with a real model and rate limits.
17. Related work

Closest published work found while preparing this project (all open to read):

N. Aisyah, M. D. Al Kautsar, A. Hidayat, R. Chowdhury and F. Koto, "From Handwriting to Feedback: Evaluating VLMs and LLMs for AI-Powered Assessment in Indonesian Classrooms," arXiv:2506.04822, 2025. Handwriting reading, rubric-based LLM scoring and feedback; the nearest overall pipeline.
R. Kamble et al., "Multi-Modal Automated Evaluation of Handwritten Answer Sheets: A Framework Integrating SBERT and CLIP Encoding," SN Computer Science, vol. 6, art. 787, 2025.
A. Caraeni, A. Scarlatos and A. Lan, "Evaluating GPT-4 at Grading Handwritten Solutions in Math Exams," arXiv:2411.05231, 2024. A rubric and the correct answer improved agreement with human graders, but accuracy was still too low for unsupervised use.
O. Henkel, A. Boxer, L. Hills and B. Roberts, "Can Large Language Models Make the Grade?", arXiv:2405.02985, 2024. GPT-4 with few-shot prompting came close to expert teachers on short answers.

What this project adds: marks enforced in code, the N-of-M paper rules, a fairness audit with flags from measurable signals only, a prompt-injection guard, Excel and marked-PDF output, a local-model option, and a test suite that isolates the AI.
