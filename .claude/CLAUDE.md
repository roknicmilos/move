# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commits

One-line imperative subject, no body (e.g. "Add User entity and CRUD endpoints").

Never stage or commit changes on your own initiative — only when the user explicitly asks for a
commit in that turn. When they do, `git add` everything outstanding and make a single commit with
one one-line message, rather than splitting the outstanding changes into several commits.

Never switch branches (checkout or create) unprompted — commit on the current branch, even `main`.
