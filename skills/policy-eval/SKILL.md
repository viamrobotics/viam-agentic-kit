---
name: policy-eval
description: Evaluate a trained arm policy on the simulated workcell it was trained for, using the cosmos-tutorial scripts. Covers the success checker contract shared with episode capture, choosing seeds and trial counts, running eval_policy.py, reading k/n with a confidence interval and stage-reached counts, triaging failures (with a VLM only as a second opinion), and reporting. Use after a model is deployed with the arm-policy module, or when the user asks how well their policy works, whether it improved, or why rollouts fail.
compatibility: Needs a checkout of https://github.com/viam-labs/cosmos-tutorial, uv, machine credentials, and a machine running the simulator with the arm-policy module configured. The Viam MCP server is optional (useful for reading logs and camera images).
---

# Policy evaluation

The eval scores a trained policy with the **same success checker that labelled its training episodes**. That is the whole point: the number is comparable to the success ratio the user saw during episode capture, and the data filter and the evaluator cannot drift apart.

## The checker contract

`episodes/task_check.py` in cosmos-tutorial defines success as a pure function of sampled simulator state (object poses, gripper holding state, gripper position). It never looks at what the controller commanded.

- Ordered stages. Pick: `reached -> grasped -> lifted`. Pick and place adds `over_pad -> placed`. The last stage is the success condition.
- Hold window: the final stage must hold for 3 consecutive samples, so a bounce or a slipping grasp does not score.
- Two numbers: `success` (held at the end of the run) and `success_once` (held at some point). They differ when the policy drops the object after lifting.
- `oracle_episode.py` labels each captured episode with `PickTask.check`; `eval_policy.py` scores each rollout with it.

If the user designed a different task, the checker has to change with it, and the order matters: **write the check first, then the oracle**, so the oracle's end state is what the check accepts. Keep the check sparse (final-state predicates with explicit tolerances, a release condition for place tasks, a hold window). Do not write a dense reward; it gets gamed.

Before a dataset is generated with a changed checker, audit it on the live sim:

```sh
cd cosmos-tutorial/episodes
make test                                      # offline predicate tests
uv run check_task.py <machine address> --seed 3000 [--place-prop place_pad]
```

`check_task.py` asserts that the fresh scene is not success, a noise-free oracle episode is success, and a wrong ending (drop after lift, or the block 6 cm off the pad) is not. Add equivalent cases for a new task. An LLM-written predicate that has not passed these is not evidence of anything.

## Seeds and trial counts

- Use seeds that `batch_episodes.py` did not use. The scene sampler is seeded, so the same seed gives the same block pose for every model; compare models on the identical seed set.
- 10 seeds is a smoke test. A Wilson 95% interval at n=10 spans about 45 points, so it only separates policies that differ by 40 points or more. Treat 7/10 or better as "works".
- To compare two models or checkpoints, budget 25 to 50 seeds each (about 30-point gaps resolve at 50). A rollout costs several minutes on an L4, so plan the time.
- Cap the run at a multiple of the oracle's duration: the default `--max-chunks 20` is about 1.5x the chunks a working pick needs. A timeout is a failure, not a missing data point.

## Run it

```sh
cd cosmos-tutorial/episodes
uv run eval_policy.py <machine address> --seeds 3000-3009 --policy <arm-policy resource name>
uv run eval_policy.py <machine address> --seeds 3000-3009 --place-prop place_pad   # pick and place
```

Credentials come from `VIAM_API_KEY` / `VIAM_API_KEY_ID` or `/etc/viam.json`, as for the other scripts. The policy server must already be started on the module (the deploy step does this). Useful flags: `--prompt` to override the module's configured prompt, `--max-chunks`, `--run-all-chunks` to keep executing after success is held, `--rerun` to ignore cached seeds, `--output-dir`.

Output: `runs/eval/<label>/seed-<seed>.json` per rollout (scene, verdict, stage times, chunk count, sampled state history) and `summary.json`, plus a printed table:

```
success 7/10 (95% CI 0.40-0.89); success at any point 8/10
stage reached: reached 10, grasped 8, lifted 7
median time to stage (s): reached 14.2, grasped 31.0, lifted 38.5
| seed | success | highest stage | chunks | duration s | state | error |
```

Watch the Control tab's scene camera in Live mode while it runs if the user wants to see it.

## Read the results

Report raw counts and the interval, never a bare percentage, and always with the seed set, the model version, and the `execute` arguments (`summary.json` records them).

Triage by the highest stage failed runs reached:

| Highest stage | Likely cause | Check first |
|---|---|---|
| none (never `reached`) | observation mismatch: camera poses, image size, scene differs from training; or the policy hovers | compare a rollout camera frame with frame 0 of a training episode; module logs for rejected chunks |
| `reached`, not `grasped` | grasp placement off by more than the jaw margin, descends beside the block, closes early | wrist camera at the close; the training set's final-descent noise |
| `grasped`, not `lifted` | slip, closed on the edge, gripper force | `firm_grasp` in the seed JSON history; gripper settle |
| `lifted`, not `placed` | carries to the wrong spot or releases too high | pad tolerance (8 cm pad, 1 cm rest tolerance); release height in the oracle |
| `success_once` but not `success` | drops after the hold | the "hold still" tail of the training episodes (33 frames) |
| run `failed` with an error | executor rejected chunks (velocity limits, workspace floor) | module logs; arm-policy safety profile |

Two models on the same seeds: compare `success`, then stage counts, then median time to `lifted`. A difference of 1 or 2 out of 10 means nothing; rerun with more seeds before drawing a conclusion.

## Looking at rollouts with a vision model

The checker is the ground truth. A vision-language model is a second opinion for **why** a run failed, not for whether it did: zero-shot frontier models reach roughly 0.75 to 0.80 balanced accuracy as success judges and err toward calling failures successes.

When you do look at frames:

- Use the fixed scene camera, not the wrist camera, and 8 to 16 evenly spaced frames with the first and last labelled. A single final frame is close to useless.
- Ask staged questions ("did the gripper reach the block, close on it, lift it, carry it to the pad?") and a failure category (missed grasp, dropped, wrong object, collision, no motion, timeout). Allow "unsure" and default to failure under ambiguity.
- Record the model's answer next to the checker's verdict. If they disagree on whether the task succeeded, the checker wins; note the disagreement, it usually means an occluded view.

Eval rollouts are not recorded as videos by default. Take frames from the camera components during the run (the Viam MCP server or the Control tab), or ask for `--capture` support if videos are needed routinely.

## Do not

- Compare models on different seed sets, or a 10-seed run against a 50-seed run.
- Change the checker between data generation and eval without re-running `check_task.py` and noting it in the report.
- Report a VLM's success rate as the policy's success rate.
- Stop a batch early because the first few seeds look good; the seed set is the unit.
