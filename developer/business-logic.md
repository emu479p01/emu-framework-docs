# Add business logic

## Purpose

Choose the smallest server-side mechanism that matches the behavior and keeps authorization, validation, and transactions consistent.

## Audience

Application developers and reviewers.

## Prerequisites

An application with its tables, permissions, and metadata workflow defined.

## Choose the mechanism

| Behavior | Use |
| --- | --- |
| Record defaults or validation | [Hooks and data events](hooks-events.md) |
| Pre/post lifecycle behavior | [Hooks and data events](hooks-events.md) |
| Explicit user operation | [Function/action](functions.md) |
| Several related registrations | [Script](scripts.md) |
| Native or external integration | Reviewed TypeScript |

Execute related writes inside a transaction through `DataContext`. Keep validation deterministic, avoid network calls inside database transactions, and enforce authorization on the server even when the client hides an action.

## Testing

Test success, rejection, rollback, unauthorized access, and concurrent behavior. See [Run tests and debug](testing.md).

## Related topics

[Architecture](architecture.md) · [Testing](testing.md)
