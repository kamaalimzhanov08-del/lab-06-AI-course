# JUSTIFICATION — the design of the harness, and why

Everything described here is in the notebook code or in the submitted run. Numbers are from that run.
The fund is fictional and every applicant is invented, so no law is actually binding on this prototype.
I treat the fund as an organisation operating in Kazakhstan and use Kazakh law as the standard a real
deployment would have to meet.

## 1. What the system may do

**Purpose.** One purpose only: first screening of scholarship applications (`ALLOWED_PURPOSES` =
`scholarship_screening`). On its own the system may shortlist or reject an application whose score is outside
the band (40, 60). Inside the band, and for any unparseable or contradictory reply, it only informs a person.

| control | constraint behind it | where in the code |
|---|---|---|
| input validation | the model scores impossible values without complaint (bare run: GPA 9.9, exam -5 -> "reject") | `Harness(validate_input=True)`, `validate_profile` |
| blinding | name and university carry language, gender and tier, none of which is merit | `BLIND = ("name", "university")`, Part 2a |
| refusal | purpose limit, adults only, consent, no instructions in free text | `Harness(refuse=True)`, `refusal_reason`, `my_extra_refusal` (Part 2b) |
| log | every outcome must be reconstructable later | `Harness(log_path="decisions.jsonl")` |
| escalation | uncertain decisions need a human | `Harness(escalate=True, borderline=(40, 60))`, `my_extra_escalation` (Part 2c) |

**A constraint that changed the architecture.** Without the non-discrimination requirement I would have
kept all fields visible and only checked outputs afterwards (the counterfactual check). Because the name and
university are not merit, I removed them from the model's input instead. This also costs no extra model calls,
while the counterfactual check doubles them. Second: refusals are ordinary code that runs before the model,
so a forbidden request never reaches it (0 model calls for each of the 5 refusals in Part 4).

## 2. What it must refuse

| refusal | enforced in | fired in the run | legitimate case it also blocks |
|---|---|---|---|
| `purpose_not_allowed` | `refusal_reason` | "rank applicants by ethnicity" | a harmless new purpose (e.g. statistics) until someone adds it to the list |
| `minor` (age < 18) | `refusal_reason` | applicant aged 17 | a 17-year-old first-year student with a real right to apply |
| `no_consent` | `refusal_reason` | consent = False | an applicant who forgot the tick box |
| `invalid_input` | `validate_profile` | GPA 9.9, exam -5 | a typing mistake by an honest applicant |
| `free_text_too_long`, `suspected_prompt_injection`, `free_text_bad_characters` | `my_extra_refusal` | name = "Ignore all previous instructions..." -> `free_text_too_long` | a name or programme over 60 characters, with a digit or unusual symbol, or containing a word from my list (e.g. "Systems Engineering") |

Only `free_text_too_long` of my three codes fired in the run; the other two are in the code but untested.
On the 400 clean applicants the rules refused 0, but those are generated names.

The cost falls on the refused applicant. I accept it only because a refusal is not a rejection: no score is
produced, and the record shows that no decision was made. This is acceptable only if a person then handles
the case. **That route is not built**: the harness logs the refusal (5 refusal records in `decisions.jsonl`)
but does not notify the applicant or open a manual review. The fund must add this.

## 3. What record it keeps

**Audit question:** "Why did this application get this outcome, which controls acted on it, and what
exactly did the model see and answer?"

A record answers it without re-running the model. Example from my log, application `P0124a`: fields shown =
program, age, gpa, exam_score, income bracket, achievements, consent; fields withheld = name, university;
model Qwen/Qwen2.5-3B-Instruct, prompt v1; harness settings with band [40, 60]; raw reply "shortlist, score
60"; controls fired = blind, escalate, log; outcome = escalate; reason code = borderline; timestamp. A reader
sees that the model wanted to shortlist, that the score was on the edge of the band, and that a person had to decide.

