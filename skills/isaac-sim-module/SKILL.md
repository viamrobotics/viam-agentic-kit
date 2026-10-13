---
name: isaac-sim-module
description: Tips for working with the viam-labs:isaac-sim module (https://github.com/viam-modules/viam-isaac-sim)
---

## Architecture

the module ships several component modules like arms and camera, and also a central :world model that has settings that apply to the sim generally.

## Populating a scene

(todo: how prims work in :world attrs, embedded USDs, where to look for new arms and how to connect them to the :arm model, using blender where needed, ref to workcell-design)

## Framesystem

(todo: fill in which components respect framesystem and which don't)

## Viam UI 3d viewer

The 3d scene tab in the viam UI is useful for previewing and editing your layout

## When to reset

some state can be refreshed without a module restart, e.g. you can more camera framesystems.

## Caching and startup slowness

isaac can be slow on first start or after some scene modifications because it needs to rebuild shader state. this happens in sim world steps, not immediately on boot. look for isaac-sim:world model reconfiguring for multiple minutes. restarting the module should be quick because these things are cached. if restarts are slow as well, your cache location may be getting deleted. note it is sometimes in the venv.
