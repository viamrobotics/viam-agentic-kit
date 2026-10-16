---
name: isaac-sim-module
description: Tips for working with the viam-labs:isaac-sim module (https://github.com/viam-modules/viam-isaac-sim)
---

## Architecture

the module ships several component modules like arms and camera, and also a central :world model that has settings that apply to the sim generally.

## Populating a scene

you can define primitives and embedded USDs in the :sim:world model's attribute config.

when you add an object, use the get image API to read a few of the available cameras and inspect yourself to make sure the scene looks correct. it's okay to do a few turns of interactive improvement

if you're debugging reachability, offer to the user to run through target locations while they watch the camera.

## Framesystem

Most components (cameras, arms, etc) respect the viam framesystem; not all do. If you run into trouble positioning something with the framesystem, look for config attrs on the specific model.

## Viam UI 3d viewer

The 3d scene tab in the viam UI is useful for the user to understand x/y/z plus-minus relationships; if they're confused about axes, tell them to look there.

## When to reset

some state can be refreshed without a module restart, e.g. you can move camera framesystems.

## Caching and startup slowness

isaac can be slow on first start or after some scene modifications because it needs to rebuild shader state. this happens in sim world steps, not immediately on boot. look for isaac-sim:world model reconfiguring for multiple minutes. restarting the module should be quick because these things are cached. if restarts are slow as well, your cache location may be getting deleted. note it is sometimes in the venv.

## Keeping the simulation running

a simulation is billed per minute while it runs, but don't call `stop_simulation` to save money while the sim is being set up. stopping it mid-setup interrupts the user's work, and starting it again takes a few minutes before the part is back online. it already stops on its own when it sits idle. only stop it when the user explicitly asks.
