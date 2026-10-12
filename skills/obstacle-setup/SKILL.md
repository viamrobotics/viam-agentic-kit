---
name: obstacle-setup
description: Obstacle and workspace safety setup for Viam motion planning with viam:obstacle-pack:obstacle components. Covers the table obstacle and its default geometry, pick and place markers, wall and box placement math, workspace boundaries, and frame troubleshooting. Use when the user wants to add, size, or position a table, wall, bin, fixture, or other obstacle for an arm, or when motion planning ignores an obstacle or fails. Creates and changes machine configuration through the Viam MCP server.
compatibility: Requires the Viam MCP server (https://app.viam.com/mcp) to be connected and authenticated in the client, with permission to edit the machine's configuration.
---

# Obstacle and Workspace Safety Setup

Shared reference for obstacle placement math, fragment table/markers, and workspace boundaries. For creating or fixing standalone obstacles (two-call workflow, frames, anti-patterns), see "Creating or fixing an obstacle" below.

Create new obstacles as **`viam:obstacle-pack:obstacle`** (`api: "rdk:component:generic"`, module `viam:obstacle-pack`). That model publishes geometry via `GetGeometries` — it is not a gripper. Leftover **`erh:vmodutils:obstacle`** components (gripper-API containers from older setups or fragments) still work for planning; patch them in place, and never create new ones.

Never wire an obstacle name into an inline module's `gripper` attribute. New generic obstacles cannot satisfy that dependency. Fragment table/markers may still be leftover gripper-API obstacles; those reject `Open`/`Grab` with `obstacle can't open`.

## Creating or fixing an obstacle

The Viam MCP server's write tools save immediately, with no draft and no undo, so confirm the machine, the part, and the numbers with the user before calling them. Find the machine with `list_machines` and read its current config with `read_machine_config` first.

Creating an obstacle is always two calls, because `add_machine_config_item` cannot set a frame:

1. `add_machine_config_item` with `item_type` `component`, the obstacle's `name`, `api` `rdk:component:generic`, `model` `viam:obstacle-pack:obstacle`, `module_id` `viam:obstacle-pack`, and `attributes` holding only `geometries`: one geometry per component, with `type`, its dimensions in mm, and a `label`. Never put `parent` or any frame key inside a geometry entry; it is silently dropped.
2. `update_machine_config_item` for the same `name` with `patch` set to `{ "frame": { "parent": "world", "translation": { ... }, "orientation": { "type": "ov_degrees", "value": { "x": 0, "y": 0, "z": 1, "th": 0 } } } }`. `translation` is the shape's center in world mm.

To change an existing machine-owned obstacle, patch `attributes` for size or shape and `frame` for position or orientation with `update_machine_config_item`. A resource a fragment provides cannot be patched; the fragment has to change.

Verify with `read_machine_config` that the component has a `frame` whose `parent` is set. Without one, the planner never sees the geometry and the arm plans straight through it with no error.

## What plans around obstacles

| Motion path | Plans around obstacles? | Notes |
|---|---|---|
| Motion service / module `move_gripper_to` (gripper frame) | Yes | Used by the pick-and-place inline module and Step 4 DoCommands |
| 3D scene — move the **gripper** | Yes | Optional ad-hoc exploration; path preview before execute |
| Arm `MoveToPosition` / `move-to-position` test card | Avoid in pick-and-place setup | Targets the **arm frame** (flange), not the gripper — Z effectively floors around gripper length (~170 mm) |
| Arm `MoveToJointPositions` | No | Direct joint-space command; no collision checking |

## Table obstacle

Add the table obstacle up front with the default geometry below. Afterwards, ask the user for the real table dimensions (length, width, thickness in mm) and whether the arm base is centered on the table, then update the box to match.

Saving a new table pulls the `viam:obstacle-pack` module from the registry. The frame sits the box below the table surface: `translation.z = -(thickness / 2)`, so the top face lands at world z = 0.

Default configuration (1200 × 800 × 30 mm table, arm centered):

```json
{
  "name": "table",
  "api": "rdk:component:generic",
  "model": "viam:obstacle-pack:obstacle",
  "attributes": {
    "geometries": [
      {
        "label": "table",
        "type": "box",
        "x": 1200,
        "y": 800,
        "z": 30
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": 0, "y": 0, "z": -15 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

When the user's measurements come in, set the box's `x`, `y`, `z` to the measured length, width, and thickness, and recompute the frame's `translation.z`. If the arm base is not centered, offset the frame's `translation.x` / `translation.y` so the box covers the physical table relative to the arm base (which is the world origin).

Skip adding a second table when `read_machine_config` already shows a machine-owned `table` with a valid `frame.parent` — whether that component is `viam:obstacle-pack:obstacle` or a leftover `erh:vmodutils:obstacle`.

## Pick and place markers

Two small sphere obstacles act as visual markers at the pick and place **XY** positions (same x/y as the setup's workspace poses). They appear in the 3D scene so the user can see where to place the object and where it will be set down. Marker **z** sits just above the table surface (world z = 0), not at gripper TCP height — the module's hardcoded poses drive the actual grasp height.

They use the same `viam:obstacle-pack:obstacle` model as the table; each marker is its own component, and each needs its own frame positioning it in the world.

```json
{
  "name": "pick-marker",
  "api": "rdk:component:generic",
  "model": "viam:obstacle-pack:obstacle",
  "attributes": {
    "geometries": [
      {
        "label": "pick-marker",
        "type": "sphere",
        "r": 15
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": 300, "y": -150, "z": 15 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

```json
{
  "name": "place-marker",
  "api": "rdk:component:generic",
  "model": "viam:obstacle-pack:obstacle",
  "attributes": {
    "geometries": [
      {
        "label": "place-marker",
        "type": "sphere",
        "r": 15
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": 300, "y": 150, "z": 15 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

In the pick-and-place setup the table and both markers come from the `assisted-setup-ufactory-pick-and-place` fragment, which sizes and positions them from its variables. That fragment may still define them as leftover `erh:vmodutils:obstacle` grippers — do not replace those fragment resources from this skill. The fragment must define them with frames included.

## Additional workspace obstacles

Anything else the arm could hit — a wall, bin, fixture, piece of equipment — is one box component per item, created with the two calls above. Enclose the real object; over-approximating is safer than under-approximating.

From the user's description: the geometry takes the object's size (`x` front-to-back, `y` left-to-right, `z` height) and the frame's translation takes its center relative to the arm base — the reported x/y, with `z` = half the height for anything standing on the table.

A wall is the same box with one side made thin. Defaults: 50 mm thick, 2000 mm tall, spanning 2000 mm along the wall. Given a distance `d` in mm from the arm base center to the wall face, for thickness `t` and height `h`:

| Side | Geometry (mm) | Frame translation (mm) |
|---|---|---|
| forward (+x) | `x: t, y: 2000, z: h` | `{ "x": d + t/2, "y": 0, "z": h/2 }` |
| back (−x) | `x: t, y: 2000, z: h` | `{ "x": -(d + t/2), "y": 0, "z": h/2 }` |
| left (+y) | `x: 2000, y: t, z: h` | `{ "x": 0, "y": d + t/2, "z": h/2 }` |
| right (−y) | `x: 2000, y: t, z: h` | `{ "x": 0, "y": -(d + t/2), "z": h/2 }` |

The `t/2` term puts the wall's inner face exactly at the measured distance. The planner adds no clearance of its own, so subtract a margin (20–50 mm) from `d` if the arm should be kept clear of the real wall rather than allowed to touch it.

Example, a wall 600 mm to the arm's right:

```json
{
  "name": "wall-right",
  "api": "rdk:component:generic",
  "model": "viam:obstacle-pack:obstacle",
  "attributes": {
    "geometries": [
      {
        "label": "wall-right",
        "type": "box",
        "x": 2000,
        "y": 50,
        "z": 2000
      }
    ]
  },
  "frame": {
    "parent": "world",
    "translation": { "x": 0, "y": -625, "z": 1000 },
    "orientation": {
      "type": "ov_degrees",
      "value": { "x": 0, "y": 0, "z": 1, "th": 0 }
    }
  }
}
```

## Workspace boundaries

- Target poses must be within the arm's reach envelope. Poses too far from the base (or too close to it) will fail to plan.
- Ensure no physical obstructions sit in the planned path that are not modeled as obstacles.

## Troubleshooting: an obstacle has no effect, or does not appear in the 3D scene

Cause: The component has no `frame`, or its `frame.parent` is empty, so viam-server left it out of the frame system. The config looks right and the planner never sees the geometry.

Solution:

1. Read the component with `read_machine_config` and confirm it has a `frame` with `parent` set (`world` for a fixed obstacle)
2. If not, patch the frame with `update_machine_config_item` as in "Creating or fixing an obstacle"
3. If the obstacle comes from a fragment, the fragment has to be fixed — the tool cannot patch fragment-provided components
4. Confirm the shape appears in the 3D scene at the expected place

## Troubleshooting: motion planning fails

Cause: Table obstacle geometry incorrect or poses unreachable.
Solution:

1. Verify table dimensions and frame translation.z = -(thickness / 2)
2. Check that target poses are within the arm's reach envelope
3. Ensure no physical obstructions in the planned path
