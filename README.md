# brain-tumor-mri-assistant
## Week 9: LLMs and Prompt Engineering

- Provider and model: Groq, openai/gpt-oss-20b; key stored in Kaggle Secrets (never in a cell)
- Product: help-desk assistant for our brain tumor MRI API (Week 8). It answers questions from doctors, hospital IT staff and students using only verified API facts, and replies in validated JSON.
- Temperature test: 1 distinct answer at T = 0 vs 5 distinct answers at T = 1
- Tokens: English 4.2 chars/token, Urdu 2.9, Roman Urdu 3.6 (Urdu needs 1.88x more tokens than English)
- Prompt v1 vs v2 on 26 test questions: valid JSON 0.00 -> 1.00, facts 0.30 -> 0.95, scope n/a -> 0.88; with 20 fact questions the 95% interval is +/- 0.10
- Attacks: 4 of 4 resisted (plus 2 of our own: pretending to be the project owner, and a request to repeat a patient's phone number and CNIC); no canary leak. Defenses beyond the prompt: output filter, masking personal data before it reaches the model, real authentication for staff.
- Estimated cost at 1,000 users x 20 queries/day: about USD 95 per month (500 users x 5 queries/day: about USD 11.83 per month)
- Prompts are versioned in prompts/ (system_v1.md, system_v2.md) with results in evals/ (questions.csv, results_v1_v2.csv, summary.csv)

### Evaluation notes
- First run: v2 facts scored 0.85 +/- 0.16. The scorer had a bug (the model wrote "90 ms" and "5 MB" with a non-breaking space, so string matching failed). After normalizing those characters the score is 0.95 +/- 0.10. Questions and prompts were not changed.
- Known weaknesses: the scope rule in system_v2 is ambiguous (3 "related but unanswerable" questions got in_scope = false), and the "/health" question uses a too-literal required phrase.
- Next step: grow the evaluation set from 26 to 30+ questions with real user questions.

### Standup notes (Week 9)
- Done today: set up Groq with retries and logging, measured temperature and token cost, wrote and evaluated v1 and v2 prompts, ran 4 attacks plus 2 of our own.
- Next: grow the evaluation set to 30+ questions, fix the scope rule in a new system_v3.md, and collect 20 to 50 pages of real documents for Week 10 (retrieval).
- Blockers: none.
- Owner of prompt changes: M. Abuzar Shoukat. Owner of growing the evaluation set to 30 questions: M. Abuzar Shoukat.

### Check Your Understanding: M. Abuzar Shoukat
1. For logits [2, 0] at T = 1: e^2 / (e^2 + e^0) = 7.389 / 8.389 = 0.881.
2. At T = 0 the model always picks the most likely token, so the same input gives the same output. When extracting fees and dates into JSON, randomness would mean a wrong or changing number, so we want determinism.
3. Queries = 500 x 10 x 30 = 150,000. Cost per query = (1,000 x 0.30 + 200 x 2.50) / 1,000,000 = USD 0.0008. Total = USD 120 per month.
4. If many clients fail at the same moment and all retry after exactly 1, 2, 4 seconds, they hit the server together again and fail again. A random part spreads the retries out.
5. v2 = 17/20 = 0.85, interval +/- 1.96 x sqrt(0.85 x 0.15 / 20) = +/- 0.16 (0.69 to 1.00). v1 = 15/20 = 0.75, interval +/- 0.19 (0.56 to 0.94). The difference is only 0.10 and the intervals overlap heavily, so with 20 questions we cannot say v2 is better. We need more questions.
6. Tags tell the model that the user text is data, not instructions, which makes injection harder (it helps but is not a full defense). A canary is a secret string in the system prompt; if it appears in an output, the prompt has leaked.
