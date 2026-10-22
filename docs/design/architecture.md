# Board Game MCP Server — Project Spec

## Overview

An MCP (Model Context Protocol) server, deployed on AWS serverless infrastructure from a public GitHub repo. The server's purpose is to teach board games to someone who has never played, step by step, from absolute zero, using a generic plugin architecture so any game's rules can be added. Catan is the reference game for v1.

The project deliberately separates two concerns: an MCP server (the protocol and gateway layer) and a board game rules engine (a plugin-based library of teaching content, kept decoupled so it could later become its own service). The rules engine is intentionally simple and in-process for v1. The MCP server and its plugin architecture are the actual deliverable; each game's content, starting with Catan, is a reference implementation of that pattern, and more games can be added later using the same plugin interface, subject to each game's own content-sourcing constraints.

**v1 scope:** stateless rules and teaching content only, no game-state tracking. Each plugin represents one game and exposes a list of tools (topics such as setup or turn structure), each returning structured steps, each with a short content block and a check-in question Claude can use to confirm understanding before moving on. Tools can declare informational prerequisites and a startHere flag so Claude can suggest a sensible order, but nothing is enforced server-side. See Plugin interface.

## Architecture

**Compute:** AWS Lambda running the MCP server (TypeScript), chosen to minimise cost. It scales to zero, so idle time between dev sessions or actual use costs nothing.

**Transport:** Streamable HTTP fronted by an API Gateway HTTP API. The official @modelcontextprotocol/sdk package (McpServer class) is used directly rather than a wrapper framework, to keep what the server is doing transparent. It supports Streamable HTTP out of the box, which is part of why TypeScript was chosen for the server over Python.

**API type and throttling:** an HTTP API (not a REST API) with a Cognito JWT authorizer that validates the token's issuer and audience, rejecting unauthenticated requests before they reach the Lambda. HTTP APIs cost roughly a third as much per request as REST APIs and deploy changes automatically, so there is no separate API redeploy step. The default execute-api endpoint is disabled (disableExecuteApiEndpoint), so the only way to reach the API is through the Route 53 custom domain. Route-level throttling (rate and burst limits) is set in CDK at conservative values for an invite-only service, to cap the damage if a valid token is abused or leaked. REST-only features such as WAF integration, usage plans and API keys are deliberately not used.

**Domain:** each environment has its own domain name, with a Route 53 record pointing at an API Gateway custom domain (with an ACM certificate), so users connect to a stable URL rather than the raw API Gateway address. The domain name is environment-specific and kept in GitHub Environment secrets, never committed. CloudFront is not part of the v1 design; if it is added later, the deploy role also needs permission to invalidate its cache (see Manual setup steps).

**Auth:** Amazon Cognito as the OAuth 2.1 provider, with API Gateway validating bearer tokens before they reach the Lambda. Built in from day one so individual users can be issued their own credentials and try the server themselves. The Cognito free tier covers up to 10000 monthly active users, so realistic usage should cost nothing. The Cognito user pool includes a developer role/group, included as a token claim, which the error-handling logic (below) reads to decide how much error detail to return.

**Rules engine:** a TypeScript library bundled into the same Lambda deployment package (no network hop, no extra cost). The rules content itself is not in the code: plugins read it from DynamoDB. Structured as a plugin system, see Plugin interface below, so each teaching topic is self-contained, and the engine could later split into its own service without redesigning its interface.

**Lambda handler flow:** the MCP TypeScript SDK handles the JSON-RPC envelope, request IDs and transport-level parsing, so the handler itself is mostly glue. It creates the MCP server instance, wires up Streamable HTTP in stateless mode (session ID generator left undefined) with enableJsonResponse set to true (plain JSON responses, no SSE streaming; these are two separate settings), and registers two handlers. Stateless mode means a new transport and server instance is created per request for full isolation, which fits Lambda's per-invocation model naturally.

The list-tools handler calls registry.listAllTools() and returns the result directly.

