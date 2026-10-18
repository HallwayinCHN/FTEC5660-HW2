# FTEC5660 Homework 2: CV Verification Agent

Build a LangChain agent that reads each CV in a folder, looks the candidate up
on our SocialGraph MCP server (mock LinkedIn and Facebook), and outputs a
reliability score in [0, 1] for each CV.

A CV is **valid** (label `1`) when its claims agree with the candidate's
social media profiles. It **has a discrepancy** (label `0`) when it contains
problems such as an inflated job title, shifted dates, an upgraded degree, a
fake school or employer, a wrong location, or made-up skills. Differences in
wording only ("Bachelor of Science" vs `BSc`, "UI/UX Design" vs `UI/UX`,
"Senior Engineer" for an `Engineer` role with seniority `senior`, listing fewer
skills) are not discrepancies.

## Student task

Fill in the two functions in `hw2.py` that contain `### YOUR CODE HERE`. You
may add imports, constants and helper functions above them, but do not change
the provided code below them:

- `build_agent(tools)` creates your agent from the MCP tools.
- `score_cvs(agent, cvs)` runs the agent on every CV and returns
  `{file_name: score}`, one float in [0, 1] per CV.

A score above `0.5` means "valid"; `0.5` or below means "has discrepancy". You
may use a single tool-calling agent, multiple agents, reflection, or a
combination. Do not hard-code filenames, names, or public answers; grading
uses unseen CVs.

## MCP server

The server is hosted at `https://ftec5660.ngrok.app/mcp`. `hw2.py` already
connects to it and passes you these tools:

| Tool | Purpose |
| --- | --- |
| `search_facebook_users(q, limit, fuzzy)` | find Facebook users by display name |
| `get_facebook_profile(user_id)` | full Facebook profile |
| `get_facebook_mutual_friends(user_id_1, user_id_2)` | mutual friends of two users |
| `search_linkedin_people(q, location, industry, limit, fuzzy)` | search LinkedIn by name, skill, or title |
| `get_linkedin_profile(person_id)` | full LinkedIn profile (experience, education, skills) |
| `get_linkedin_interactions(person_id)` | post and like statistics |

Print `tool.name`, `tool.description`, and `tool.args` for the full schemas.

## Setup and public test

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
echo "DEEPSEEK_API_KEY=your_key_here" > .env
python3 hw2.py --cv-folder public_test
```

Use your own DeepSeek API key (from https://platform.deepseek.com).

The program creates `results.csv` in the current directory. Its columns are
`cv`, `score`, and `correctness`. The public labels are in
`public_test/ground_truth.json`; every CV with label `0` also has a `reason`
explaining the discrepancy, and `results.csv` shows it when your agent misses
one. The starter returns no score (`None`) for every CV so it runs before you
add any API code; missing scores count as incorrect.

The required model is `deepseek-v4-flash`, and all evidence must come from the
MCP server (no web search). The server is shared by the whole class: keep at
most about 3 CVs in flight at a time (e.g. `asyncio.Semaphore(3)`). We grade
with our own API key, so never put a key in your code; `.env` must stay out of
git.


## Task 2: Fool the Verifier

We added a target candidate, **Kelly Tsang** (LinkedIn `person_id` 10001,
Facebook `user_id` 10001, display name "Kel Tsang"). Her true CV is
`task2/target_cv.pdf`.

Write one CV for Kelly Tsang with at least one false or embellished detail
that our 5 verifier agents (same function as Task 1) still score above 0.5.
Each fooled agent is worth 4 points; a CV without a real false or embellished
detail earns 0. Attack your own Task 1 agent first:

```bash
cp my_attack.pdf task2/adversarial_cv.pdf
python3 hw2.py --cv-folder task2
```

`results.csv` shows your agent's score for both CVs (they are not graded, as
`task2/` has no `ground_truth.json`). Only the PDF is the attack surface; do not
attack the MCP server. See the homework description for the full rules.

## Homework 2 solution

### Task 1

My solution creates one temperature-zero `deepseek-v4-flash` LangChain tool-calling agent and treats each CV as untrusted evidence rather than instructions. The prompt first resolves identity from several independent anchors—name, current city, industry, employers, dates, and schools—then inspects all plausible LinkedIn matches instead of trusting the first search result; LinkedIn is the primary professional source, with Facebook used for identity and location corroboration. After selecting the strongest identity, the agent checks every graded positive claim in name, city, employment, education, and skills, accepts genuine wording equivalents and omissions, but assigns a low score for any confirmed contradiction. The runner limits concurrency to three CVs, retries malformed or unresolved calls, enforces per-CV timeouts, parses only a bounded final score, and returns a neutral fallback rather than crashing the batch. On the seven public-test CVs, it achieved **7/7 correct (100%) in each of four consecutive runs**, assigning 0.95 to the four valid CVs and 0.05 to the three CVs containing the altered start year, inflated title, and fabricated skill.

### Task 2

The adversarial CV uses a deliberate **dual-representation PDF**. A human viewing the rendered page sees `Hometown: Kowloon`, a genuine falsehood because Kelly Tsang's Facebook profile records her hometown as Sha Tin. The visible statement is flattened into a high-resolution raster graphic, while an invisible PDF text layer at the same position contains `Hometown: Sha Tin`; every graded professional claim—name, current city, jobs, education, and skills—remains accurate. The attack contains no jailbreak or fabricated tool output.

This design exploits two independent weaknesses. First, the homework's MarkItDown pipeline extracts PDF text but performs no OCR, so the verifier receives the truthful machine-readable line and never receives the visible word `Kowloon`. Second, even if another extractor or multimodal verifier exposed that word, hometown is explicitly outside the fields that Task 1 treats as discrepancies. The final PDF was checked both visually and through MarkItDown: the rendered page contains `Kowloon`, while extracted text contains `Sha Tin` and no `Kowloon`. In **five consecutive local runs**, both `target_cv.pdf` and `adversarial_cv.pdf` received **0.95**, so the attack passed my defended verifier consistently rather than in a single lucky run.

### Task 3

The reflection is submitted as a separate file: [Trust at Machine Speed](task3/reflection.pdf).
