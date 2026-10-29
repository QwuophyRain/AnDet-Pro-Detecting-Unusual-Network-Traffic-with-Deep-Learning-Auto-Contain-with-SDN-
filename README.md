AnDet Pro — the evaluation project
AnDet Pro (the full name used in the thesis is AnDePeD Pro) watches network traffic, spots attacks that look unlike normal traffic, and turns an alert into firewall rules for a software-defined network. This project runs the whole evaluation for the thesis "AnDet Pro: Finding Network Attacks Early and Blocking Them Automatically" and writes the tables, charts and draft sentences you paste into the thesis and the slides.

Start with RUN_GUIDE.md. It lists every step for a MacBook (M1), in the Warp terminal or PyCharm.

What the program does, in six steps
Step

In plain words

Where in the code

Read the traffic
Reads the NF-CSE-CIC-IDS2018 flow table (10 numbers per flow). Keeps every 59th flow, in time order. The first 70% in time is for training, the last 30% for testing.

andet/data.py

Smooth it
VMD (Variational Mode Decomposition) splits each of the 10 numbers into 5 smooth "modes". The modes are averaged over the 10 numbers, which gives 5 channels.

andet/vmd.py, andet/features.py

Learn "normal"
An LSTM autoencoder reads 60 steps at a time and tries to rebuild them. It is trained on normal traffic only, so it rebuilds normal traffic well and unusual traffic badly.

andet/model.py

Score the risk
Risk R is the rebuilding error of a window. The alert threshold is Θ = μ + 3σ, measured on normal windows that were not used for training. R ≤ Θ is NORMAL, Θ < R ≤ 2Θ is MODERATE, R > 2Θ is CRITICAL.

andet/model.py

Block
MODERATE blocks the source but keeps whitelisted ports open (priorities 1100 and 1200). CRITICAL blocks everything from that source (priority 65000). Rules are written for a Ryu controller (OpenFlow 1.3); in these runs they are created and replayed in simulation mode.

andet/policy.py

Compare
300 trials per condition: AnDet Pro against a static baseline (a fixed 3-sigma rule that a person must review). Measures: time to detect (MTTD), time to respond (MTTR), false positive rate (FPR), containment, plus statistical tests.

andet/trials.py, andet/baseline.py, andet/stats.py

The three run modes
Command

What it is for

Time

python run.py --mode fast

Checks that everything works. Uses a tenth of the data and 4 epochs. Its numbers are never used.

a few minutes

python run.py --mode standard

The real run for the thesis. Same sample, split, model and 300 trials per condition as Run 1, with the fixes listed in the update sheet.

the long one (mostly model training; the program prints its own estimate after the first epoch)

python run.py --mode full

Standard plus extra checks: five seeds, window length 30 and 90, block VMD, other attack densities.

several times longer than standard

What you get
Every run makes its own folder, results/<date_time>_<mode>/. Open these two files first:

File

What it is

RESULTS_FOR_THESIS.md

The tables (same columns as Tables 4.3–4.6 of Run 1), draft sentences for Chapter Four, and a block of numbers for the slides

DOCUMENT_UPDATE_SHEET.md

A checklist: every place in the thesis and the slides that shows a Run 1 number, the sentence as it stands now, and what to change it to

The rest of the folder: tables/ (every table as a CSV that opens in Excel), figures/ (8 PNG charts), records/ (the records of thesis Table 3.7: TRIAL_RESULT, POLICY_ACTION, WINDOW_and_RISK_SCORE, RUN), models/ (the trained model and its training history), config_used.json (every setting used), environment.json (computer and package versions), run.log (everything the program printed).

What is assumed, and what is not measured
These are written into the results files too, so the thesis can state them honestly.

One flow = one second. The data set has no clock inside a trial, so each flow advances time by data.step_seconds (1 s) plus the measured computing time. Detection times are therefore counted in flows × 1 s.
The person's review time is assumed: 38 s (SD 5 s), carried over from Run 1. The baseline's detection is executed on the traffic; only the human delay is an assumption.
Containment is an upper bound. The data has no IP addresses, so the rules are assumed to find the attacker perfectly. Harm to normal users is reported only as the share of normal traffic hit by a false alarm.
Simulation mode. No Ryu controller or Open vSwitch was used, so NFR-06 (rule installed within 15 ms), NFR-07, NFR-08 and NFR-10 stay Open.
VMD sees the whole test stream by default (vmd.scope = stream). A check in the report compares this with VMD that only sees past flows (look-ahead check).
Sustained attacks. The report also replays the untouched test stream and shows how the risk depends on how much of a window is attack. In a size test, windows made only of near-identical attack flows were scored lower than normal windows. The trials mix attack flows with normal flows, so they do not show this. Your own run shows what happens with a fully trained model.
False alarms. Run 1 had a high FPR (37%, SD 48.4). The report separates the causes it can test (calibration windows, test-stream segments, windows near and far from attacks, look-ahead). It words the likely cause as likely, not as a finding.
Folder map
AnDet_Pro/
  run.py                  the one command you use
  check_setup.py          checks Python, packages, the data file and the settings before a long run
  config/config.ini       every setting, with comments
  requirements.txt        packages to install
  data/                   put NF-CSE-CIC-IDS2018.parquet (or .csv) here
  andet/                  the code: data, vmd, features, model, policy, baseline, trials, stats,
                          metrics, analysis, figures, report, runner, config
  tests/                  158 automated tests (python -m pytest tests -q)
  results/                created when you run; one folder per run
Change a setting
Edit config/config.ini (every setting is explained there), or change one setting for one run:

python run.py --mode standard --set model.epochs=60 --set trials.reps=20
A typo in a setting name stops the program with a message before anything long starts. Whatever you change is recorded in config_used.json and shown in the Model table of the results, so the thesis stays truthful.

Tests
python -m pytest tests -q
The tests use a small made-up flow table, not the real data. They check the VMD, data split, windows, policy rules and priorities, baseline, statistics, trial construction, the model, the report files, and a whole tiny run including --resume. About a minute and a half.

Limits of this project
It evaluates the method on one recorded data set (NF-CSE-CIC-IDS2018) with injected attack trials. It is not a production system: live capture and a live Ryu controller are not part of this project.