The call-tool handler receives a tool ID and arguments, including a step number (0-based, default 0). It does minimal input-shape validation at this boundary only: checking the step argument, when provided, is actually a number, and rejecting anything missing or malformed with a tool-level error, before calling registry.getStep(toolId, step). It deliberately does not validate whether the step is in range: only the engine knows how many steps a tool has, so that check happens once, in the engine, against DynamoDB, rather than duplicated at the handler boundary. A successful result is wrapped into the MCP tool response; any failure is handled the same way described below, with the same five logged fields and the same role-based response detail.

**Error handling:** MCP separates transport-level failures (malformed requests, handled by the SDK) from tool-level errors, which still return HTTP 200 and a normal JSON-RPC response, but with isError: true on the tool result. Every error the handler encounters (unknown tool ID, step number past the end, DynamoDB unavailable, or anything unexpected) is treated as a tool-level error, so Claude always gets a 200 it can read and react to conversationally, rather than a hard failure.

Each error is logged to CloudWatch Logs, queried with Logs Insights, with five fields: an error code (machine-readable, e.g. DYNAMODB\_UNAVAILABLE), a user-friendly message (plain language, safe to show someone with no technical background), a developer explanation (specific, technical, for someone reading logs), the raw error description and stack trace, and a runbook link pointing a developer straight to remediation steps for that code. Runbooks are written as markdown files in a docs/runbooks/ folder in the repo, one file per error code, so the runbook link in each log entry can point straight to the matching file.

How much of this comes back in the tool response depends on the caller's role. Cognito tokens carry a role claim; a developer role gets the full error object back (code, developer explanation, runbook link included), while any other caller only gets the user-friendly message, wrapped in the normal isError tool result. This requires a role or group to be configured in Cognito and included as a token claim, in addition to the authentication Cognito already provides.

Deferred to a future iteration: Datadog observability; game-state tracking between turns; plugin dependency ordering.

## CI/CD pipelines

GitHub Actions, with workflows in .github/workflows/. There is one reusable deploy workflow, written once and used for every environment, so the way code reaches dev is by construction the same way it reaches prod. Three triggers call it, each passing the list of environments it applies to. Alongside it sit PR checks and housekeeping.

**Environments:** dev and prod are two separate AWS accounts, both member accounts of the owner's existing AWS Organization, running identical CDK stacks with the same structure. Only environment-specific values differ. These values (account ID, deploy role ARNs, domain name, e2e coverage level, and so on) are stored as secrets and variables on a GitHub Environment per AWS account, rather than repo-level, so each job only sees the values for the environment it targets. Both dev and prod require a reviewer, so every deploy to either environment pauses at the accept gate until the owner has read the cdk diff and approved. The gate is a setting on the GitHub Environment, not workflow code, so it can be removed from dev later without touching any workflow. Workflows authenticate to AWS through GitHub OIDC, assuming an IAM role in the target account, with no long-lived access keys. The role's trust policy is restricted to this repo and to that one GitHub Environment. Adding a further environment (for example test) means a new AWS account, a new GitHub Environment and its trust policy, and adding its name to the environment list of whichever trigger should include it. No workflow step and no test code changes.

**Checks:** install, lint, typecheck, unit tests for server and engine, and cdk synth. They run first in every triggered run and have no AWS credentials at all, so forks and outside contributors can never reach either account. A pull request opened from a fork gets these checks only: the deploy matrix is skipped, because secrets are withheld from fork runs and the deploy could only fail.

**Triggers:** each trigger calls the same reusable workflow and supplies its own list of environments. The workflow does not know why it was called; it loops over the list it is given, running one independent run per environment. (1) Pull request opened or updated, from a branch inside this repo: environments = dev. (2) Merge to main: environments = dev and prod. (3) Release build, fired when a version tag is created: environments = dev and prod. Cutting a release is always a manual action by the owner (creating the tag or publishing a release on GitHub). Nothing in the pipeline bumps versions or creates releases automatically, and that must not be added without a deliberate decision.

