# Game choice and content sourcing: Warhammer Age of Sigmar or Catan

## Context

The project first targeted Warhammer Age of Sigmar, a game whose publisher restricts how its rules content can be copied and used with AI. Teaching content is the core of the product, so the licensing position decides what can be written, stored and published. The question was whether to keep that game or pivot to one with no such restriction.

## Options considered

### Option 1: Keep Warhammer Age of Sigmar (copyright-restricted)

#### Description

Continue with Warhammer Age of Sigmar and try to work within its publisher's content and AI-use restrictions, for example by paraphrasing or keeping content strictly private.

#### Pros

- Matches the original idea and the game the author cared about most.
- Rich, well-known rules with plenty of depth to teach.

#### Cons

- Copyright and AI-use restrictions make it unclear what content can be written, stored or shown to users.
- The risk lands on a public repo and a live endpoint, where a complaint is easy to trigger.
- Content would need heavy legal caution, which slows every content change.

### Option 2: Pivot to Catan as the v1 reference game

#### Description

Build a generic board game MCP server and use Catan as the first reference game, with teaching content written independently rather than copied from the official rulebook.

#### Pros

- Catan has no equivalent IP or AI-use restriction, so content can be written freely, including with AI assistance.
- The plugin architecture stays content-agnostic, so other games can be added later.
- A generic server is a more reusable deliverable than a single-game one.

#### Cons

- Drops the original game, at least for v1.
- Teaching content for Catan still has to be written independently, not copied.

### Option 3: Keep Warhammer Age of Sigmar, but only as private content

#### Description

Ship the generic server publicly and load any restricted game's content only into private deployments.

#### Pros

- Keeps the original game reachable without publishing its content.

#### Cons

- The public release would have nothing to show for that game.
- Still needs a licensing answer for any shared or invited use.

## Decision

Pivot to a generic board game MCP server with Catan as the v1 reference game. The licensing risk of Warhammer Age of Sigmar is removed, and content can be written freely. As a related safeguard that applies to every game, content is stored in DynamoDB and loaded manually. It is never committed to the repo, so the pattern stays safe even when a future game has restrictions. Adding another game later remains possible, subject to its own content-sourcing constraints.
