Live Classroom Quiz Engine
A database-less live quiz server for 100 to 200 students. It runs on the invigilator's laptop and gives students a public link through a Cloudflare tunnel, so they can join from any network (school Wi-Fi, mobile data, hotspots). Students need nothing installed, just a browser.
Features
Public join link and terminal QR code, created automatically on startup
Students log in with Roll Number and Name and wait in a lobby until approved
One question at a time, with the answer key never sent to the browser
Random question set per student, with no two students getting the same combination (as long as the bank allows it)
Multiple choice and fill-in-the-blank, with typo-tolerant grading (for example "arrays" is accepted for "array")
Tab-switch detection: the current question is forfeited, scored 0, and the student moves on
One device per identity, and a second device gets HTTP 403
Live invigilator dashboard with progress, scores and cheat flags
Every event is logged to `live_quiz_backups.csv` through a single writer thread
Requirements
Python 3.10 to 3.13. Python 3.12 is recommended. Python 3.14 is very new and some dependencies fail to build there.
`cloudflared` for the public link. Without it the app still runs, but only on your local network.
Install cloudflared:
Windows: `winget install --id Cloudflare.cloudflared`
macOS: `brew install cloudflare/cloudflare/cloudflared`
Linux: download the package from https://github.com/cloudflare/cloudflared/releases
Setup
```
git clone <your-repo-url>
cd quiz_engine
pip install -r requirements.txt
```
On Windows with several Python versions installed, use the launcher instead:
```
py -3.12 -m pip install -r requirements.txt
```
Run
```
python app.py
```
(Windows with the launcher: `py -3.12 app.py`)
Run it as a single process. Do not use `--workers` greater than 1 and do not use `--reload`. All quiz state lives in this one process's memory, so extra workers would each hold their own separate copy.
On startup the terminal prints:
the public join URL and a QR code
the invigilator dashboard URL, which includes a secret key
the invigilator key on its own line
Keep the key off the projector. Share only the join URL or the QR code with students. Keep the terminal open for the whole quiz, because closing it stops the server and the tunnel.
Running a quiz
Invigilator:
Start the app and open the dashboard URL printed in the terminal.
Share the join link or QR code with students.
Students appear in the Waiting Lobby. Click Approve for one student, or Approve All.
Watch progress, scores and "TAB SWITCH DETECTED" flags update every 2 seconds.
Press Ctrl+C in the terminal when finished. Results are in `live_quiz_backups.csv`.
Students:
Open the link or scan the QR code.
Enter Roll Number and Name, then click Join Quiz.
Wait for approval, then answer one question at a time.
Stay on the quiz tab. Switching tabs or minimizing the window forfeits the current question.
Editing the questions
The question bank is the `QUESTION_BANK` list near the top of `app.py`. Each entry has an `id`, a `type` (`mcq` or `fitb`), the question text, the `answer`, and for MCQ an `options` dictionary. `QUESTIONS_PER_STUDENT` sets how many questions each student gets.
No cloudflared? LAN mode
If `cloudflared` is not found, the app prints install instructions and keeps running. Students on the same Wi-Fi can join at `http://<your-laptop-LAN-IP>:8000`. The invigilator key is still printed.
Testing
Start the server in one terminal, then in another:
```
python stress_test.py --key <INVIGILATOR_KEY> --students 150
```
This registers 150 students at once, checks that exactly one of ten simultaneous registrations for the same identity succeeds, runs everyone through the quiz, and checks the CSV log for corruption.
To compare against the naive version, run the same cases against each server (one at a time, since both use port 8000):
```
python eval_suite.py --key <INVIGILATOR_KEY>
python eval_suite.py --key "" --csv-path live_quiz_backups_baseline.csv
```
The second command assumes `baseline_app.py` is the server running.
Project files
`app.py`: the hardened server
`index.html`: student page
`dashboard.html`: invigilator dashboard
`stress_test.py`: 150-student concurrency test
`baseline_app.py`: the simple first-pass version, kept for before and after comparison
`eval_suite.py`: the same test cases run against either version
`EVALUATION.md`: results of the baseline vs hardened comparison
`CHANGELOG.md`: how the design evolved, including experiments that were removed
Design notes
A single global lock protects all shared student state, and the duplicate-login check is atomic.
Grading runs outside the lock, so one slow answer cannot stall other students.
Logging uses a queue and one background writer thread, so request handlers never touch the CSV file. Writes are eventually consistent, and the in-memory state is the source of truth.
Fill-in-the-blank grading is exact match, then `difflib` fuzzy match at a 0.85 threshold. If `OPENAI_API_KEY` is set, borderline cases get an optional second opinion from an LLM, and this needs `pip install openai`.
Known limitations
Cloudflare Quick Tunnels have no uptime guarantee and get a new URL on every restart. Restarting mid-quiz means re-sharing the link.
State is in memory only. If the server stops, in-progress sessions are lost, and only the CSV log survives.
A student whose device dies cannot rejoin with the same identity, because there is no invigilator override yet.
There is no quiz timer or deadline yet, and no unit or chapter grouping for the question bank.
The invigilator key is passed in the URL, which is fine for a classroom but not for a hostile network.
