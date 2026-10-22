# Secrets in the public cdk diff output

## Context

The repository is public, and the cdk diff is printed in the run summary so it can be read before approving a deploy. Values such as the account ID, region and domain are needed by CDK and could appear in that output.

## Options considered

### Option 1: Accept the masked baseline

#### Description

Leave the cdk diff in the public run summary and rely on GitHub secret masking. Set disableExecuteApiEndpoint in CDK so the raw API Gateway URL stops responding, and only the Route 53 custom domain works.

#### Pros

- No extra setup for a solo project.
- The raw API URL stops responding, so only the custom domain works.

#### Cons

- Relies on masking, which reviewers are asked to challenge. This is flagged in the design document and as a comment.

### Option 2: Send the diff out of band

#### Description

Email or notify the diff through SMTP, a transactional email service, SES or push.

#### Pros

- The diff never appears in the public run summary.

#### Cons

- Too much setup for a solo project.

### Option 3: Redact with a script

#### Description

Remove sensitive values from the diff output with a script.

#### Pros

- Keeps the diff in the run summary with sensitive values removed.

#### Cons

- Misses encoded values.

### Option 4: Keep values out of the template with SSM or Secrets Manager

#### Description

Store the values in SSM or Secrets Manager instead of passing them into the template.

#### Pros

- Would keep the values out of the template, if it worked.

#### Cons

- Does not work, because the account ID, region and domain are needed by CDK at synth time.

### Option 5: Make the repository private

#### Description

Keep the repository private so the run summary is not public.

#### Pros

- The run summary is not public.

#### Cons

- Defeats the purpose of a public project.
- Concerns about environment protection on the free plan were flagged but not verified.

## Decision

Accept the masked baseline, with disableExecuteApiEndpoint set. The risk is written in the design document, and a comment asks reviewers to challenge it.
