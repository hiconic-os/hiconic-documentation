# Background work

> Status: skeleton. Each section states the question it must answer. Answers are written in Phase 3, from code.

**Scope.** Work that runs without a caller: workers, schedules, and work that must run on one node only.

## Workers

How is a Worker declared, started and stopped?

## Scheduling

How is a job scheduled, and where is the schedule configured?

## Locking

How do two nodes agree that only one of them proceeds?

## Leadership and cluster singletons

How does one node become the leader, and how does a singleton use that?

## Messaging

What does the messaging abstraction offer, and which backends exist?

## Topology

How does a node learn about the other live nodes?

## Cheat sheet

| Module or type | Purpose |
|---|---|

## See also

- [application.md](application.md)
- [persistence.md](persistence.md)