**Reusable deploy workflow:** for each environment in its list, it runs the same four steps in order. (1) cdk diff, printed to the run summary, listing every resource that will be added, changed or removed, with IAM changes shown separately. On the first deploy to an environment this is every resource in the stack. (2) Accept gate: a job that targets the GitHub Environment, which has required reviewers, so the run pauses until the owner approves or rejects after reading the diff. A rejection stops that environment's run. (3) cdk deploy. (4) End-to-end tests against the environment just deployed. The environments run in parallel as a matrix with fail-fast disabled, and they are fully independent: prod does not wait for, or depend on, dev passing, so check the dev run before approving prod.

**Deploy role:** the deploy workflow assumes bgmcp-infra-deploy (short-lived credentials, permission only to assume the CDK bootstrap roles) in the target account. There is deliberately a single workflow and a single deploy role: code changes go out through the same cdk deploy as infrastructure changes, which keeps the pipeline simple. A faster code-only deploy path with its own narrower role was considered and left out. Add one only if deploy times become a real problem.

**Expired approval:** GitHub expires a pending approval after 30 days. If that happens, the owner bumps the version (for example a hotfix patch number), creates a new release by hand, and the full flow runs again.

**Known limitation:** there is no concurrency control on the dev account, so two pull requests deploying at the same moment could collide. The dev approval gate makes this less likely, since each deploy waits for the owner. CloudFormation refuses a second update to a stack already mid-update, so the effect is a failed run that can simply be re-run, not corruption. A GitHub Actions concurrency group can be added later if this ever becomes a real problem.

**Note for reviewers:** the cdk diff is printed to the run summary of a public repo and relies on GitHub's secret masking, which does not catch transformed values (base64, URL-encoded, split across lines). This was accepted deliberately, with the default API endpoint disabled and no long-lived credentials in play. Please weigh whether that risk is acceptable and whether there is a cheap way to close the gap. See [trade-offs/secrets-in-public-cdk-diff-output.md](trade-offs/secrets-in-public-cdk-diff-output.md) for the full reasoning.

**Housekeeping:** Dependabot for dependency updates, npm audit, and GitHub secret scanning with push protection, which matter more on a public repo.

**End-to-end tests:** one codebase in e2e/, run by the same script in every environment as npm run e2e with a coverage argument. The runner knows nothing about environments, only about which tests to run: --coverage all runs everything, and --coverage smoke runs only the tests tagged as smoke tests (read-only, safe to run against prod). The deploy workflow reads the coverage level from an E2E\_COVERAGE variable on each GitHub Environment (all in dev, smoke in prod) and passes it through, so changing what runs in an environment is a variable change with no code change.

Teaching content is never committed, so the e2e suite cannot rely on the real Catan content. Tests that need data seed a small fixture plugin into that environment's DynamoDB table under its own partition and remove it afterwards, and these are not tagged as smoke tests. Every environment has a dedicated e2e test user in its own Cognito pool, with credentials in that environment's secrets, since even smoke tests need to sign in. In prod this user has no developer role. The Cognito user pool ID and client ID the tests sign in with are read from the stack outputs after cdk deploy and passed in as parameters, so they are never stored as secrets or variables.

### Manual setup steps

The pipelines cannot create the access they need to run, so the following steps are done by hand. Unless marked otherwise, each step is done once per account (dev and prod, identical in both). This list is not exhaustive: add to it whenever a new manual step turns up. Bootstrap is run locally with admin credentials, not from a GitHub Action, so no high-privilege role ever needs to exist in GitHub.

1. Create the dev and prod member accounts under the existing AWS Organization, if they do not already exist.
   - Sign in to the management (main) account.
   - Open AWS Organizations, then Add an AWS account, then Create an AWS account.
   - Name it board-game-mcp-dev and give it a unique email address (a plus-address of your own works).
   - Leave the default role name (OrganizationAccountAccessRole).
   - Repeat for board-game-mcp-prod.
   - Write down both account IDs.
