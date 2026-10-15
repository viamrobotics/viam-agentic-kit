---
name: episode-capture
description: Guidance for creating training data for policy training
---

## Helper scripts

In the cosmos-tutorial repo, there are `batch_episodes.py` and `oracle_episode.py` scripts which are templates for designing and collecting episodes that will lead to good training data.

when calling these scripts, you'll need credentials, most likely from /etc/viam.json (automatic) or ./viam.json (pass cli arg). the part ID in viam.json should be the same as what you're interacting with through the MCP; if it's the first time you're touching a robot, look up its name and confirm it's the one the user expects.

## Manual run-throughs

when designing the motions for a robot task, you can do manual runthroughs using mcp commands. your call when to do this in python vs issuing the commands directly, but 'direct' will better for demoing short sequences while the user watches a camera.

when you lift or manipulate something, check the simulator state to make sure that it really was lifted.

when you issue mcp grip commands, use `disable_sessions=True` to prevent it from relaxing when the connection closes. only do this for grip commands.

## Capturing

The user should give you a `batch_episodes.py` script that you can modify if needed. Before running it, check the robot's config and set CLI args accordingly (number of cameras, component names, etc).

This can be told to upload a dataset at the end. Use the viam python sdk to download the dataset and inspect the episodes.

For the initial smoke testing runs, create a composite image with a grid of episode end states from different cameras for the user to look at.

## Episode count schedule

Start low (5) so the user can investigate them and so you can debug success rate. The script allows you to add more episodes to a dataset if things work. Use separate datasets when logic changes, though.

You can download datasets through the SDK to inspect video and parquet.

A good number for a starter training run is 50. (A compromise between quality and generation / training time). Go to 100 if the resultant policy needs to be smoother.

## multiple tasks

it's possible to train a reasonable policy with a single task, and for simplicity, that's our beginner flow. if after that the user wants to increase model quality, multiple tasks will improve it (even for the original task prompt).
