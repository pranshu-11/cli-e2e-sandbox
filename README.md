# cli-e2e-sandbox

Throwaway sandbox repository for the Abacus.AI CLI end-to-end regression suite
(`apps/cli/src/__tests__/regression/cli-repl.test.ts`).

The suite clones this repo, lets the agent edit `user-service.md`, pushes a
`test-pr-<timestamp>` branch, opens a pull request against `main`, asserts the
title/body are well formed, then closes the PR and deletes the branch.

Do not put anything real in here. Branches and pull requests are created and
deleted automatically.