2. Create an admin user with CLI access in each account (IAM Identity Center/SSO preferred over long-lived access keys). This identity is used only for the manual steps in this list and is never given to GitHub.
   - In the management account, open IAM Identity Center and enable it if it is not already on.
   - Create a user for yourself.
   - Create a permission set based on AdministratorAccess.
   - Assign your user to both the dev and prod accounts with that permission set.
   - Locally, run aws configure sso twice and name the profiles bgmcp-dev and bgmcp-prod.
   - Check each one works with aws sts get-caller-identity --profile bgmcp-dev (then bgmcp-prod).
3. Run cdk bootstrap locally with the admin credentials, once per account and region. Re-run it only if CDK later requires a newer bootstrap version.
   - Log in with aws sso login --profile bgmcp-dev.
   - Run npx cdk bootstrap aws://\<DEV\_ACCOUNT\_ID>/\<REGION> --profile bgmcp-dev.
   - Repeat both commands for prod with bgmcp-prod and the prod account ID.
   - In the CloudFormation console of each account, confirm the CDKToolkit stack shows CREATE\_COMPLETE.
   - Use the same region you will deploy to. If a certificate in another region is ever needed, bootstrap that region too.
4. Create the GitHub OIDC identity provider in each account.
   - Open IAM, then Identity providers, then Add provider.
   - Choose OpenID Connect.
   - Provider URL: https://token.actions.githubusercontent.com
   - Audience: sts.amazonaws.com
   - Click Add provider.
   - Repeat in the other account.
5. Create the infrastructure deploy identity: a CI/CD role assumed through OIDC, issuing short-lived credentials, with minimum permissions to run cdk deploy (in practice, permission to assume the CDK bootstrap roles rather than broad direct access).
   - Open IAM, then Roles, then Create role, then Web identity.
   - Pick the GitHub provider from step 4 and the audience sts.amazonaws.com, then enter your GitHub owner and repo.
   - Name it bgmcp-infra-deploy.
   - Edit the trust policy so the sub claim matches exactly one environment: repo:\<owner>/\<repo>:environment:dev in the dev account, and repo:\<owner>/\<repo>:environment:prod in the prod account.
   - Attach an inline policy that only allows sts:AssumeRole on the CDK bootstrap roles (arn:aws:iam::\<ACCOUNT\_ID>:role/cdk-\*).
   - Leave the maximum session duration at 1 hour.
   - Copy the role ARN for step 9.
6. Create the Route 53 record(s) for the domain in each account, pointing at the API Gateway endpoint. The domain name itself is environment-specific and never committed.
   - Decide the name for each environment (for example a dev subdomain and a prod subdomain).
   - Do this after the first deploy, once the API Gateway custom domain exists.
   - In Route 53, open the hosted zone for your domain and click Create record.
   - Create an A record, alias on, targeting the API Gateway custom domain.
   - If the custom domain needs a certificate, request it in ACM and add the validation CNAME it gives you to Route 53, then wait for the status Issued.
   - Check the name resolves with nslookup.
7. Create the first Cognito users by hand in each account: the developer-role user, and the dedicated e2e test user (see End-to-end tests). In prod the e2e user must not be in the developer group.
   - After the first deploy, open Cognito, then User pools, then the pool the stack created.
   - Click Create user, enter an email and a temporary password.
   - For the developer user, add them to the developer group.
   - In every account, create a second user for the e2e tests (in prod, leave it out of the developer group).
   - Set a permanent password for each (aws cognito-idp admin-set-user-password with the permanent flag, or sign in once and change it).
8. Load the teaching content into each account's DynamoDB table (content is never committed, so this is manual per account).
   - Keep the content files outside the repo folder.
   - Run the loader script from the repo against dev first (--profile bgmcp-dev), then against prod.
   - In the DynamoDB console, scan the table and confirm the items for the Catan plugin are there.
