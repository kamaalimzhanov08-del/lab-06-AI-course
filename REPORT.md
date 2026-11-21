# REPORT — what the measurement showed, who it affects, and whose responsibility it is

All numbers are from my run of `lab06_harness_says_no.ipynb`: Qwen2.5-3B-Instruct on a Colab GPU, greedy
decoding, 200 matched pairs (40 per group), 400 profiles, seed 2026.

## 1. Measurement read-out

The bare agent parsed 400 of 400 replies and shortlisted 356 of 400 profiles (reject: 44).

| attribute | pairs | flips | flip rate [95% CI] | direction | pairs with different scores | mean \|diff\| | floor |
|---|---|---|---|---|---|---|---|
| name_language | 40 | 2 | 5.0% [1.4%, 16.5%] | Kazakh-language name favoured 2/0, p = 0.500 | 11 | 3.12 | null_name |
| gender_coded_name | 40 | 4 | 10.0% [4.0%, 23.1%] | female-coded name favoured 4/0, p = 0.125 | 6 | 1.12 | null_name |
| university | 40 | 1 | 2.5% [0.4%, 12.9%] | flagship favoured 1/0, p = 1.000 | 10 | 2.50 | null_university |
| null_name (floor) | 40 | 0 | 0.0% [0.0%, 8.8%] | – | 7 | 2.00 | – |
| null_university (floor) | 40 | 1 | 2.5% [0.4%, 12.9%] | – | 9 | 2.12 | – |

**No attribute stands clear of its floor on decisions.** Every interval overlaps the interval of its floor.
University gave 1 flip, exactly as many as its floor. The gender-coded name gave the most flips (4 of 40
against 0 of 40), all in one direction: in those pairs female-coded names were shortlisted 90.0% of the time
and male-coded names 80.0%. But p = 0.125, so I cannot call it an effect.

**What 40 pairs could detect.** The exact McNemar test at 0.05 needs at least 6 one-sided pairs of 40 (15.0%)
with none the other way. An effect that moves fewer pairs than that is invisible to this run. So the honest
statement is "no measured effect on decisions", not "no effect": the upper ends of the intervals (16.5% for
name language, 23.1% for gender) are still compatible with a real disadvantage.

**Scores.** Mean absolute differences are also at floor level: 3.12 [1.38, 5.25] and 1.12 [0.25, 2.13] against
2.00 [0.62, 3.62] for null_name; 2.50 [1.12, 4.12] against 2.12 [0.88, 3.62] for null_university. One result
is different: the signed gap flagship minus regional is +2.5 points [+1.1, +4.1], sign p = 0.002. The
university tier moved the score in a consistent direction, but it changed only 1 decision of 40. The signed
gaps for names are not distinguishable from zero (Kazakh +0.7 [-1.8, +3.8], Russian +0.2 [-1.7, +2.1],
English -1.0 [-4.2, +1.9], male vs female -0.4 [-1.2, +0.6]).

**What the floors cover.** They show how much the output moves when another name of the same language and
gender, or another university of the same tier, is shown. They do not cover other wording of the prompt, the
order of the lines (I did not run the order probe) or run-to-run noise (there is none with greedy decoding).
The harness run showed that this other fragility is large: removing two lines changed 166 of 400 scores by
14.01 points on average, far more than any pair difference above.

**Limits.** One model, one prompt, one seed, invented applicants, and a very lenient agent (89% shortlisted),
so few applicants are near the boundary where a flip can happen. A stricter prompt, another model or real
applications could give different results. A stronger estimate needs several hundred pairs per attribute and
several prompt wordings.

## 2. Who is affected

**Wrongly rejected.** In my run the applicants on the losing side of the flips were those with a male-coded
name (4 pairs), with a non-Kazakh name (2 pairs) and from a regional university (1 pair). None of these is a
proven bias, but each flip is a person who would lose a place because of a field that is not merit. The cost
is money for study, possibly the study itself, and a rejection that stays in the fund's records. Applicants
from regional universities also get 2.5 points less on average; if the fund ever ranks by score or raises the
threshold, this group pays first.

**Wrongly shortlisted.** The twin who was shortlisted takes one of a limited number of places, so the cost
falls on a third applicant who is displaced and never learns why. The costs are not equal: a wrong shortlist
can still be corrected at the next stage, a wrong rejection usually ends the process for that person.

**If an effect exists below what I could detect.** Up to 5 of 40 one-sided pairs (12.5%) would pass unnoticed
in this run. With thousands of applications this means many people in the disadvantaged group, each rejected
with no visible reason.

**Parties the brief does not mention.**
- The reviewers: my harness sends 114 of 400 applications (28.5%) to a person. If they are overloaded they
  will approve the model's suggestion without reading, and the control becomes a formality.
- The 8 applicants whose final decision changed after blinding with no human review (7 shortlist -> reject,
  1 reject -> shortlist). They are affected by my control, not by the model's bias.
- Honest applicants refused by my free-text rule (a long name, a digit or an unusual symbol in the name).
- People who do not apply because they expect a machine to reject them, and applicants from universities the
  model has never seen.
- The fund itself, which carries the legal and reputational risk.

**Over time.** If the output is reused (next year's selection, another fund, a published ranking), a small
score gap such as the 2.5 points for regional universities compounds: fewer regional students are funded,
fewer apply, and the rejections become "data" that seems to confirm the pattern.

## 3. Professional responsibility

**Mine, as the developer of the harness.**
- Measure before and after and report honestly. Done: the tables above. I must not claim that the agent is
  unbiased (I measured no effect at n = 40), and I must not claim that the harness removed a bias (the zero
  flips after blinding are by construction).
- Test every control. Done: all five refusal reasons fired in Part 4 and 0 of 400 clean applicants were
  refused. Not done: my escalation rule never fired, and the refusal rule was not tested on real names.
- Tell the deployer the limits and the price: 28.5% of applications go to a person; the band (40, 60) was
  calibrated on the bare agent (planned 48.2%) and must be recalibrated; blinding changed 8 unreviewed decisions.
- Set stop conditions. Proposed, not built: stop the system if the share of escalations or refusals leaves an
  agreed range, or if a new measurement shows 6 or more one-sided flips of 40 for any attribute.
- Left to do: a larger sample, the order probe, a second prompt wording, a check that the log cannot be edited.

**The institution's.** Deciding the purpose and the selection criteria, getting real consent and explaining
the automated processing to applicants, choosing and training reviewers and giving them time, handling
objections and appeals, deciding how long the log is kept and who reads it, and repeating the measurement
when the model or the prompt changes.

**Where they conflict.** The obvious pressure is to narrow the escalation band or switch escalation off,
because 114 reviews per 400 applications cost staff time. My responsibility then is to show the numbers
(without the band 114 borderline decisions are made by a model whose score moves by 14 points when two lines
are removed), to refuse to present the system as "checked by a human" if it is not, and to record the
institution's decision in writing. If the institution wants to remove the refusal of applicants without
consent or of other purposes, I should not ship that version: this is the line where the law, not only my
opinion, says no (see JUSTIFICATION.md, section 4).
