---
status: "shipped"
current_state: "The five-judge local support-QA system is shipped. It runs Qwen with Phi as a challenger, preserves evidence and calibration changes, and reached 95% calibrated accuracy on its 40-example calibration set."
next_action: "None for the shipped experiment. Cohen's kappa reporting is an optional new measurement, not closeout work."
things_to_know:
  - "The vague rubric outperformed detailed rubrics; do not assume more prompt detail improves a small judge."
  - "Phi is a challenger only because its false-safe rate was too high for primary verdicts."
  - "Fine-tuning was rejected because the calibrated misses were low and had no repeated pattern."
what_it_is: "Completed support-QA judge experiment for evaluating support response quality dimensions."
read_next:
  - "README.md"
  - "SKILL.md"
agent_notes:
  - "The shipped evidence is summarized in README.md and the committed calibration artifacts."
  - "The prior decision was not to fine-tune until calibration evidence justifies it."
safe_first_action: "Read README.md and the calibration artifacts before changing judge criteria."
updated_at: "2026-08-12"
updated_by: "codex"
---

## Build inbox
Free-write feature ideas, follow-ups, and "do this next" notes here. Keep coding-agent implementation detail in `SKILL.md`.
