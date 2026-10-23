# Agent instructions

Instructions for AI assistants working on this repository.

## Stateful resources

Never change a setting on the Cognito user pool or the DynamoDB table that forces CloudFormation to replace the resource without flagging it to the owner first. Replacing the user pool deletes every user. Replacing the table leaves the old table orphaned and points the stack at a new, empty one.
