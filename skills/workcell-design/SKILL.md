---
name: workcell-design
description: Guidance for designing a simulated workcell and tasks. This is intended for designing simulation workcells; don't use this guidance for production robots.
---

## Motivation

users design simulators to test hardware or software in a new layout, to iterate on a deterministic control policy, and to capture episodes for training VLA and WAM policies.

## Select a type of robot and task

some users will be here to exercise the technology -- in that case, their motivation might be easy vs hard (single arm vs bimanual, rigid vs soft vs liquid). others have a production workcell they're trying to replicate. others may be interested in a particular task and ambivalent about the specific hardware.

the easiest starting point will be picking an object with a single arm.

consider constraints like the workspace of the arm, whether the gripper can get around a target.

## scene design

you can source assets from various places. isaac has built-in arms. box primitives you can generate on your own.

tools like blender or meshy are sometimes available if you need. you can embed USDs in the simulation for representing complex objects.

## working with fragments

some configs are based on fragments (shared json templates). unless the user explicitly wants to edit a fragment, don't put edits in them, put them in the machine. the `update_machine_config_item` call will refuse to allow edits to components from fragments (check its docs though, it may have been patched). the alternative is to call UpdateRobotPart (but please read the config immediately before to avoid race conditions). if you do that, you'll need separate robot credentials. look in /etc/viam.json and ./viam.json but *make sure* it's for the same robot (using the part ID in the json config). instruct the user if you can't find the creds that they can get them from 'machine cloud credentials' on the 'ONLINE' box on the robot page and make it clear this is to support a workaround for a pending MCP feature.

## task design

for VLA training you'll design an oracle task, i.e. a task that knows the exact positions of targets. for other kinds of testing you might have a vision model in the loop.

work with the user; if they want to leave it to you, delegate it to a frontier model with good spatial reasoning (astra / fable class). you'll need to watch out for motion envelopes, overlapping, compatible sizes.

the user likely has an `oracle_episode.py` script for you to customize; try to keep to the patterns in there:
- start pose randomization
- joint nudges
- random initial placement of objects in the scene (including the object to be picked and distractor objects)

Those will matter for training quality.

as you make changes, ask the user if they want to use git to make these edits on a branch so that known-good logic is kept in history.

for prototyping, offer to manually execute the motion through the mcp before changing any code.

## success detection

it's important to be able to detect success, especially if you want to collect many episodes. delegate to a frontier model for this. typically a success probe will look at the location of an object that is to be moved, but this can be time sensitive (for example probing before a drop, or after the episode ends when a gripper correctly relaxes).

## Considerations for cosmos 3 policies

bear in mind that changing the DOFs changes the action space, i.e. involves new work on the inference side.

the task should be visually obvious given the cameras in the scene, i.e. not obscured or at a strange angle. use a frontier VLM (astra / fable class) to verify this.

consider randomness. randomly placed targets, start episodes at random times and with random offsets, with nudges. (the starter scripts, if present, will guide you here; they contain tested assumptions and should usually take precedence over the assumptions in this markdown)
