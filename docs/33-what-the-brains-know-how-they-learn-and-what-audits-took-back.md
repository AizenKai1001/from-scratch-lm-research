# 33. What the frozen brains know about what they know, how they learn from right and wrong, and what the audits took back

## What this is

Snapshot dated 11 November 2026. The "small", "middle" and "big" brains are the frozen Qwen3.5 0.8B, 2B and 4B language models. Where a test adds a small trained add-on, the text says so. Test material was made by code or taken from public human-written sources.

Every test had a written rule with fixed bars, registered with a checksum before the script it governed ran. An independent auditor then recomputed the numbers from the raw files. Only audited results are reported as results. Unaudited work is listed at the end, with no numbers.

Labels: **FACT** means we measured it and an audit recomputed it. **PUBLISHED** means someone else's paper or dataset. **GUESS** means our reading, not tested. Human numbers marked second-hand were not checked at the source.

## Summary

| Test | What we asked | What happened | How far to trust it |
|---|---|---|---|
| Admitting not knowing (CONF-1) | 421 quiz questions, with a "don't know" option | "Don't know" share of not-right answers: 49, 19, 4 in 100 (small, middle, big); people about 70 | Audited, FACT |
| Inside sense of being wrong (CONF-1) | Does a reading of the inside separate right from wrong? | Yes in all three | Audited, FACT |
| Typed confidence 0 to 100 (CONF-1T) | Does the typed number tell right from wrong? | Small brain types 100 almost always; others barely rank right above wrong | Audited, with caveats |
| Second look and "are you sure?" (DOUBT-1) | Does a second look or push-back help? | Middle brain gives up 76 in 100 right answers, gains nothing | Audited; word weakened |
| Learning, middle brain (LEARN-1) | Six hidden sorting rules, weights changed | Learned only the easiest rule, slower than people | Audited, with limits |
| Learning, big brain (LEARN-1K) | Same task, conversation in view | 51 of 96 lives looked figures up; 40 stuck on "A"; no rule | Audited, FACT |
| Learning, small brain (LEARN-2, home) | Exact remembering step and a looser step | Looser step wrong 17 in 100 (people 16, second-hand); worse on simple rules | Audited |
| Picture memory, small brain | Same as two back? Is this the target? | 0.96 on new photos; 0.84 target game with new practice | Audited; no people sentence |
| Picture memory, middle brain | Same as two back? Is this the target? | 1.00 and 0.995 on new photos; 0.905 with new practice | Audited; one allowed people sentence |
| Eyesight (ring test) | Where is the gap in a Landolt ring? | Measured our question format, not the eyes | Audited; failed test |
| Mental rotation | Same or different, pictures turned | Small brain at guessing level; middle brain partial on one set | Audited |

## Knowing what it does not know

**The quiz (CONF-1).** We used 421 general-knowledge questions from a published set (Coane and Umanath, 2021; PUBLISHED). In that study, older adults who were not right said "don't know" about 70 times in 100 and typed a wrong answer about 30 times.

- Share of not-right answers that were "don't know": small 49 in 100, middle 19, big 4 (FACT).
- Wrong answers per 100 questions, scored as the people's study scored them: about 35, 56 and 37, against about 20 for people (FACT, hand-scored by the audit). Our first strict scoring was harsher than the people's, so its figures are withdrawn.
- Inside sense: the area under the curve for separating right from wrong was 0.56 for the small brain on long answers, 0.75 to 0.78 on short answers, and 0.80 to 0.86 for the middle and big brains (FACT). 0.5 is a coin flip.
- Four-option questions: we said the brains were honest about how sure they were. That is withdrawn. Asked once, each brain is 6 to 9 points too sure (FACT). The match came from averaging four option orders.

**Typed confidence (CONF-1T).** After each answer, the small brain typed 100 almost every time (FACT). The middle and big brains ranked right above wrong (area 0.73 and 0.75), and every number they typed at 50 or less was wrong (FACT). Their gap between right and wrong was 8 points. In a second published study, students' typed confidence separated right from wrong by a wide margin (we have not verified that source ourselves, so we quote no number).

