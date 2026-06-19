# Reflection — Day 17 (≤ 200 words)

Answer briefly, in your own words. This is graded on reasoning, not length.

1. **The flywheel.** Day 13 emitted agent traces; today you turned them into an
   eval set and DPO pairs that Day 22 will train on. Which step in
   `traces → Bronze → datasets` would break most silently in production if you
   got it wrong — and how would you detect it?

2. **Decontamination.** Your run dropped 2 of 3 preference pairs because their
   prompts were in the eval set. What concretely goes wrong if you *skip* this
   step and train on those pairs? How would the lie show up in your metrics?

3. **Point-in-time.** The naive join leaked a future `lifetime_spend` into the
   training row. Describe one feature in a system you know that would be
   dangerous to join without an `ASOF`/point-in-time guard.

4. **Graph vs vector.** From `kg_demo.py`, name one question the knowledge graph
   answers well that flat chunk retrieval (`embed.py`) would struggle with, and
   one where the graph is overkill.

1. **The flywheel.** The *decontamination step* (removing eval prompts from DPO pairs) is most prone to silent failure in production (e.g. due to minor prompt formatting changes or casing mismatches). If it fails, evaluation prompts leak into training, artificially inflating evaluation metrics. We can detect this by implementing post-pipeline validation checks (e.g., an assertion that the intersection of eval and DPO prompt sets is empty) and monitoring the dropped-pair count.

2. **Decontamination.** Skipping decontamination causes data leakage. The model trains on the exact prompts it is graded on, leading to severe overfitting. In metrics, this "lie" shows up as an inflated, near-perfect offline evaluation score (e.g., extremely high win rate or low perplexity on the eval set), while real-world online performance on unseen customer prompts remains poor.

3. **Point-in-time.** A user's **current credit score** or **account balance** in a credit scoring/fraud detection system. A naive join would leak post-transaction credit downgrades or balance depletion into the training features of that transaction. The model would learn a false correlation that is impossible to leverage at serving time.

4. **Graph vs vector.** The graph excels at multi-hop queries like *"Where is the warehouse that ships the returnable widget's accessories?"* which requires traversing multiple split relationships. The graph is overkill for simple semantic queries like *"What is the return period of a widget?"*, where flat-chunk vector search is faster and cheaper.
