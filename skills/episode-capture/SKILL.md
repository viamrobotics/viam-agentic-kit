---
name: episode-capture
description: Guidance for creating training data for policy training
---

## Helper scripts

In the cosmos-tutorial repo, there are `batch_episodes.py` and `oracle_episode.py` scripts which are templates for designing and collecting episodes that will lead to good training data.

when calling these scripts, you'll need credentials, most likely from /etc/viam.json (automatic) or ./viam.json (pass cli arg). the part ID in viam.json should be the same as what you're interacting with through the MCP; if it's the first time you're touching a robot, look up its name and confirm it's the one the user expects.

machine address may not be available directly from the MCP yet. if not, you can synthesize it from machine metadata as `{part.dns_name}.{part.location_id}.viam.cloud`.

for connection trouble: if you get a PermissionDenied error in `dial_ffi`, or other OS-level error, try `sandbox_permissions: "require_escalated"` in the harness.

## Manual run-throughs

when designing the motions for a robot task, you can do manual runthroughs using mcp commands. your call when to do this in python vs issuing the commands directly, but 'direct' will better for demoing short sequences while the user watches a camera.

when you lift or manipulate something, check the simulator state to make sure that it really was lifted.

when you issue mcp grip commands, use `disable_sessions=True` to prevent it from relaxing when the connection closes. only do this for grip commands.

don't call `stop_simulation` to save money between run-throughs or capture runs, even though the sim is billed per minute; only stop it when the user explicitly asks.

## Capturing

The user should give you a `batch_episodes.py` script that you can modify if needed. Before running it, check the robot's config and set CLI args accordingly (number of cameras, component names, etc).

This can be told to upload a dataset at the end. Use the viam python sdk to download the dataset and inspect the episodes.

For the initial smoke testing runs, create a contact sheet (composite image with a grid of episode end states from different cameras for the user to look at).

If a run takes longer than expected it may be hung; you can check this by looking at cameras in the scene or querying positions. It's useful to know if there's a hang even before a timeout is triggered, so for an above average run. timeouts should be in sim time where feasible; if you use wall clock timeouts make them 2x slower.

## Episode count schedule

Start low (5) so the user can investigate them and so you can debug success rate. The script allows you to add more episodes to a dataset if things work. Use separate datasets when logic changes, though.

You can download datasets through the SDK to inspect video and parquet.

A good number for a starter training run is 50. (A compromise between quality and generation / training time). Go to 100 if the resultant policy needs to be smoother.

## success state

The last frame of an episode should be a success state. E.g. if the task is 'pick up the block', don't then drop the block before ending the episode; end with it picked up.

## multiple tasks

it's possible to train a reasonable policy with a single task, and for simplicity, that's our beginner flow. if after that the user wants to increase model quality, multiple tasks will improve it (even for the original task prompt).
