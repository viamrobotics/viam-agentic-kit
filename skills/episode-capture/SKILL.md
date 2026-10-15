---
name: episode-capture
description: Guidance for creating training data for policy training
---

## Helper scripts

In the cosmos-tutorial repo

when calling these scripts, pass down `VIAM_API_KEY` and `VIAM_API_KEY_ID` environment for the target robot from the mcp. make sure to use a key scoped to the robot.

> THAT DOESN"T WORK ^^^

## Manual run-throughs

...


when you lift something, check the simulator state to make sure that it really was lifted.

when you issue mcp grip commands, use `disable_sessions=True` to prevent it from relaxing when the connection closes. only do this for grip commands.

## Capturing

The user should give you a `batch_episodes.py` script that you can modify if needed. Before running it, check the robot's config and set CLI args accordingly (number of cameras, component names, etc).

This can be told to upload a dataset at the end. Use the viam python sdk to download the dataset and inspect the episodes.

For the initial smoke testing runs, create a composite image with a grid of episode end states from different cameras for the user to look at.

## Episode count schedule

Start low (5) so the user can investigate them and so you can debug success rate. The script allows you to add more episodes to a dataset if things work. Use separate datasets when logic changes, though.

You can download datasets through the SDK to inspect video and parquet.

A good number for a starter training run is 50. (A compromise between quality and generation / training time). Go to 100 if the resultant policy needs to be smoother.
