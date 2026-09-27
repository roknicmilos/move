# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commits

One-line imperative subject, no body (e.g. "Add User entity and CRUD endpoints").

Never stage or commit changes on your own initiative — only when the user explicitly asks for a
commit in that turn. When they do, `git add` everything outstanding and make a single commit with
one one-line message, rather than splitting the outstanding changes into several commits.

Never switch branches (checkout or create) unprompted — commit on the current branch, even `main`.

## Implementation workflow

When implementing a step from `specs/specs-mvp.plan.md` Part 3 that includes backend changes, do it in this order,
as separate turns:

1. **Backend tests first.** Write or update only the backend tests for that step's changes — no implementation yet.
   These tests are expected to fail at this point.
2. **Stop and wait.** Let the user review the tests and run them manually. Do not run the tests yourself and do not
   start on the implementation until the user tells you to continue.
3. **Backend implementation** that makes those tests pass.
4. **Frontend changes** for the step (tests and implementation together, no separate pause).

A step with no backend changes skips straight to frontend (tests and implementation together).