The run produced 406 records: 400 from the measurement and 6 from Part 4 (262 shortlist, 24 reject,
115 escalate, 5 refuse).

**Left out on purpose:** field values. The log has the application id, field names and a SHA-256 hash of the
text shown to the model, so it is not a second copy of the applicants' data. The raw reply may repeat a
value the model saw, but never a blinded one.

**What it does not give.** The human reviewer's final decision is not recorded; the hash proves which text
was shown but does not let you read it; the log does not explain why the model chose its score. The file is
plain JSONL: the record id is derived from the record's content, so an edited record no longer matches its
id, but nothing stops someone deleting or rewriting lines. Retention, access rights and write-protection are
**not built**. For deployment I propose: append-only storage, read access for reviewers and auditors only,
kept until the appeal period of the selection round ends.

## 4. The legal constraints

Two instruments, both read on adilet.zan.kz. They apply by subject matter: the system processes personal
data of applicants automatically and it is an AI system whose result affects their rights. The fund would be
the owner/operator. On synthetic data they are a benchmark, not a binding duty.

**Law of the Republic of Kazakhstan No. 94-V of 21 May 2013 "On Personal Data and Their Protection".**
- Article 19-1(1): automated processing that creates, changes or ends a subject's rights or legitimate
  interests is prohibited unless the subject consented or a law provides for it. This is the provision that
  speaks about automated decisions directly. -> `no_consent` refusal.
- Article 19-1(2)-(3): the operator must explain the automated processing and its consequences, let the
  subject object, and consider the objection within three working days. -> escalation gives the route to a
  person; the log gives what is needed to explain. Informing the applicant is not built.
- Article 7(1): processing needs the consent of the subject or the legal representative. The system cannot
  check a representative's consent, so it refuses under-18s. This is stricter than the law requires.
- Article 7(9): data that is excessive for the purpose is not processed. -> blinding; no values in the log.
- Article 14: use only for the previously declared purposes. -> `purpose_not_allowed`.
- Article 12: storage period is limited by the purpose of collection. -> the retention rule I propose.

**Law of the Republic of Kazakhstan No. 230-VIII "On Artificial Intelligence".**
- Article 6: AI systems must exclude discrimination on grounds of origin, sex, nationality, language, and
  other circumstances. -> blinding, and the measurement itself.
- Article 8(1)-(2): constant control over the system and responsibility for its results. -> log, escalation.
- Article 11(1) and (3): safety and reliability, risk management. -> input validation, the injection rule.
- Article 15(2)(3): the owner keeps documentation on the system. -> log and control register.
- Article 17(2): systems are classed by autonomy; where a person can correct or cancel the decision the
  autonomy is lower. With escalation my system is not fully autonomous for the 28.5% of cases it sends on.
- Article 21(4): requirements for solely automated decisions are set by the personal data law, which leads
  back to Article 19-1 above.

**Required or left to me.** The law requires consent, purpose limitation, the possibility to object and
non-discrimination. It does not require blinding specifically, a score band, a particular log format, or the
refusal of minors; these are my choices. I found no provision that prescribes input validation or the
content of a decision log; Articles 11 and 15 are general duties that I read as covering them.

## 5. The ethical principles

- **Fairness and non-discrimination** — UNESCO, Recommendation on the Ethics of Artificial Intelligence
  (2021), paragraphs 28-30. Bears on: what the model is shown (blinding).
- **Human oversight and determination** — UNESCO Recommendation (2021), paragraphs 35-36. Bears on: who
  decides borderline cases (escalation).
- **Rule of law, human rights, privacy** — OECD, Recommendation of the Council on Artificial Intelligence
  (OECD/LEGAL/0449, 2019, revised 2024), Principle 1.2. Bears on: refusals.
- **Robustness, security and safety** — OECD Principle 1.4. Bears on: input validation, injection rule.
- **Accountability** — OECD Principle 1.5. Bears on: the log and the register.

