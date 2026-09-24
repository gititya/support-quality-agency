# Quality Agency

I wanted to test one simple, yet important question: *Can a small local model review a support response like a QA lead, catch the risky parts & explain what needs to be fixed?*

So,  I broke down Support QA into 5 different specialists:

1. **Source of truth** - did the agent use the policy, account data, tool output, or logs?
2. **SOP/Process adherence** - did the agent follow the required support workflow?
3. **Unsupported promise** - did the agent promise a fix, refund, credit, backfill, escalation, or timeline without authority?
4. **Technical diagnosis** - did the technical explanation actually make sense?
5. **Handoff completeness** - could the next agent continue without making the customer repeat everything?


> Here, “agent” means the human support rep.

Each judge gives a structured verdict:

1. Pass or fail
2. how confident it is &  its reasoning
3. the exact piece of text that caused the failure
4. what a safer reply would have looked like.

While this may seem simple & straightforward for anyone who isn't from the Support world, I chose these "judges" because a single good/bad score can hide which requirement failed, so I wanted to test these questions:

1. A support call can be polite, helpful, and still ignore policy.
2. It can technically be correct, yet still skip a required process step.
3. It can give a good customer answer and still leave the next support agent with a useless handoff.


I tried three versions of the review instructions on the same 50 source-of-truth examples, including 30 bad replies:

1. **Short instructions:** missed 4 bad replies.
2. **Detailed instructions:** missed 14.
3. **Detailed instructions with graded examples:** missed 16.

More detail did not help in this run. The question became: **which bad replies was the reviewer letting through?**

It caught useful mistakes in the fictional examples I gave it, but it also let bad replies pass. I would treat this as a second set of checks for a human reviewer. These tests do not show how reliably it would review a real support operation.

## What the judge actually reads

1. Each judge is a small AI model with one job: To read a single support reply and decide whether it holds up.
2. It gets the same three things every time:
	1. the customer's question
	2. the reply the agent sent back
	3. and the source material the agent was supposed to rely on, which is usually a policy document or the customer's account data.
3. In these examples, the judge checks the reply against the supplied policy and account records. It does not independently verify those records.

> **A fictional support example**
> 1. A customer asks, What is your refund policy for annual subscriptions?"
> 2. The supplied policy says the refund window is 30 days.
> 3. The agent writes back that the refund window is 60 days & adds that anything later is handled case-by-case.
> 4. Both are wrong: The agent doubled the window & invented something that the policy never mentions.

## What was built

Along with the 5 judges, I used:

1. A Qwen3-4B-4bit local MLX model as the Primary model
2. A Phi-4-mini-instruct-4bit local MLX model as a challenger.

The pipeline can:

1. Run one judge on a labelled example set.
2. Compare 3 rubric variants with the below. A rubric variant is essentially a set of scoring criteria for the judges.
	1. Short review instructions
	2. Detailed review instructions
	3. Detailed review instructions with sample graded answers
3. Compare Qwen vs. Phi
4. Save metrics  & the disagreement reports.
5. Run all 5 judges on a support case.
6. Synthesize those verdicts into one readable support QA report
7. Run a calibration check to inspect the remaining mistakes before deciding whether another training experiment was worth doing.

The important part is that the report does not hide the evidence. It shows which judge passed or failed, what text caused the failure, what requirement was missing, and when a specific correction rule changed a model verdict.

### For each judge:

1. Built gold pass/fail examples - *essentially, some obvious good and some obvious bad responses for the judge for which the correct answers were pre-decided by me*.
2. Built red-team examples that sound polished but should fail (*think, trick questions*)
3. Tested the three rubric styles: **Vague, Detailed, Detailed with examples.**
4. Ran Qwen and Phi.
5. Measured:
	1. how often it was right
	2. how often it let a bad answer through (the false-safe rate)
	3. how often it failed a good answer
	4. whether its output came back in the format I asked for
	5. whether it pointed at the exact text that was wrong, and
	6. how often it disagreed with the second model.
6. Wrote corrected records for the model mistakes.

> **Reviewing the mistakes**
> A judge was not done when its run finished. For every verdict the model got wrong I classified the miss, wrote a corrected record with the real verdict and the exact failure span, and filed it into the gold set or a future fine-tuning set.
> Each judge also got a short wind-up note: what it catches, what it misses, and whether the challenger was worth running. These corrected records are evaluation annotations, not new model output or fresh performance evidence.

## Findings from the rubrics

I tested the same examples using three versions of the review instructions: short, detailed, and detailed with sample graded answers.

