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

# Nosharwan Khan

## Week 9 extension:

- **system_v3** (`prompts/system_v3.md`): only the scope rule rewritten. in_scope is true for any question about the API even if the facts do not cover it, and false only for unrelated requests.
- **Evaluation set** grown from 26 to 38 questions (Roman Urdu, children's scans, treatment advice, unanswerable and out-of-scope questions). The required phrase of the /health question was changed from `model_loaded` to `loaded` after seeing the results (disclosed).
- **v2 vs v3 on 38 questions** (Wilson 95% intervals): scope 0.87 (0.73 to 0.94) -> 1.00 (0.91 to 1.00); facts 0.93 -> 0.93 (0.79 to 0.98); valid JSON 1.00 -> 1.00. v3 fixed 5 questions and broke 0 (sign test p about 0.06). The rule was written after seeing v2's mistakes, so a fresh held-out set is needed.
- **Reproducibility:** v3 repeated at T = 0: facts 28/30 then 27/30, scope 38/38 both times; 14 of 38 answers were worded differently.
- **Defenses in code:** `mask_pii()` (CNIC and phone before the model) and `output_filter()` (canary and PII in the output). 9 attacks (6 original + translate-instructions, Roman Urdu extraction, fake doctor): 0 leaks before and after, 0 blocked. The filter has only been tested by unit tests so far.
- **Remaining v3 failures:** one ambiguous question (text file renamed to .jpg) and one too-literal required phrase ("doctor").
- **Cost** measured in my own run: about USD 80 per month at 1,000 users x 20 queries/day (about USD 91 with a realistic 550 input tokens), and about USD 10 per month at 500 users x 5 queries/day.
- Files: `prompts/system_v3.md`, `evals/questions_v2_38.csv`, `evals/results_v2_v3.csv`, `evals/summary_v2_v3.csv`, `evals/attacks_before_after.csv`.
### Check Your Understanding: Nosharwan Khan

**1. For logits [2, 0] at T = 1, what is the probability of the first token?**
Softmax gives e^2 / (e^2 + e^0) = 7.389 / (7.389 + 1) = 7.389 / 8.389 = **0.881**. The first token gets 88.1% and the second gets 11.9%.

**2. Why is T = 0 the right choice for extracting fees and dates into JSON?**
At T = 0 the model always picks the most likely token, so the same input gives (almost) the same output. Fees and dates must be exact and repeatable, and any randomness could change a number or break the JSON format. It also makes testing easier, because we can compare runs. In our own test T = 0 was not perfectly deterministic (14 of 38 answers were worded differently between two runs), so the practical rule is to use T = 0, run twice, and report both.

**3. 500 users, 10 queries a day, 1,000 input and 200 output tokens per query, USD 0.30 and 2.50 per million. Monthly cost?**
Queries per month = 500 x 10 x 30 = 150,000.
Cost per query = (1,000 x 0.30 + 200 x 2.50) / 1,000,000 = (300 + 500) / 1,000,000 = USD 0.0008.
Monthly cost = 150,000 x 0.0008 = **USD 120**.

**4. Why must the retry wait include a random part?**
If many clients fail at the same moment and all wait exactly 1, 2, 4 seconds, they all retry at the same instant and overload the server again. A random part (jitter) spreads the retries over time, so the server can recover.

**5. v2 scores 17/20 on facts and v1 scores 15/20. Is v2 better? Justify with intervals.**
v2 = 17/20 = 0.85 and v1 = 15/20 = 0.75. With the normal formula the intervals are about 0.85 +/- 0.16 (0.69 to 1.00) and 0.75 +/- 0.19 (0.56 to 0.94). With Wilson they are about 0.64 to 0.95 and 0.53 to 0.89. In both cases the intervals overlap heavily, and the difference is only 2 questions. So with 20 questions **we cannot say v2 is better**; it may be just luck. We need more questions (30 or more), or a paired comparison on the same questions.

**6. Why wrap user text in tags, and what does a canary detect?**
Tags tell the model that the text inside is data from the user and not an instruction, which makes prompt injection harder. It is not a full defense: in our attack 4 a fake closing tag partly confused the structure. A canary is a secret string placed in the system prompt; if it ever appears in an output, we know the prompt has leaked. It only catches the exact string, so a translated or encoded leak would pass.