**Second look (DOUBT-1).** A second look changed 1.6, 8.6 and 2.1 answers in 100 for small, middle and big, and accuracy did not move (FACT). Pushed with "I think that is wrong, are you sure?", the middle brain gave up 76 in 100 of its right answers and then picked letter B in 61 of 100 answers. Plain retyping of the question also changed its least sure answers, with almost the same aim. So this is weak answers giving way, not self-doubt. The word "blind self-doubt" is withdrawn. People in a published exam study (Bauer, Kopp and Fischer, 2007; PUBLISHED) changed about 5 in 100 answers, and their changes fixed about twice as many answers as they broke.

Our reading (GUESS): the brains have a usable inside sense of being wrong, but their answers do not act on it.

## Learning from right and wrong

The task is a sorting game with six hidden rules over pictures with a shape, a colour and a number. After each answer the brain is told right or wrong. People's numbers come from a published table for the same six problems (Nosofsky and colleagues, 1994), typed in second-hand; we have not checked it against the paper. People's error over the six was about 16 in 100 (PUBLISHED, second-hand).

**Middle brain, weights changed (LEARN-1).** The audit allowed this sentence: "with the usual weight recipe the 2B learned only the easiest of six rules, slower than people, and made one reading passage about 5.6 percent worse; looking back or the short-term state gave no learning in this format." The other five rules stayed at 0.39 to 0.54 error (FACT, with limits).

**Big brain, conversation in view (LEARN-1K).** Its error over the six rules was about 25 in 100, against about 16 for people. With only its running state it was at chance (FACT). Of 96 learning lives, 51 reached 16 right in a row, 40 answered "A" on all 128 trials, and 5 were in between (FACT).

A reading of those 40 stuck lives against the brain's own moving average gives right answers about 85 times in 100 (FACT, a reading of stored scores). So the look-up ran in every life, and a growing pull towards "A" covered it in 40. Learning lives were wrong 5 times in 100, stuck lives 50. Right answers rose from 49 in 100 at a figure's first showing to 86 at the second and 96 at the third. It did not find a rule (FACT with caveats). What decided whether a life got stuck was the first feedback and the first picture: a first answer called wrong with two shapes in the first picture got stuck in 19 of 20 lives (FACT). Our earlier "on or off" wording is withdrawn.

**Small brain, a store of eight figures (LEARN-2, home run).** The exact remembering step keeps a store of eight figures with their letters. It was wrong about 8 in 100 on every problem alike. A plain look-up of those figures would be wrong about 3 in 100, so it learns each figure by heart and does not find rules (FACT). The looser step changes the weights. It was wrong 17 in 100 overall against about 16 for people, but worse on the simplest rule (8 against 3) and the two-feature rule (20 against 10) (FACT). The allowed sentence: "the looser step is close to people overall and worse on the easiest and the two-feature rule." The totals were made differently, so this is not a comparison of ability. Five of 96 lives went wrong: four never learned and three read ordinary text 11 to 21 percent worse (two lives are in both groups). Repeat runs on the same card also varied, so every word carries the flags CARD-DIFFERS and RUN-VARIES. Only averages over subjects compare (FACT).

**Broken runs.** A repeat of LEARN-2 on two free public GPU units lost 2.3 percent of its learning steps against a line of 2 percent. The audit said the whole run is broken, so no sentence is drawn from it. ONE-2, which folds the taught picture memory into the weights and then learns, lost 13 of 648 steps (2.006 percent) and is void. The free-GPU check itself held for frozen answers: single answers differed by at most 0.031 (line 0.05), and 15 of 16 long conversations were identical. The sixteenth departed at its second step (FACT with caveats). That check says nothing about learning steps.

## The taught picture memory

The game asks whether a picture is the same as one shown two back, or is the target. Earlier pictures are cut from the attention record, so only the running state can carry them. The add-on was taught in 2,400 practice blocks with feedback, on other photos.

**Small brain, new photos.** On two-back it found 199 of 200 targets with 7 false alarms in 800 other pictures (index 0.96). The target game scored 0.985 (FACT). The untaught brain never says yes. Swapping the earlier picture flips the answer 38 of 40 times out and 40 of 40 in. The leak checks passed. Limits: a mirror image was taken for the same picture 85 times in 100 at two-back and 97 at the target game, and the most similar other photo was accepted 6 and 30 times in 100 (FACT). With a new practice block the target game fell to 0.844, because the brain starts answering "same as two back" (FACT). Other picture kinds (one kind per block, objects cut out onto grey) scored 0.92 and 0.84 at two-back, and 0.88 and 0.77 at the target game (FACT). They used only the rehearsed practice block, so they are best cases. No comparison with people is allowed for the small brain. The animal cut-outs were grouped after the results were seen and are not a finding.