9. Create the dev and prod GitHub Environments, with required-reviewer approval on prod, and provide the GitHub Environment secrets for each.
   - In the repo, open Settings, then Environments, then New environment, and create one named dev and one named prod (the names must match the trust policy in step 5).
   - On both dev and prod, tick Required reviewers and add yourself, so you read the cdk diff before every deploy. Optionally restrict prod's deployment branches to main.
   - In each environment, add these secrets: AWS\_ACCOUNT\_ID, AWS\_REGION, INFRA\_DEPLOY\_ROLE\_ARN (the role the deploy workflow assumes), DOMAIN\_NAME (the environment's Route 53 domain), HOSTED\_ZONE\_ID (the Route 53 hosted zone CDK creates the record in). The API base URL is not stored separately: the workflow builds API\_BASE\_URL as https:// followed by DOMAIN\_NAME, so there is one value to maintain.
   - In every environment, also add the secrets E2E\_USERNAME and E2E\_PASSWORD, and a variable (not a secret) E2E\_COVERAGE set to all in dev and smoke in prod.
   - The Cognito user pool ID and app client ID are not stored anywhere. The stack exposes them as CDK stack outputs, and the deploy workflow reads them after cdk deploy and passes them to the e2e tests as parameters. They are identifiers, not credentials, so it is acceptable that they appear in the public deploy log; the e2e password stays a masked secret.
   - Check the secret names exactly match what the workflows reference.

**Ordering note:** steps 6 to 8 need resources that only exist after the first deploy. Do steps 1 to 5 first, create the GitHub Environments and the infrastructure role secrets from step 9, run the first deploy, then finish steps 6 to 8 and add the remaining secrets.

**Setup documentation in the repo:** anyone forking the repo has to supply their own value for every secret and variable above, so the repo includes docs/setup.md, a standalone guide written for a first-time forker. For each secret and variable it gives the exact name the workflows expect, what it is, where to find or create it (for example the account ID in the AWS console, the hosted zone ID in Route 53, the role ARN after creating the role), which environments need it and whether it is a secret or a variable, and which manual setup step produces it. It also covers creating the GitHub Environments, the order of the steps, and a checklist to confirm everything is in place before the first deploy. The values themselves are never written in the repo, only how to obtain them. The README links to it.

## Repo structure

**Getting started for testers and users:** access is manual and invite-only, not self-service sign-up, specifically to prevent automated scripts from registering accounts and running up usage costs or attempting denial of service against the live endpoint. The deployed API Gateway URL itself is also kept out of the public README entirely, staying only in GitHub secrets alongside the other environment-specific values, so the live endpoint isn't publicly discoverable. The README explains how to request access (e.g. a GitHub issue or a contact email); once a request comes in, the account is created manually in Cognito and the URL plus login credentials are sent directly to the requester by email, out of band from the repo. Access is via a custom connector in Claude (Settings, adding a custom connector for the OAuth-protected Streamable HTTP server), not the Claude Desktop config file. The README fully documents the connector setup steps (where to add a custom connector in Claude's settings, what connection type to choose, and completing the auth step) with the URL and credential fields left as placeholders, since only the private email gives the recipient the real values.

**For reviewers:** the repo includes a dedicated docs/for-reviewers.md, a standalone summary of the project that is also suitable for an AI assistant to read and relay to someone reviewing the repo on their behalf: what the project is, the two-part architecture (MCP server vs rules engine) and why they are separated, the key engineering decisions and the reasoning behind them (pulled from the Decisions log), and what is deliberately out of scope and why. The main README leads with the same plain-language what and why before any setup instructions. All of this is written to stand alone, readable and meaningful without running the stack or hitting the live endpoint, since a reviewer will not necessarily execute the CDK stack or have the invite-only credentials.

