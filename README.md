# dirstral-spec

Canonical, implementation-neutral specification repository for the Dirstral MCP ecosystem.

## Scope

This repo is the source of truth for:
- protocol and tool contracts
- schema and error taxonomy
- versioning and compatibility policy
- x402 extension contract

## Layout

- `docs/` human-facing normative and explanatory docs
- `spec/` machine-oriented contract artifacts and indexes

## Governance

- Spec changes require maintainer review.
- Breaking changes require a major version bump in `spec/versioning.md`.
- Implementations must not diverge from this repo without explicit version negotiation.