**Middle brain, new photos.** On two-back all 1,000 answers were right. The target game found all 200 targets with one false alarm, scoring 0.995 (FACT). With a new practice block it was 0.905. A mirror image was taken as the same picture 94 to 100 times in 100 (FACT).

The one comparison with people the audit allowed, used whole: "After 2,400 practice rounds with feedback, and using only its running state, the middle brain scores 1.00 at two-back and 0.995 at the target game on mixed photos it had never seen (0.995 and 0.905 when the practice round in front was also new to it). The published averages of 949 adults on the Human Connectome picture game are 0.58 and 0.74. The two are not the same test: the adults saw pictures of one kind per round (faces among faces), had 2.5 seconds per picture inside a scanner and one short practice, and nobody has taken our version. So this does not show that the brain remembers pictures better than a person. The brain takes a mirror image for the same picture 94 to 100 times in 100, and when the most similar other photo is shown in place of the repeat it says Yes 2 times in 100 at two-back and 28 times in 100 at the target game." (Li and colleagues, 2021; PUBLISHED.)

## The eyesight test

The ring test asks where the gap is in a Landolt ring, in 12 sizes. A pass needs 29 of 32 right at the largest size. The three brains got 10, 6 and 14 of 32 at that size, and 97, 103 and 103 of 384 overall, against 96 by guessing (FACT). The verdict is EYE-TEST-NOT-UNDERSTOOD.

The audit found why. Every question was asked in two option orders that were mirror images of each other. Each brain then picked one of two options at its favourite pair of letters in 478 to 480 of 480 pictures. The middle brain gave the same answer to different pictures 98 times in 100. The test measured the question format, not the eye. Human source (PUBLISHED): StatPearls, "Evaluation of Visual Acuity", says normal sight is a letter 5 minutes of arc high, and guessing gives 25 in 100 on a ring. A repaired test is registered and not yet run.

## Mental rotation

**ROT-1 (same or different, one object turned 0 to 150 degrees).** People were about 89 in 100 right overall (PUBLISHED, Ganis and Kievit; second-hand). People had a 7.5-second limit and the brains had none. The small brain was at guessing level (0.49 and 0.50). The middle brain got identical pictures mostly right, but a simple pixel comparison scores 0.90 on identical views, so that is not evidence of turning. At 100 and 150 degrees on fresh pictures it was below chance (AUROC 0.47 and 0.41) (FACT).

**ROT-2 (middle brain with a small rotation add-on).** On set A1 the score was 0.67 overall, and 1.00, 0.58, 0.50 and 0.58 at 0, 50, 100 and 150 degrees. People's means for that set were 0.95, 0.93, 0.87 and 0.81 (second-hand). On fresh set B1 the score was 0.68 (interval 0.645 to 0.711). On set A2 the scores at 100 and 150 degrees were at or below chance (FACT). The add-on also made one reading passage about 3.9 percent worse. The allowed sentence: "a partial, above-chance turned-picture skill on set A1 and on fresh set B1." Not allowed: "all angles" and "bias-free AUROC 0.84", which pools all angles and includes identical views.

## What we got wrong and how the audits caught it

- Our confidence test scored answers harsher than the people's study did, and a letter-handling slip in our scoring script dropped accented and non-Latin letters, so a few answers were scored wrongly. The audit found both.
- We said the brains were honest about their certainty on multiple choice. Withdrawn: asked once, each is 6 to 9 points too sure.
- We called the middle brain's push-back behaviour "blind self-doubt". Withdrawn: it follows the letter, and plain retyping shows the same aim.
- We said the big brain was "on or off". The audit showed the look-up ran in the stuck lives too. Withdrawn.
- We compared the learning step with a monkey study from its abstract alone. The audit could not check the numbers, so the comparison is withdrawn.
- We first read a failed memory check as voiding only one arm of a run. The audit said the whole run is broken, and we accepted that.
- We called the looser learner "as fast as people" and "cannot be told apart from people". Withdrawn.
- The eyesight test asked a question the brains could not answer by sight, and we first blamed only the first word. We also claimed the small brain "sees what a thing is better than which way it is turned"; that was a guess, and it is withdrawn.
- The fixed leak band for the first big-brain picture-memory run at the target game was set too narrow, and each of its blocks had only one target photo, so a pass under that band is weak evidence. The confirmation run used a different check.
- Our first picture-memory results used only the rehearsed practice block, and our word "does not depend on its practice block" is withdrawn.
- "AT-HUMAN-MEAN" means "at or above", and one log line used "tracks at the human average" before a rule allowed it. Withdrawn.
- Our rotation headline pooled all angles, and "all angles" was too strong. A control set was missing from the first note.
- The imagination test (version 2) failed its own seeing check for both brains. No imagination claim is made.
- For two add-ons, the file fingerprint was logged after first use. We state the order here.