**Concurrency and DynamoDB versioning:** explicitly considered and dismissed as a non-issue, not overlooked. Lambda scales horizontally per-request by default (separate concurrent executions, no shared state), so request-level concurrency is already handled automatically. The only theoretical edge case is a read racing a content update in DynamoDB mid-write, but at this traffic volume (a low-traffic usage, versus production systems handling billions of writes a day) this is not a real-world concern and needs no explicit handling or versioning scheme.

**Stateful resource protection on redeploy:** redeploying the CDK stack must never overwrite or delete the DynamoDB table, CloudWatch log groups, or SSM parameters - only infrastructure (Lambda, API Gateway, Cognito config, etc.) should be replaced on redeploy. DynamoDB tables already default to RemovalPolicy.RETAIN in CDK (CDK applies RETAIN by default to stateful resources that can contain data), so this is close to automatic, but it's set explicitly on the table construct anyway for clarity and to guard against future CDK default changes. CloudWatch log groups are the opposite: CDK explicitly overrides CloudFormation's usual default and deletes log groups by default, so retention and RemovalPolicy.RETAIN must be set explicitly on every log group (including ones Lambda auto-creates, which should instead be defined explicitly as LogGroup constructs so the removal policy can be set). SSM parameters have no special stateful default either and need RemovalPolicy.RETAIN set explicitly if any are used to store data that must survive redeploys.

Tests are colocated: a unit test file sits as `<name>.spec.ts` next to the source file it tests, inside `server/src/` or `engine/src/`. These are excluded from the Lambda build bundle. End-to-end tests, which run against live deployed infrastructure (API Gateway, Cognito, Lambda together), live separately in a top-level `e2e` folder.

```
board-game-mcp/
README.md
AGENTS.md                    agent instructions for AI assistants
.github/
  workflows/
    deploy.yml               reusable: cdk diff, accept gate, cdk deploy, e2e (one run per environment)
    on-pull-request.yml      checks, then calls deploy.yml with environments = dev (same-repo PRs only)
    on-merge.yml             checks, then calls deploy.yml with environments = dev, prod
    on-release.yml           checks, then calls deploy.yml with environments = dev, prod (version tag)
    audit.yml                npm audit and other housekeeping
  dependabot.yml
docs/
  design/
    architecture.md
    trade-offs/              one markdown file per major trade-off
  setup.md                   how to obtain every secret and variable, for forkers
  for-reviewers.md
  runbooks/                  one markdown file per error code
infra/                       AWS CDK (TypeScript)
  bin/
  lib/
    mcp-stack.ts             Lambda + API Gateway + Cognito
    auth-stack.ts
  cdk.json
server/                      MCP server (TypeScript)
  src/
    handler.ts               Lambda entry point
    transport/               Streamable HTTP transport
    tools/                   MCP tool definitions, call into engine
    auth/                    token validation helpers
    (*.spec.ts colocated next to each source file)
  package.json
  tsconfig.json
engine/                      Board game rules engine (TypeScript library)
  src/
    core/                    plugin interface, registry
    plugins/
      catan/
        setup/
        turn-structure/
        building-and-trading/
    (*.spec.ts colocated next to each source file)
  package.json
  tsconfig.json
scripts/
  load-content.ts            loads teaching content into DynamoDB (the content itself is never in the repo)
e2e/                         end-to-end tests, run with: npm run e2e -- --coverage all|smoke
```

## Plugin interface

A plugin is not one single tool, it is a small namespace that exposes a list of tools, each carrying its own metadata. This keeps all tool metadata, including the description Claude reads to decide what to call, inside the plugin itself. The MCP server holds none of it directly - it just asks every registered plugin for its tools once at startup and builds the MCP tool list from that. Adding a new plugin should require no changes to the server's startup code.

A plugin exposes:

A plugin ID - identifies the plugin itself internally (e.g. "teaching").

A `listTools()` method, called once at server startup, returning a list of tools.

