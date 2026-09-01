# selection-loop

Reusable Codex skill for selecting exact inclusive ranges across paginated checkbox lists and result tables.

## Contents

- `SKILL.md` — the skill instructions and operating contract.
- `agents/openai.yaml` — Codex agent metadata for the skill.

## Scope

The skill is designed for authenticated web pages that split a result set across multiple pages. It maps each visible row to a global 1-based item number, applies the requested inclusive range, clears stale selections outside that range, and verifies the site-reported selected count.

The skill does not export, submit, delete, or otherwise create side effects unless the user explicitly requests that action.

## Installation

Copy this repository into the local Codex skills directory as `selection-loop`, or install it using the skill-management workflow provided by your Codex environment.

## Privacy

This repository is private by default. The skill contains no credentials or authentication data. Keep it private unless you intentionally want to distribute the skill publicly.