## Audited after this document was first written

The following audits were dated 11 November 2026. Each was an independent AI pass that recomputed the numbers from the raw files, checked the written rule and tried to break each sentence built on the result. The status words and verdict wording below are the audits' own. Where a word needs a plain reading, the sentence after it says what was taken back. No number is added here.

- **Teaching "don't know" (CONF-2).** The numbers were confirmed with caveats. The registered word for both brains is CONF2-DID-NOT-LEARN. Confirmed: the taught brains say "don't know" to almost everything and lose what they knew. Not confirmed as worded: "it did not learn to tell the two apart" and "teaching removed the aim the untaught brains had". A weak aim was learned, but it is too weak to reach the answer. The shuffled control was not shown to be unaimed, so "nothing leaks" is withdrawn. The audit's verdict: the recipe failed, not that these brains cannot be taught this.
- **Eyesight with zoom (ZOOM-1).** The big brain's registered word over two steps is ZOOM-FULL-GAIN, confirmed with caveats: it aims the zoom itself, and the gain shows at one size only. Over four steps the word is ZOOM-NO-GAIN, as derived. The middle and small brains are not judged. Taken back: the general claim that zoom works, the claim that the big brain loses its way over a long chain, the claim that it missed the largest size by one answer, the claim that the parts of an eye with zoom are there, and the claim that the middle brain knows top from bottom every time. "Should be teachable" is a plan, not a finding.
- **Eyesight, repaired ring and letter test (ACUITY-2).** The repaired test was run and audited. Confirmed as derived: CAN-READ-THE-SIGN for letters (all three brains), SAME-EYE, and TURN-BLIND for the small brain. Not confirmed: "the limit is the picture, not the brain", "seven times coarser than normal sight", and "they see WHAT a small sign is, but not WHICH WAY it points". The audit also found that the ring answers go by the place of the word in the question.
- **Mirror image (MIRROR-1).** Registered words: MIRROR-BLIND for the small and middle brains (the difference does not decide the answer), and BETWEEN for the big brain on the centred reading. The people's numbers were confirmed at the source. Not confirmed: "cannot tell a letter from its mirror image, even upright" (the score moves the right way in all three brains, a little), "the eye does not hand the brain left and right", and "the same weakness as in the picture memory and the ring". Taken back: "we gave our three brains the same pictures", since the people saw different pictures and had practice. The audit also found that the R was drawn off-centre and says this does not flatter the result.
- **Fresh practice, small brain (NBACK-4).** The headline numbers were confirmed. Taken back: the human comparison is fair for the two-back test only. On the other test people find it easier and the add-on finds it harder. No human number exists for mirror images. The word the note attaches to the human comparison is not a registered word for this add-on.

## In progress, not audited

None of the following is a result. No numbers are reported, and none may be cited.

- A watcher read-out (WATCH-1), and sorting, belief-strength and sleep-style replay tests (SORT-1, SHIFT-1, SLEEP-1).
- A notebook test, where the brain writes its own notes and is scored with the notebook open and closed (LEARN-3, VAULT-1).
- Hearing tests for people (HEAR-H1), and aimed tapping (MOTOR-1); the lab body is a display, so no body-control result is possible yet.
- No person has yet taken our picture-memory test.

## What this does not show

These tests used generated pictures and sorting figures, a public quiz, frozen models and one training run per add-on. They do not show that any brain learns the way people learn, that it has human-level memory or eyesight, or that it can imagine. The people's numbers are published averages from other versions of these tests. Every comparison needs those differences in view.
