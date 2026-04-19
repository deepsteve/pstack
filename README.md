  ⎿  # PSTACK                                                                                                                                                  
                                                                                                                                                                 
     A coordinate-descent system-prompt optimizer for small on-device models.                                                                                    
                                                                                                                                                                 
     The hypothesis: for small models (≤4B parameters, INT4-quantized, running on a phone), the **system prompt dominates output quality** more than it does for 
     frontier models. PSTACK is a focused optimizer that exploits that.                                                                                        

     ## The problem

     You have:

     - A frozen model `M` (typically a small on-device LLM you can't fine-tune).
     - A seed system prompt `P_0` that mostly works.
     - A pass/fail evaluator `score(output) → bool` (compile check, format check, classifier, etc.).
     - A frozen evaluation set `E` of `(concept, user_prompt)` pairs.

     You want a better system prompt `P*` that maximizes the pass rate of `M(P, user_prompt)` against `E`, without retraining the model.

     ## What PSTACK does

     Hill-climbs the system prompt one concept at a time:

     1. Run `M` against the full eval set with the current prompt `P`.
     2. Pick the concept with the most headroom (worst per-concept pass rate).
     3. Generate `k` candidate prompt edits targeting that concept (e.g. add a snippet, add a rule, add a reference example).
     4. Run a **per-concept hill batch**: each variant × `n ≥ 10` runs against just that concept's user prompts.
     5. Pick the winning variant by pass rate; tie-break on token cost.
     6. **Validate the winner against the full eval set** (not just the targeted concept) to catch cross-concept regressions.
     7. If clean, lock in. If regression, try a smaller-edit variant or mark the concept ceiling-bound.
     8. Repeat.

     The driver is Claude Code or any LLM-with-tools. The intelligence — proposing variants — is the model's; the bookkeeping (running batches, scoring, locking,
      regression-checking) is the algorithm.

     ## Why coordinate descent

     Full-prompt rewrites are uninterpretable. If a 5-edit rewrite gains 8 pp, you can't attribute the gain to any specific edit, so you can't iterate further or
      generalize the lesson. PSTACK's "one change per variant, one variant locked at a time" rule keeps every step diagnosable.

     ## Invariants (read these)

     These are what separate a useful run from noise:

     1. **Hold the model and the eval set constant for the whole run.** Quietly swapping either invalidates round-over-round comparisons.
     2. **`n ≥ 10` runs per variant.** At any non-zero temperature, single samples are noise (±2/n per concept). 10 minimum, 20 if you want tight CIs.
     3. **Every hill includes the unmodified baseline as `V_0`.** Without the control in the same batch, sampling drift will fool you into locking in null edits.
     4. **One change per variant.** Stack changes across rounds, not within a single hill.
     5. **After every lock, re-validate against the full eval set.** Per-concept hill scores don't predict cross-concept regressions — that's where the real bugs
      hide.

     ## Variant generation defaults

     Empirically across small on-device models:

     - **`V_0` baseline** — control, required.
     - **`V_1` declarative rule** — one-line constraint addressing the failure. Usually loses; included as a comparison floor.
     - **`V_2` in-context snippet** — 4–8 lines showing the correct pattern. Usually wins decisively (small models copy structure verbatim).
     - **`V_3` full reference output** — complete worked example. Often matches `V_2` at much higher token cost.
     - **`V_4` user-prompt rephrase** — rewrite the user prompt itself when the eval allows. Sometimes wins; not always applicable.

     A variant is *one* of these. Combining (e.g. snippet + rule) destroys attribution.

     ## Anti-patterns

     - **Iterating on a different model than the deployment target.** Same architecture at different quantization (full-precision vs INT4) or different runtime
     (CPU vs accelerator) produces different failure modes. Iterate on the deployment configuration.
     - **Skipping the `V_0` control.** Sampling noise will hand you false wins.
     - **Single-sample evaluations.** At T > 0, n=1 is noise.
     - **Adding to the eval set during a run.** Leaks signal, breaks round-over-round comparability. If the eval set is wrong, freeze a new one and restart from
     `P_0`.
     - **Trusting prose rules to bind on small models.** They often don't — the model regurgitates structure from the most recent example regardless of
     declarative constraints. Demonstrate the desired pattern instead of asserting it.

     ## Quickstart

     ```bash
     git clone <this repo>
     cd pstack
     npm install
     ```

     Configure `pstack.config.json`:

     ```json
     {
       "model": { "runner": "node ./runners/cactus.mjs", "options": { ... } },
       "seed_prompt_path": "./prompts/p0.txt",
       "eval_set_path": "./eval/baseline.json",
       "scorer": "./scorers/glsl-compile.mjs",
       "n_runs": 10,
       "concept_priority": "headroom",
       "out_dir": "./runs"
     }
     ```

     Run:

     ```bash
     pstack hill --concept worst        # one round
     pstack auto                        # loop until stopping criteria
     pstack report ./runs/round-N       # human-readable summary
     ```

     Outputs after each round:

     - `./runs/round-N/prompt.txt` — the locked prompt
     - `./runs/round-N/score.json` — per-concept and overall pass rates
     - `./runs/round-N/raw.jsonl` — every inference (preserved for re-scoring)
     - `./runs/round-N/decision.json` — locked / skipped / regression-vetoed, with reason

     ## Provenance

     PSTACK was extracted from a hackathon project optimizing a GLSL-shader system prompt against on-device Gemma 4 E2B (INT4). Headline numbers from that run:

     - ~1,200 inferences across one overnight session.
     - Best per-concept gains: +30 to +60 pp from a 4–8 line in-context snippet vs. a one-line declarative rule.
     - A naively stacked prompt ("add all the winning snippets at once") *regressed* the baseline — the snippet bias displaced a structural anchor (a mandatory
     `precision mediump float;` declaration). A one-line re-anchor at the bottom of the prompt recovered it. This is the kind of thing PSTACK's "always
     re-validate after locking" rule is for.

     ## What PSTACK is not

     - Not a fine-tuning tool. It only changes prompts.
     - Not for frontier models. The hypothesis (system prompt dominates) doesn't necessarily hold above ~7B params.
     - Not a benchmark. It's a wrapper around your benchmark.
     - Not autonomous "self-improving prompts." A model proposes variants, but the algorithm is the human.

     ## License

     MIT.
