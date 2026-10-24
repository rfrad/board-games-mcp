# Authentication: Cognito app client with or without a secret

## Context

Claude signs in to the server through a Cognito app client. Cognito cannot register clients automatically, so each tester types the client ID into Claude's custom connector settings, plus a client secret if the app client has one. The question was whether the app client should have a secret.

## Options considered

### Option 1: App client with a secret

#### Description

A confidential client. Swapping the one-time sign-in code for a token also requires the client secret. The secret is emailed to every tester with the client ID and entered in the connector settings.

#### Pros

- An intercepted sign-in code cannot be swapped for a token without the secret.

#### Cons

- PKCE, which Claude always uses, already protects the sign-in code, so the secret adds little.
- One secret is shared by every tester, so a single leak exposes it for everyone.
- Changing the secret means re-emailing every tester.
- Does not stop someone who holds a valid username and password.

### Option 2: App client without a secret

#### Description

A public client. Testers only receive the client ID, which is not sensitive. The sign-in code is protected by PKCE.

#### Pros

- Simpler invite email, with nothing secret beyond each tester's own password.
- Nothing shared between testers that could leak.

#### Cons

- Anyone who knows the client ID can open the sign-in page, but still needs an account to get past it.

## Decision

No secret. PKCE already protects the sign-in code, and a secret shared by every tester would add friction without real protection. The real safeguards are invite-only accounts, passwords, throttling and the budget alert.