For the **source-of-truth judge** on these 50 examples, the short instructions produced the fewest mistakes. Adding detail did not improve this run. That was worth recording, even though the test does not explain why it happened.

| Rubric | Accuracy | Missed bad answers | Pinpointed the exact problem |
|---|---:|---:|---:|
| **Vague** | **92% (46/50)** | **13% (4/30)** | **87% (26/30)** |
| Detailed | 72% (36/50) | 47% (14/30) | 53% (16/30) |
| Detailed + examples | 68% (34/50) | 53% (16/30) | 47% (14/30) |

The saved result is clear, but its cause is not. The short rubric produced fewer misses than either detailed variant in this run. The measured difference does not establish that short rubrics are generally better or explain why these prompt variants differed.

## Where the models disagreed

1. **A second model caught different mistakes.** On the source-of-truth run, Qwen and Phi disagreed on 14 bad replies. Qwen caught 11 that Phi passed; Phi caught 3 that Qwen passed. Two of those three were deliberately difficult examples.
2. **A confident explanation could still be wrong.** Phi approved a reply promising 90 days of data retention, even though the policy allowed only a 30-day grace period. Its explanation called the two consistent.
3. **Neither model earned the final say.** Comparing them gave me examples to review, not a reason to trust either model on its own.

Then I built the full five-judge support QA pipeline:

```text
support case
    |
    v
five specialist judges
    |
    v
calibrated verdict artifacts
    |
    v
one support QA report
```

Two fictional B2B support cases are included, each written so that all five judges have something to inspect.

1. **Acme Analytics:** an export keeps timing out, and the customer disputes a bill. The rep blames the browser despite evidence pointing to export size, promises an unauthorised fix and credit, and leaves a thin handoff.
2. **Northstar Learning:** 36 signups stop reaching the customer's system after a security-key change. The rep uses the logs to identify the likely mismatch and avoids promising full recovery. But the reply misses the urgency, does not confirm the receiving system is ready before retrying, and drops key details from the handoff.

The Northstar case is the one I used for the final saved run, and it produces this result:

| Judge | Result |
|---|---|
| Source of truth | PASS |
| SOP/process adherence | FAIL |
| Unsupported promise | PASS after calibration |
| Technical diagnosis | PASS |
| Handoff completeness | FAIL |

That is a useful QA outcome. The agent used the logs and diagnosed the issue correctly, but missed the urgency acknowledgement and wrote a weak handoff.

## What I took away

1. **A correct diagnosis is not the whole support job.** Northstar's reply used the logs well but missed the urgency and left a weak handoff. Separate checks made those failures visible.
2. **The reviewer needs QA too.** It approved “If your team tries logging in again now, it should work” without evidence of recovery. Elsewhere, it wrongly treated “I cannot guarantee a complete backfill” as a promise.
3. **A correction is not model learning.** I added a specific rule for that second mistake and kept the original verdict visible. On a separate set of 40 examples, the corrected result was 38 right instead of 37. It still let one bad answer through.

**I need to know which bad replies an AI reviewer approves, not just how often it agrees with the answer key.**

## Why I stopped here

These results did not give me a clear reason to start another training project. The remaining mistakes are recorded, and this small set does not show that the reviewer is ready for a real support queue.

I would use this as an extra set of checks for a human reviewer. I would not let it make the final quality decision on its own.

## How to run

Run all tests:

```bash
python3 -m pytest
```

Run one judge baseline:

```bash
python3 run_judge.py source_of_truth --rubric vague
```

Run the five-judge pipeline on a support case:

```bash
python3 run_pipeline.py --case data/cases/northstar_learning_webhook_secret_rotation.json
```

Synthesize a saved pipeline run:

```bash
python3 synthesize_report.py --run-dir reports/pipeline/<run_id>
```

Run the calibration eval:

```bash
python3 run_pipeline_calibration.py
```

MLX inference needs Apple Silicon with Metal access. In a sandboxed session, tests will run, but model inference may need to run outside the sandbox.

## Files

```text
judges/registry.yaml                         judge definitions and rubrics
data/gold/                                   approved pass/fail examples
data/red_team/                               subtle bad answers
data/corrected/                              corrected model mistakes
data/future_finetune/                        possible future training data
data/cases/                                  five-judge support cases
src/eval_judges/                             parser, prompt builder, scorer, pipeline, synthesis
reports/                                     saved judge runs and pipeline reports
run_judge.py                                per-judge runner
run_pipeline.py                             five-judge support QA runner
synthesize_report.py                        support QA report generator
run_pipeline_calibration.py                 fine-tuning decision eval
```
