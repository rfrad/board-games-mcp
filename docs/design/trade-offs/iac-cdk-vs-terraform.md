# Infrastructure as code: CDK or Terraform

## Context

CDK came from the early repo structure, mostly because it fits the all-TypeScript and AWS-native approach. Its syntax is very Java-like and does a lot under the hood, which makes it harder to follow, and there was a later worry that it builds a lot of resources and is a black box.

## Options considered

### Option 1: AWS CDK

#### Description

Infrastructure is written in TypeScript using high-level constructs. A cdk synth prints the full CloudFormation template, and a cdk diff shows what will be added, changed or removed before deploying.

#### Pros

- Same language as the server and engine.
- High-level constructs mean a Lambda behind API Gateway takes only a few lines, with bundling, the execution role, the invoke permission and a least-privilege table policy generated automatically.
- No state file to manage, since CloudFormation tracks it.
- The synth template and the diff make every resource visible before deployment.

#### Cons

- The syntax is very Java-like and does a lot under the hood, which makes it harder to follow and complicated to implement.
- High-level constructs expand a lot. One line for an API with an authorizer can produce a dozen resources.
- There is no saved plan file, and the diff reads less cleanly than a Terraform plan.
- Edits made by hand in the console only show up through drift detection.

### Option 2: Terraform

#### Description

Infrastructure is declared in HCL. Every resource is explicit, and a plan shows what will be created.

#### Pros

- HCL is declarative and easy to read, and the plan output is clear.
- Resources are listed plainly, so it does what it says on the tin, even if it is wordier.
- Widely used in industry.
- Fits the GitHub secrets approach, since secrets become Terraform variables set in the Actions workflow.

#### Cons

- Needs remote state, usually an S3 bucket, which has to exist before the first run.
- Packaging the Lambda zip is more manual.
- The infrastructure becomes a second language next to TypeScript.
- Roles, permissions and every other resource have to be written by hand, which means more code and more places to get IAM wrong.

## Decision

CDK, chosen by the author after the options were laid out. The black-box worry is answered by reviewing the synthesized output and the diff before deploying. Fallback, only if reviewing the synthesized output proves too hard: use low-level constructs, which map one to one onto CloudFormation resources.
