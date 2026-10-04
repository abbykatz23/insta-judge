# Jev (TypeSafe AI): judging practices for per-user post filtering

_Researched 2026-10-04. This extends [jev.md](./jev.md), which covers what Jev is, the API shape, pricing, limits, and SDKs. None of that is repeated here. Everything comes from primary sources: docs.typesafe.ai (every page discovered through [llms.txt](https://docs.typesafe.ai/llms.txt) that bears on question design, patterns, and cookbooks), the [typesafe-ai/skills](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md) agent skill, and typesafe.ai blog posts. Lines marked **Inference** are my reading of those sources, not something a source says. Many cookbook numbers were produced on `jev-1.12`, not the current `jev-1.13.0`. Each cookbook says which version it used, and I note it where it matters._

## Summary

The docs say the same thing in many places: **one narrow judgment per question, and combine the answers in code.** "Hiding several judgments inside one question" is listed as something to avoid ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)). Score and Noul pages both say to split multi-part criteria into one question per part ([Score](https://docs.typesafe.ai/primitives/score), [Noul](https://docs.typesafe.ai/primitives/noul)). The agent skill speaks to our OR case directly. It says "Weighted scores suit compensating preferences; an 'any serious violation' rule needs separate conditions," and to use one Noul "per label when several may apply" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)). Extra questions in one request are said to cost only their tokens and barely change latency ([primitives](https://docs.typesafe.ai/primitives)). A `choice` returns probabilities that sum to 1, so it always ranks something first, even when nothing fits. It also has a documented first-option bias. The cookbooks that need "does anything fit?" ask a separate Noul for that ([Line-by-line search](https://docs.typesafe.ai/cookbooks/semantic_find), [Skill suggestion](https://docs.typesafe.ai/cookbooks/skill_suggestion)). The docs show three ways to combine answers: weighted averages ([Composite scoring](https://docs.typesafe.ai/patterns/composite-scoring)), "any hazard fires" rules with precedence ([Guardrails](https://docs.typesafe.ai/cookbooks/llm_guardrails)), and ordered first-match gates ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages)). For uncertainty, the documented pattern is an explicit "uncertain" band that sends cases to review. Prompt injection is a documented weakness. The only mitigations documented are explicit criteria, testing, and a detection Noul, and the docs say this is "not a security boundary." No source discusses criteria written by end users.

## 1. One combined question vs one question per interest

**What the docs say:**
- "Ask for one snap judgment per question… If the judgment you want depends on several independent factors, ask about each factor separately and combine the answers with your own logic… When priorities shift, change the value of weights rather than rewriting a prompt" ([primitives](https://docs.typesafe.ai/primitives)).
- The building guide calls decomposition "probably the most important concept in this guide. Broad questions hide several judgments behind one answer." Its bad example is a single `is_spam` Noul, and its good example is six atomic Nouls ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)).
- **Multi-part criteria:** "Keep each Score question to one dimension. If a description says 'punctual and smart and experienced', the question is measuring three things, and an input that is high on one and low on another can't be placed. Confidence drops and the score means less" ([Score](https://docs.typesafe.ai/primitives/score)). For Noul: "If a question has two conditions… the model has to judge both at once and the value means less. Ask two Nouls and combine them in code" ([Noul](https://docs.typesafe.ai/primitives/noul)).
- **Literal reading:** jev-1.13 "answers the question you wrote, not the one you meant… Where interpretation is unavoidable, split it into two literal questions and combine them in code" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- **Irrelevant content:** "Accuracy falls as the state grows with content unrelated to the decision… send only the fields the question needs. When it's not possible to filter in state, you can use a Noul to filter for relevance" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- **Independence:** every question in a request "sees the same state, is evaluated independently… One question's answer is not hidden context for another. You can add or remove questions without changing the others' results" ([primitives](https://docs.typesafe.ai/primitives)). The [Parallel questions](https://docs.typesafe.ai/cookbooks/parallel_questions) cookbook tested this with 13 questions sent batched and unbatched, and found "no batching effect."
- **Cost and latency:** "Adding questions barely changes the response time and costs only the tokens for the extra questions" ([primitives](https://docs.typesafe.ai/primitives)). The state is ingested once. Batching 13 questions was "12.2x cheaper and 10.0x faster" than 13 separate calls ([Parallel questions](https://docs.typesafe.ai/cookbooks/parallel_questions)). The primitives page quotes the same experiment as "11.5x cheaper and 9.6x faster." The two pages disagree, but both show a large saving. An 8-question Choice rubric averaged 114 ms and about $0.000046 per call ([Self-consistency: choices](https://docs.typesafe.ai/cookbooks/consistency_choice_cookbook)).
- **Limits:** no maximum number of questions per request is documented. The binding limit is tokens: 64k for the state plus all questions, and 32k for the state plus the longest single question ([Models](https://docs.typesafe.ai/models)).
- **Where per-question context goes:** a question's `instructions` can be an object, "with the question in one field and supplementary data in the others." The documented example puts a database record in `potential_duplicate` and asks one Noul per record, all in one request ([Noul](https://docs.typesafe.ai/primitives/noul#structured-instructions), [How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one#use-structure-in-the-questions)). [Line-by-line search](https://docs.typesafe.ai/cookbooks/semantic_find) puts the varying query in `instructions` and keeps the state the same across searches. The [RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages) cookbook does the reverse: query and passage both go in the state, with one request per pair.

**Inference:** a single "everything I value" description is exactly the "punctual and smart and experienced" case. A post that strongly matches one interest and none of the others "can't be placed," and literal reading makes the outcome depend on wording. The documented shape is one question per interest in a single request, with the post as `state` and each interest's text in that question's structured `instructions`. Putting all of a user's interests into the shared `state` would make every other interest a distractor for each question. Splitting per interest costs a few hundred extra tokens per request, which is negligible at the documented price.

## 2. noul vs score vs choice for "is this post relevant to interest X"

**Documented guidance:**
- Choice: "one of a known set of options with no order between them." Score: "falls on a spectrum and you can describe what each point… means." Noul: "a clean yes/no question where the probability itself is the useful signal." "Prefer the one whose answer your code can act on directly… A Noul maps onto an `if`" ([primitives](https://docs.typesafe.ai/primitives)).
- The agent skill adds: Noul is the "probability of yes… use one per label when several may apply." Score is for "degree along a described dimension; use comparable per-item Scores for graded ranking" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).
- **Noul semantics and calibration:** a Noul "is not a scale of the thing you asked about. It is the probability that the answer is yes… A middle value can mean medium experience or an unclear case" ([Noul](https://docs.typesafe.ai/primitives/noul)). The calibration claim covers groups of predictions: "Outcomes assigned a probability of 0.8 should occur about 80% of the time… not a guarantee about any single answer" ([AI primer](https://docs.typesafe.ai/introduction/machine-learning-primer)). "Make the boundary between yes and no unambiguous… When the boundary is subtle, add `criteria` with `true` and `false` descriptions… try your questions with and without `criteria`" ([Noul](https://docs.typesafe.ai/primitives/noul)).
- **Noul works well for relevance:** the RAG cookbook's relevance floor is `is_relevant`: "Does this passage address the subject of the query?" It is thresholded at 0.45 ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages)). The re-ranking cookbook sorts candidates by the raw noul ([Re-ranking](https://docs.typesafe.ai/cookbooks/rerank_typesafe)).
- **Score with a labelled middle level:** the entity-alignment cookbook picks a 3-level Score (different / "related, but possibly not the same" / same) "because we want to attach a semantic label… directly to each outcome, including the middle outcome. A Noul… could accomplish this indirectly through thresholding… a Choice… would lose the ordered relationship." It routes on cut points at 0.5 and 1.5, which it says requires "no threshold you had to fit" ([Entity alignment](https://docs.typesafe.ai/cookbooks/entity_alignment)).
- **Choice properties that matter here:**
  - Probabilities sum to 1, so "some line ranks first even when the document doesn't answer the question… Unlike the Choice probabilities, the Noul probability doesn't depend on the other options, so it can fall near zero" ([Line-by-line search](https://docs.typesafe.ai/cookbooks/semantic_find)).
  - A Choice returns one winner. An input that belongs to two options splits the mass and lowers confidence. In the docs example, a ticket for two teams scored 0.61/0.35 with confidence 0.42, and the code notified the runner-up when its share exceeded 0.25 ([Choice](https://docs.typesafe.ai/primitives/choice)).
  - **First-option bias:** "the order of a Choice's options can affect the answer, and jev-1.13 leans toward the option that comes first. Instead: reorder the options to double check that the answer stays consistent" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
  - **"None" options:** "add an `other` or `none of the above` option when the list might not cover every input" ([primitives](https://docs.typesafe.ai/primitives), [Choice](https://docs.typesafe.ai/primitives/choice)). The skill says: "Include a no-match outcome when nothing may fit; use a separate presence judgment when it is independently useful" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)). The skill-suggestion cookbook gave its 182-option Choice **no** "none" option. It decided whether to suggest anything with separate Nouls, and explained: "The Choice settles *which* skill, and the nouls settle *whether* to say anything at all" ([Skill suggestion](https://docs.typesafe.ai/cookbooks/skill_suggestion)).
  - Size: the API allows up to 255 options ([API reference](https://docs.typesafe.ai/api)). One cookbook says a Choice "works reliably up to roughly 240 options" ([Classification using confidence](https://docs.typesafe.ai/cookbooks/classification_using_confidence)).

**Inference:** one `choice` over the user's interest categories plus "none" maps poorly onto "pass if relevant to *any* interest":
1. A post that fits two interests, such as a vegan cookbook that is also a book list, splits probability between them and looks "uncertain" even when it should clearly pass. Summing the non-"none" mass would partly compensate, but that is my workaround, not a documented one.
2. "None" competes with the real options and is subject to order bias.
3. Adding or removing one interest changes every other option's probability. Per-interest Nouls do not have this problem.

One per-interest `noul` matches "use one per label when several may apply" and the RAG `is_relevant` pattern. A per-interest `score` makes sense if we want graded ranking or a labelled "borderline" level. A `choice` is still useful as a speculative extra, for example to label which interest matched for display.

## 3. Writing score levels

- "Describe situations, not degrees. 'Broken or degraded feature, but workaround exists' gives the model something to match… 'Moderately severe' doesn't" ([Score](https://docs.typesafe.ai/primitives/score)).
- "Every level is evaluated separately. The model doesn't see a level's number or its neighbours, so 'worse than the previous level' means nothing to it, and numbers in the descriptions or the instructions don't help." In the docs example, levels written as `"0","1","2"` gave a split 0.55 where descriptive levels gave 0.0 at confidence 1.0 ([Score](https://docs.typesafe.ai/primitives/score)). The skill puts it as "Score levels must describe concrete situations and stand on their own" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).
- **Number of levels:** "Use as many levels as you can describe distinctly, up to 10. Three is fine. Don't add levels you can't describe distinctly." A rare extreme that needs different handling should get its own level ([Score](https://docs.typesafe.ai/primitives/score)).
- **Objects with examples:** start with strings, and switch to objects such as `{what, examples}` "when the model keeps scoring between two neighbouring levels on inputs you think are clear… Use the same field names on every level." Examples "only help when they look like your real inputs." An unrelated example left the result unchanged ([Score](https://docs.typesafe.ai/primitives/score)). Choice options can also carry `not_for` to sharpen boundaries. Field names are free-form and "The model sees the names along with the values, so use short names that label what follows" ([Choice](https://docs.typesafe.ai/primitives/choice), [Advanced: structure](https://docs.typesafe.ai/primitives/advanced)).
- **Consistency between instructions and criteria:** "treat the criteria as an extension of the instruction" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- **Testing wordings:** "Two wordings of the same scale can behave differently on your data." "Higher confidence does not establish which answer is correct. Choose examples with known expected levels, then test the revised descriptions on separate inputs" ([Score](https://docs.typesafe.ai/primitives/score)).
- **Don't treat scores as measurements:** do not "compute the exact magnitude of a number between two levels… You can use the expectation to check if it passes a particular threshold" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- **Criteria written by end users vs developers: not covered by any source.** The closest material:
  - "Agents aren't great at writing questions, so expect to edit collaboratively with them" ([Agent skill](https://docs.typesafe.ai/agent-skill)).
  - Domain rules belong in `instructions`/`criteria` ([Models](https://docs.typesafe.ai/models#customizing-jev)).
  - The skill suggests letting "code or user controls change weights, thresholds, rankings, and views". That covers user-adjusted weights, not user-written criteria ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).
- **Inference:** a free-text user interest such as "lists of books someone found inspiring" is likely to contain degree words, multiple conditions, or negations, all of which the docs warn against. Some developer-controlled template (fixed question wording, with the user's text as a named field) is probably needed.

## 4. Combining answers

**Composite scoring in detail** ([Composite scoring](https://docs.typesafe.ai/patterns/composite-scoring), [Score](https://docs.typesafe.ai/primitives/score#splitting-a-complex-judgment-into-several-score-questions)):
- Use one Score per independent dimension, all in one request.
- "Normalize each score… Divide each score by its top level number, `len(criteria) - 1`, to put every score on 0 to 1. Then the weights mean what they say."
- Combine with a weighted sum (example: `0.6*severity + 0.3*frustration + 0.1*report_quality`). The same answers can feed several weight profiles (senior IC vs engineering manager). "When the combined result doesn't match what your team would decide, change them in code and run again."
- Nouls can be weighted in the same way, including inverted signals (`0.2 * (1 - contradicts_context)`) ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)).
- For learned weights: "use the probabilities as features in a downstream classical machine-learning model" ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one), [Autoresearch](https://docs.typesafe.ai/cookbooks/autoresearch_feature_discovery)).

**Any-of (OR), max, and gating:**
- "Weighted scores suit compensating preferences; an 'any serious violation' rule needs separate conditions" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)). This is the only explicit statement contrasting OR with weighted averages.
- **Any-of with precedence:** in the guardrails cookbook, each hazard Noul is checked against its own review and action thresholds. Any hazard that fires triggers its action, and a precedence list decides between actions. A severity Score can upgrade a review to a block ([Guardrails](https://docs.typesafe.ai/cookbooks/llm_guardrails)).
- **Max over candidates:** in the skill-suggestion cookbook, "a shortlist whose highest one [`fits` noul] lands under 0.30 gets dropped entirely." Its gate is the *mean* of three request Nouls, with one Noul inverted ([Skill suggestion](https://docs.typesafe.ai/cookbooks/skill_suggestion)).
- **Ordered first-match gates:** in the RAG cookbook, checks run in a fixed order: injection > 0.70 excludes, contradiction > 0.70 marks a conflict, relevance < 0.45 excludes, evidence > 0.55 includes, and anything else is excluded. "Injection comes first because it is a security decision" ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages)).
- **Speculative questions:** ask questions that only matter on some branches and "ignore uncertainty on unused branches" ([Speculative fan-out](https://docs.typesafe.ai/patterns/fan-out), [SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).

**Confidence, abstention, and low-confidence handling:**
- Three paths: high confidence → act, medium → confirm or review, low → "Do not act. Route to a human…" Thresholds "scale with risk" ([Confidence](https://docs.typesafe.ai/confidence), [Confidence-gated routing](https://docs.typesafe.ai/patterns/confidence-routing)).
- **Noul has no confidence.** Use `|2p − 1|` if you want one, or an explicit band. Example: "`uncertain` from 0.30 through 0.70… Uncertain cases go to a human… The band is illustrative… Set production boundaries from labeled examples and from the cost of incorrect decisions and of review" ([Confidence](https://docs.typesafe.ai/confidence#noul), [Self-consistency: nouls](https://docs.typesafe.ai/cookbooks/consistency_noul_cookbook)). The Noul page uses YES = 0.8 and NO = 0.2, with anything between sent to review ([Noul](https://docs.typesafe.ai/primitives/noul)).
- **Choice abstention:** require a top probability of at least 0.60, otherwise return "uncertain." This raised run-to-run agreement from 90.8% to 99.2%, with 74.2% of answers decided automatically ([Self-consistency: choices](https://docs.typesafe.ai/cookbooks/consistency_choice_cookbook)). Alternative statistics are top probability and top-to-second ratio ([Confidence](https://docs.typesafe.ai/confidence)).
- **Fallback to a coarser answer:** below confidence 0.9, report the parent category instead. Accuracy on that half went from 40% to 70% ([Classification using confidence](https://docs.typesafe.ai/cookbooks/classification_using_confidence)).
- **Uncertainty on a composite:** the building guide applies a band to the weighted sum (`0.4 < spam_risk < 0.6` → human review) ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)).
- **Caveats:**
  - "If all you care about is choosing the best option, you just need to choose the option with the highest confidence (rather than setting a confidence threshold)" ([Agent skill](https://docs.typesafe.ai/agent-skill)).
  - "low confidence need not invalidate a harmless preference choice," and confidence is "not overall workflow correctness or permission to act" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).

**Inference:** "passes for this user" is an any-of rule over interests, so the documented fit is `pass = max_i(noul_i) ≥ T`, or any interest over its own threshold, as in the guardrails pattern. A weighted average across interests would dilute a strong single-interest match, which is exactly what the skill warns about. Posts in an uncertain band (for example a max between 0.3 and 0.7) could be held back or shown with a "maybe" marker. There is no human reviewer in this product, so "abstain" has to mean some product behavior.

## 5. Shaping the state

- **JSON vs string:** "Use an object for most requests so each part of the state has a descriptive name… A string is suitable when the use case is simple and requires only one piece of text." Keep content in the state and judgments in the questions ([State](https://docs.typesafe.ai/concepts/state)).
- **Pointing questions at fields:** name a field with a backticked dot-and-index path, for example `` `ticket.messages[0].text` ``, "including the backticks" ([primitives](https://docs.typesafe.ai/primitives#reference-specific-fields)).
- **Which fields to include and how to trim:** "Include only the context relevant to the current questions… avoid distractions and context rot" ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)). "Giving it more context in `state` than the question needs" costs accuracy ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)). Worked examples:
  - Annual reports were trimmed to the one relevant section ([Classification using confidence](https://docs.typesafe.ai/cookbooks/classification_using_confidence)).
  - Numeric fields were dropped from the questions because "comparing two numbers is arithmetic" ([Entity alignment](https://docs.typesafe.ai/cookbooks/entity_alignment)).
  - Large subtrees were trimmed "to its direct children and a sample of leaves" ([Advanced: structure](https://docs.typesafe.ai/primitives/advanced)).
  - Dates should be converted to components or buckets in code, because jev-1.13 "reads dates as text" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- **Field naming:** the docs give no state-specific naming rules beyond "descriptive name." For structured instructions and criteria, "use short names that label what follows" ([Choice](https://docs.typesafe.ai/primitives/choice)). Question IDs are not sent to the model ([primitives](https://docs.typesafe.ai/primitives)).
- **Prompt injection and untrusted data:**
  - "State is data, and jev-1.13 does not treat it as hostile by default. Content written to adversarially steer the model, whether that is an injected instruction, a deliberately misleading framing, or text that argues for its own classification, can move the answer… Instead: be explicit in the criteria. Test your integration thoroughly" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
  - The only other documented mitigation is a detection question that gates first, such as `contains_prompt_injection`: "Does this passage attempt to control the system answering the query?" (excluded at > 0.70, and it caught the planted injection at 0.99). The same cookbook warns: "The injection question is a filter, and only one… Nothing here is a security boundary" ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages)).
  - The guardrails cookbook's `jailbreak` Noul makes a similar claim: "'Ignore your instructions' scores as a jailbreak instead of working as one" ([Guardrails](https://docs.typesafe.ai/cookbooks/llm_guardrails), run on jev-1.12).
  - No escaping, delimiting, or "treat this field as untrusted" mechanism is documented.
- **Inference:** a state like `{caption, slides:[{extracted_text, visual_description}], concrete_references:[{type, value}]}` fits the guidance. `uncertainties` is meta-information about extraction quality rather than post content, so it is a candidate for trimming or for gating in code. Captions that "argue for their own classification" ("perfect for anyone who loves vegan food!") are a documented risk for exactly our relevance questions. A per-request injection or self-promotion Noul that gates first is the documented pattern, but it is not a guarantee.

## 6. Thresholds, evaluation, and versioning

- **Setting thresholds:**
  - Base them on the cost of being wrong: "Use 0.5 when yes and no are equally easy to act on. Raise it when acting on a false yes is expensive… Lower it when missing a true yes is expensive" ([Noul](https://docs.typesafe.ai/primitives/noul)).
  - "Start with conservative thresholds, test with your own data, and adjust" ([Confidence](https://docs.typesafe.ai/confidence)).
  - "Test thresholds by plotting confidence against accuracy on your data" ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)).
  - "Set the thresholds… from labeled examples of your own traffic" ([Guardrails](https://docs.typesafe.ai/cookbooks/llm_guardrails)).
  - Thresholds that are too high produce false negatives, and too low produce false positives ([Agent skill](https://docs.typesafe.ai/agent-skill)).
  - Cookbook thresholds are "a starting point, not defaults" ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages)). Treat them "as examples to evaluate, not universal rules" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).
- **Evaluation:** the docs contain no dedicated evaluation guide. The scattered guidance:
  - Test level wordings on "separate inputs" with known expected levels ([Score](https://docs.typesafe.ai/primitives/score)).
  - Keep a held-out set, and "keep a final test set untouched by both feature discovery and model selection" ([Autoresearch](https://docs.typesafe.ai/cookbooks/autoresearch_feature_discovery)).
  - The cookbooks build small labelled sets that deliberately include hard negatives. For example, 173 of 488 requests were "written to punish guessing" ([Skill suggestion](https://docs.typesafe.ai/cookbooks/skill_suggestion)).
  - "Test representative cases and the resulting application behavior… Separate missing evidence, model errors, code errors, and service failures" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)).
  - Thresholds are cheap to re-tune because routing reads stored answers: "re-routing every passage costs no API calls" ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages)).
  - TypeSafe's own [WorkflowEvals](https://github.com/typesafe-ai/WorkflowEvals) repo scores agreement with reference labels from other models and a consensus, which is one way to bootstrap labels. The building guide also suggests using "an ensemble of expensive reasoning models to generate" labels ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)).
- **Determinism:** this is answered in part (it was open question 2 in jev.md). Identical requests are **not guaranteed** to return identical results. With a byte-identical document, most answers had "std dev exactly 0.0," but two Nouls "carry a little run-to-run sampling noise" ([Parallel questions](https://docs.typesafe.ai/cookbooks/parallel_questions)). In 15 repeats, jev-1.13 Choice labels flipped on 2 of 8 questions, and one Noul ranged 0.43–0.53 across a 0.5 threshold. Those runs added a changing `uid` field, so they "cannot separate sensitivity to the irrelevant field from variation that would occur on identical requests" ([Self-consistency: choices](https://docs.typesafe.ai/cookbooks/consistency_choice_cookbook), [Self-consistency: nouls](https://docs.typesafe.ai/cookbooks/consistency_noul_cookbook)). No seed or temperature control is documented.
- **Versioning:** "An alias moves when a new release ships, so the answers behind it can change without a change on your side… If you have tuned confidence thresholds against a specific version, pin that version's ID instead of the alias." Log the returned `model` ([Models](https://docs.typesafe.ai/models)). The cookbooks record both the requested and the returned model "because an alias can resolve to a different version later" ([Self-consistency: nouls](https://docs.typesafe.ai/cookbooks/consistency_noul_cookbook)). The [jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13) list is version-specific, and "many of these will be fixed in later versions."

## 7. `instructions` best practices; reusing questions

- Write the complete question in `instructions`. IDs are "not sent to the model" ([primitives](https://docs.typesafe.ai/primitives)).
- Phrase it "as a clear, specific question, or as a statement for the model to judge" ([primitives](https://docs.typesafe.ai/primitives)). "Try both phrasings with your own data" ([Noul](https://docs.typesafe.ai/primitives/noul)).
- Phrase it so that a high value means yes, and avoid inverted questions like "Is the message free of…" ([Noul](https://docs.typesafe.ai/primitives/noul)). Avoid double negatives and indirection, and "identify the relevant parts of state by name" ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- Keep questions short. Use an object when the question needs context, examples, or data that comes from code ("put it in its own field instead of splicing it into a string template"). Structure also helps "make questions distinct" when several are similar ([How to build](https://docs.typesafe.ai/concepts/how-to-build-with-system-one)). Field names like `question`, `focus`, `inspect`, `compare`, and `note` appear in examples. None are reserved ([Choice](https://docs.typesafe.ai/primitives/choice), [Advanced: structure](https://docs.typesafe.ai/primitives/advanced)).
- Instructions and criteria must not contradict each other ([jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13)).
- "Put the constants (questions and thresholds) in a single place so they're easy to review" ([Agent skill](https://docs.typesafe.ai/agent-skill)).
- **Reuse and caching:** no server-side or prompt caching is documented. Pricing is per input token on every request ([Models](https://docs.typesafe.ai/models)). The documented reuse pattern is to define questions once as constants and vary only the state ([RAG passages](https://docs.typesafe.ai/cookbooks/classifying_rag_passages): "Use the same four questions for every query. Only the state changes"), and to store answers so weights and thresholds can change without new inference: "Changing a weight or display filter need not rerun inference when evidence and question meanings are unchanged" ([SKILL.md](https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md)). The cookbooks cache API responses on the client (`json_cache.json`) only so the published numbers can be replayed.
- **Inference:** storing per-(post, interest, model version) answers would let us re-threshold, or re-combine after a user edits weights, at no cost. A user editing an interest's *text* changes the question, so it needs new inference.

## Open questions

1. What is the maximum number of questions per request, other than the 64k token budget? Still undocumented.
2. Does per-interest Noul relevance hold up when the interest text is written by end users? No source covers this, so it needs our own labelled set.
3. How much does a self-promoting caption move a relevance Noul, and does a gating "self-promotion / instruction" Noul catch it on jev-1.13? The documented injection results are from jev-1.12, on RAG passages and chat prompts.
4. Is run-to-run variance on byte-identical requests large enough to flip pass/fail near our thresholds? The docs show small but non-zero noise, and the uid-perturbed runs mix in input sensitivity.
5. Should the state include `uncertainties`, and does it change answers? Not covered.
6. The blog post "AI: too good to be true, too bad to be useful" renders client-side and could not be read. The other blog posts ([antibenchmaxxing](https://typesafe.ai/blog/antibenchmaxxing), [bitterest-lesson](https://typesafe.ai/blog/bitterest-lesson), [launch](https://typesafe.ai/blog/introducing-system-one-models-and-jev)) contain no practice guidance beyond what jev.md already cites.
7. Two docs pages give different figures for the same batching experiment (11.5x/9.6x vs 12.2x/10.0x). It does not affect the design.

## Sources

- https://docs.typesafe.ai/llms.txt
- https://docs.typesafe.ai/concepts/state
- https://docs.typesafe.ai/concepts/how-to-build-with-system-one
- https://docs.typesafe.ai/introduction/machine-learning-primer
- https://docs.typesafe.ai/primitives
- https://docs.typesafe.ai/primitives/choice
- https://docs.typesafe.ai/primitives/score
- https://docs.typesafe.ai/primitives/noul
- https://docs.typesafe.ai/primitives/advanced
- https://docs.typesafe.ai/confidence
- https://docs.typesafe.ai/patterns/composite-scoring
- https://docs.typesafe.ai/patterns/confidence-routing
- https://docs.typesafe.ai/patterns/fan-out
- https://docs.typesafe.ai/patterns/intent-routing
- https://docs.typesafe.ai/cookbooks/parallel_questions
- https://docs.typesafe.ai/cookbooks/consistency_noul_cookbook
- https://docs.typesafe.ai/cookbooks/consistency_choice_cookbook
- https://docs.typesafe.ai/cookbooks/classifying_rag_passages
- https://docs.typesafe.ai/cookbooks/llm_guardrails
- https://docs.typesafe.ai/cookbooks/rerank_typesafe
- https://docs.typesafe.ai/cookbooks/semantic_find
- https://docs.typesafe.ai/cookbooks/skill_suggestion
- https://docs.typesafe.ai/cookbooks/entity_alignment
- https://docs.typesafe.ai/cookbooks/classification_using_confidence
- https://docs.typesafe.ai/cookbooks/autoresearch_feature_discovery
- https://docs.typesafe.ai/cookbooks/citation_check
- https://docs.typesafe.ai/models
- https://docs.typesafe.ai/model-jaggedness/jev-1.13
- https://docs.typesafe.ai/api
- https://docs.typesafe.ai/agent-skill
- https://github.com/typesafe-ai/skills/blob/main/skills/typesafe-ai/SKILL.md
- https://github.com/typesafe-ai/WorkflowEvals (README)
- https://typesafe.ai/blog/introducing-system-one-models-and-jev
- https://typesafe.ai/blog/antibenchmaxxing
- https://typesafe.ai/blog/bitterest-lesson