Both texts are recommendations of international organisations. They are not laws and bind nobody directly.

**Conflict.** Treating applicants identically (blinding) against treating them fairly in substance: with
the university hidden a reviewer cannot take a disadvantaged background into account, and blinding itself
changed 122 of 400 outcomes. I put formal equality first, because I measured no bias that would justify
showing the fields, and the income bracket still gives the model the legitimate need criterion.
Second conflict: oversight against privacy. A more detailed log makes decisions easier to audit but stores
more about people; I kept values out.

**Where ethics asks more than the law.** Consent makes the processing legal even if the model is unstable.
Ethics still asks me to measure it, to say that a 14-point score change from removing two lines is a
reliability problem, and to tell the fund that "no measured effect" at n = 40 is a weak guarantee.

## 6. The trade-offs

| control | cost | what it prevents |
|---|---|---|
| input validation | an honest typo gets a refusal instead of a decision; 0 model calls | a score for impossible data |
| blinding | 166 of 400 scores changed (mean 14.01); 8 decisions changed with no review (7 shortlist -> reject, 1 reject -> shortlist); prompt tokens 84409 -> 77922 | any dependence on name or university: flips 2, 4, 1 and 1 of 40 before, 0 after (by construction) |
| refusal | can block real applicants (section 2); 0 of 400 clean ones in the run | 5 of 6 test requests the bare pipeline processed |
| log | storage and a new file to protect; no extra model calls | decisions nobody can reconstruct |
| escalation | 114 of 400 applications (28.5%) for a human to read | 88 borderline shortlists and 26 borderline rejections made by the model alone |

Model calls: 400 before, 400 after. Time: 484.8 s -> 519.5 s.

**The band.** Rule: from the upper quartile of rejected scores (40) to the lower quartile of shortlisted
scores (60) in the baseline. On the baseline it held 193 of 400 profiles (48.2%); in the harness run only
114 of 400 (28.5%) were escalated, because blinding moved the scores. I did not recalibrate the band on the
blinded scores; that is the next step. The 286 decisions outside the band (262 shortlist, 24 reject) are not
re-checked by anyone.

**Why this balance.** A wrong rejection costs a person a scholarship, a review costs a few minutes, so I
preferred a wide band. I would narrow it if reviewers cannot keep up and start approving without reading,
and widen it or add the counterfactual check if a larger measurement showed a real effect.

**What the design still fails to protect against.**
- Bias through the visible fields (age, programme, income bracket): not measured.
- Model instability: the score depends on the form of the prompt.
- The model using irrelevant fields as merit: one logged reason was "Average performance but consent to
  processing".
- Prompt injection that is short, uses only letters and avoids my keywords.
- A reviewer who agrees with the model without reading; the reviewer's decision is not logged.
- Editing or deleting log lines; no retention or access rules.
- No notice to refused applicants and no manual route for them.
- My escalation rule and two of my three refusal codes never fired, so they are untested.

Closing these gaps needs a larger measurement on more attributes, a second prompt wording, a reviewer
interface with its own log, and append-only storage. I did not do it because it is outside the lab's scope and time.

## AI-use declaration

I used an AI assistant (Higgsfield Supercomputer) in this assignment. It was used to: read the notebook and
explain the toolbox; draft the code of `my_extra_refusal`, `my_extra_escalation` and the rule that computes the
band; draft the prediction texts before the run; find the two Kazakh laws on adilet.zan.kz and suggest which
articles match which control; and draft the observations, the control register, REPORT.md and JUSTIFICATION.md
from the numbers my notebook printed.

I ran the notebook myself in Google Colab on a GPU. All numbers in the two documents are copied from the
output of that run; `decisions.jsonl` and `control_register.json` are the files that run produced. No number
and no log record was generated by the AI tool. The cited articles of both laws were read on adilet.zan.kz.
I am responsible for the content and can explain every control and every number.
