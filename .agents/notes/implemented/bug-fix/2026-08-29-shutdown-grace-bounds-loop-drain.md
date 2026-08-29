# Agent Note: Shutdown grace bounds the drain, not only disposal

Status: implemented

English | [中文](2026-08-29-shutdown-grace-bounds-loop-drain.zh.md)

## Problem

`createProcessShutdown` armed a five-second timer, disposed the plugin tree, and on success cleared that timer and recorded an exit code. Exiting then depended on the event loop draining by itself. Disposal releases what the tree owns, but a provider's keep-alive TCP sockets are not part of it: after `dsh --profile headless` answered a task that delegated nothing, three established sockets held the loop and the process never returned to its caller. The one guard against that outcome was disarmed precisely when disposal reported success.

## Decision

The grace timer stays armed across a successful disposal, so five seconds bounds disposal *and* the drain that follows. A process still held at that deadline exits with the code it already resolved. The timer is unreferenced, so it never keeps the loop alive on its own and a run whose handles do drain still exits immediately.

Failure and signal paths are unchanged: a rejected disposal and the surfaces that pass `forceAfterDispose` still force exit at once, and a second signal still bypasses the grace entirely.

## Alternatives considered

**Find and close the offending handles.** Rejected because the set is open-ended and not the CLI's to enumerate — any plugin, provider, or transitive dependency can open a handle the tree does not own, and one missed owner reintroduces a hang that presents as a silent stall rather than an error.

**Force exit unconditionally once disposal resolves.** Rejected because it discards the natural drain that lets buffered output flush, and `forceAfterDispose` already marks the surfaces that genuinely want an immediate exit.

**Leave the timer referenced.** Rejected because a referenced timer keeps the loop alive by itself, converting every fast clean shutdown into a five-second wait.

## Consequences

A one-shot run returns to its caller even when something outside the tree holds the loop, which is what a non-interactive supervisor requires. The cost is that a drain still making progress at five seconds is cut off; that budget was already the documented ceiling for shutdown, now applied to the whole of it. `apps/cli/tests/process-shutdown.spec.ts` pins the new path — disposal resolves, no exit is forced, and advancing to the deadline forces one — alongside the existing natural-completion and signal cases.
