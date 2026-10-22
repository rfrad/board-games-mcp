# Access: invite-only or open sign-up

## Context

People should be able to try the live server. The endpoint is public, so it must be protected against automated scripts that could push for denial of service or run up a large bill.

## Options considered

### Option 1: Open self-registration

#### Description

Anyone can create an account through a public sign-up page.

#### Pros

- No manual work for the owner.

#### Cons

- Scripts could register their own users, push for denial of service, or run up a huge bill.
- Unlimited sign-ups could run up Cognito costs if the repository gets attention.
- Bots crawling GitHub could find and abuse it.

### Option 2: Manual, invite-only accounts

#### Description

Accounts are created manually in Cognito for anyone who asks. The API URL is not published and is kept in GitHub secrets. When someone asks for access, the URL and login are sent only by private email. The README documents the connector setup steps with placeholders.

#### Pros

- More control, and protection against abuse.
- An extra layer of obscurity on top of the authentication.

#### Cons

- More hands-on for the owner.

## Decision

Manual, invite-only accounts with a hidden URL, shared privately by email with the credentials.