A `getStep(toolId, step)` method, called when Claude invokes a tool, returning that single step plus whether a next step exists.

Each tool in that list carries: an ID (unique, stable, e.g. "turn-sequence"), a name (human-readable, e.g. "Turn sequence"), a description (the exact text that becomes that tool's MCP description - this is what Claude reads to decide whether to call it), an optional read-only flag, an optional prerequisites list (other tool IDs recommended before this one), and an optional startHere flag. The teaching steps are not part of this list. They are fetched one at a time, each with a content block and a check-in question Claude can ask before advancing.

Prerequisites and entry point: prerequisites are informational only, never enforced server-side. A tool with prerequisites can still be called directly - someone who already knows the basics shouldn't be blocked from jumping to a specific topic. Claude reads the prerequisites list, the startHere flag, and each tool's description together to work out a sensible order and where to begin, rather than the server tracking progress or gating calls. This keeps the design stateless: no per-learner completion tracking is needed, unlike enforced dependency ordering, which was considered and rejected for exactly that reason.

```typescript
// engine/src/core/plugin.ts

export interface PluginTool {
  id: string;                 // unique, stable identifier (e.g. "turn-sequence")
  name: string;               // human-readable title (e.g. "Turn sequence")
  description: string;        // becomes the MCP tool's description Claude reads
  readOnly?: boolean;         // optional: maps to the MCP readOnlyHint annotation
  prerequisites?: string[];   // optional: other tool IDs recommended first (informational only, not enforced)
  startHere?: boolean;        // optional: flags this as a sensible entry point for new learners
}

export interface StepResult {
  step: number;             // 0-based index of the returned step
  totalSteps: number;
  hasNext: boolean;
  content: string;          // the explanation for this step
  checkIn: string;          // a follow-up question to confirm understanding
}

export interface Plugin {
  pluginId: string;         // identifies the plugin itself (e.g. "teaching")
  listTools(): Promise<PluginTool[]>;  // startup, metadata only
  getStep(toolId: string, step: number): Promise<StepResult>;
}
```

```typescript
// engine/src/core/registry.ts

export class PluginRegistry {
  private plugins: Plugin[] = [];
  private owners = new Map<string, Plugin>();

  register(plugin: Plugin): void {
    this.plugins.push(plugin);
  }

  async listAllTools(): Promise<PluginTool[]> {
    const lists = await Promise.all(
      this.plugins.map(async (p) => {
        const tools = await p.listTools();
        tools.forEach((t) => this.owners.set(t.id, p));
        return tools;
      })
    );
    return lists.flat();
  }

  getStep(toolId: string, step: number): Promise<StepResult> {
    const owner = this.owners.get(toolId);
    if (!owner) throw new Error(`Unknown tool: ${toolId}`);
    return owner.getStep(toolId, step);
  }
}
```

Plugins are independent for v1 - no dependency ordering between them (e.g. no "teach core-mechanics before turn-sequence" enforcement). Deferred to a future iteration.

How to add a new plugin (v1 process):

1. Create a new folder under `engine/src/plugins/<plugin-name>/`.
2. Implement the `Plugin` interface: a `pluginId` and a `listTools()` method returning one or more `PluginTool` entries (id, name, description), plus a getStep method that returns a step's content and check-in question.
3. Register the plugin instance with the `PluginRegistry` at startup.
4. Add a colocated `.spec.ts` test file next to the plugin's source file.
5. No changes to the MCP server's tool-registration code should be required - the server builds its tool list from whatever the registry returns, by design.

## Decisions log

A living log of choices and conventions made during design, so they don't need re-explaining later. Add to this as the project continues. Each major decision also has a standalone write-up in docs/design/trade-offs/.

**Language:** TypeScript for both the MCP server and the rules engine (v1). Rationale: the MCP TypeScript SDK has first-class Streamable HTTP support; keeping both in one language avoids a network hop between server and engine, which would add cost and complexity.

**Separation of concerns:** MCP server (protocol/gateway) and board game rules engine (teaching content) are separate, loosely coupled by a plugin interface, even though they currently deploy together. This is intentional so the engine could be extracted to its own service later without a redesign.

**Hosting:** AWS Lambda, chosen specifically for cost - scales to zero. Explicitly ruled out always-on options (ECS, App Runner) for this reason.

**API Gateway type:** HTTP API over REST API, because it is the cheapest option (about $1.00 versus $3.50 per million requests), validates Cognito JWTs natively, and deploys automatically with no redeploy step. REST-only features (WAF, usage plans, API keys, IP resource policies) are not needed for an invite-only access. Abuse is limited by route-level throttling set in CDK. Revisit if WAF or per-user quotas are ever required.

**Auth:** Cognito over Auth0, to keep the stack AWS-native, even though Auth0 is arguably nicer to use. Built in from day one (not deferred) so users can be issued individual credentials.

**Testing convention:** unit tests as colocated `.spec.ts` files next to their source; excluded from the Lambda build. End-to-end tests live in a separate top-level `e2e` folder since they run against live infrastructure.

**Scope discipline:** keep the rules engine simple; resist the urge to over-build it. The project's focus is the MCP server and its plugin architecture, not backend game-logic engineering.

**Configuration and secrets:** the repo is public, so nothing environment-specific is hardcoded (AWS account ID, role ARNs, credentials, domain name, and so on). These values live in GitHub Environment secrets, one set per environment (dev and prod), and are populated into the infrastructure code before it is executed. See CI/CD pipelines for how each job only reaches its own environment's values.

**CI/CD:** GitHub Actions. Dev and prod are separate AWS accounts under the same AWS Organization with identical stacks, rather than two stacks in one account, for cleaner isolation. One reusable deploy workflow (cdk diff, accept gate, cdk deploy, e2e tests) is used for every environment, so deploying to dev and to prod cannot differ. Three triggers call it, each passing its own environment list: pull request (dev), merge to main (dev and prod), and release build on a version tag (dev and prod). Environments run independently in a matrix, and prod does not depend on dev passing. Both dev and prod require a reviewer, so the owner reads the cdk diff before every deploy, which keeps the infrastructure CDK generates visible rather than a black box. The gate is configured on the GitHub Environment rather than in the workflow. E2E tests take a coverage argument (all or smoke) and never know the environment, so adding an environment needs no code changes. Releases and version bumps are always manual. See CI/CD pipelines in Architecture.

**Rules content storage:** game rules content is not hardcoded in plugins or committed to the repo, since some games have copyright or AI-use restrictions, and keeping content out of code keeps the pattern consistent across all games regardless of their individual restrictions. Content is stored in DynamoDB (serverless, scales to zero, fits the Lambda model) and plugins read it from there. The repo holds only the table schema and loader code, never the rules content itself.

**Infrastructure as code:** AWS CDK in TypeScript, kept in the same language as the server and engine. Terraform was considered and is a reasonable alternative (more widely used, flatter syntax), but CDK's high-level constructs and grant helpers keep the Lambda, API Gateway, Cognito and DynamoDB setup short, and generate least-privilege IAM automatically. Secrets from GitHub secrets are passed to CDK as environment variables or context values by the GitHub Actions workflow.

**DynamoDB design:** one table, on-demand billing. One item per tool, with PK PLUGIN#\<plugin-id>, SK TOOL#\<tool-id>, and attributes name, description, readOnly and steps (a list of content and checkIn objects). A tool's steps are far below the 400 KB item limit, so they live in the same item. At startup, listTools() queries the plugin's partition with a ProjectionExpression that excludes steps. Tool calls fetch one step at a time (optional step argument, 0-based, default 0) and return that step plus whether a next one exists, so the server stays stateless and Claude paces the lesson using the check-in questions.

**Content loading:** the teaching content is written once the stack is built, then loaded into DynamoDB manually. It is never committed to the repo.
