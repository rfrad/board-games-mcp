# Tool prerequisites: enforced or informational

## Context

Many tools are exposed to Claude, and nothing told it where to start. The idea was that a tool should list which other tools are best used first.

## Options considered

### Option 1: Enforced by the server

#### Description

The server refuses to run a tool until its prerequisites have been completed.

#### Pros

- Guarantees the learner follows the intended order.

#### Cons

- The server would need to track what the learner has completed, which pulls in session or user state. That is the deferred game-state feature.
- Learners who already know part of the game would be prevented from learning what they want, and people would not use the server.

### Option 2: Informational only

#### Description

Each tool's metadata lists the tools recommended first. Claude uses that, with the description, to decide what order to suggest, and nothing blocks a direct call.

#### Pros

- Fits the stateless design, since it is only extra metadata with no tracking.
- Someone who already plays a similar game is not forced through a beginner tutorial.
- A tool with no prerequisites and a start-here flag becomes the natural entry point, with no dedicated entry tool.

#### Cons

- Nothing stops Claude from calling a tool out of order.

## Decision

Informational only. Prerequisites are metadata, and the server does not enforce them.
