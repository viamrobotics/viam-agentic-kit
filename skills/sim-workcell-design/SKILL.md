---
name: sim-workcell-design
description: guidance for designing a simulated workcell and tasks
---

## Motivation

users design simulators to test hardware or software in a new layout, to iterate on a deterministic control policy, and to capture episodes for training VLA and WAM policies.

## Select a type of robot and task

some users will be here to exercise the technology -- in that case, their motivation might be easy vs hard (single arm vs bimanual, rigid vs soft vs liquid). others have a production workcell they're trying to replicate. others may be interested in a particular task and ambivalent about the specific hardware.

the easiest starting point will be picking an object with a single arm.

consider constraints like the workspace of the arm, whether the gripper can get around a target.

## scene design

you can source assets from various places. isaac has built-in arms. primitives you can generate on your own.

tools like blender or meshy are sometimes available if you need. you can embed USDs in the simulation for representing complex objects.

## task design

for VLA training you'll design an oracle task, i.e. a task that knows the exact positions of targets. for other kinds of testing you might have a vision model in the loop.

work with the user; if they want to leave it to you, delegate it to a frontier model with good spatial reasoning (astra / fable class). you'll need to watch out for motion envelopes, overlapping, compatible sizes.

## success detection

it's important to be able to detect success, especially if you want to collect many episodes, and the same check scores the trained policy later (see the `policy-eval` skill). write the check before the oracle, as a pure function of simulator state (object poses, gripper holding state), never from what the script commanded. a success probe typically looks at the location of the object to be moved, and it is time sensitive: require the final state to hold for a few samples, and for place tasks require the gripper to have released (probing before the drop, or while the block is still in the jaws, gives false successes). give it stages (reached, grasped, lifted, placed) so failed episodes and rollouts say how far they got. cosmos-tutorial's `episodes/task_check.py` is the pattern; audit a new check with `check_task.py` (fresh scene is not success, the oracle is success, a wrong ending is not) before generating a dataset. delegate the spatial parts to a frontier model if needed, but the audit is not optional.

## Considerations for VLAs

bear in mind that changing the DOFs changes the action space, i.e. involves new work on the inference side.

the task should be visually obvious given the cameras in the scene, i.e. not obscured or at a strange angle. use a frontier VLM (astra / fable class) to verify this.

consider randomness. randomly placed targets, start episodes at random times and with random offsets.

VLA training will go better with several tasks (2-5 rather than 1), and various aspects of diversity.
