# checkout-guard-repro

A controlled reproduction of the `actions/checkout` change that started failing
`pull_request_target` workflows on fork pull requests on **2026-07-20**.

## What changed

`actions/checkout` **v4.4.0** was published on 2026-07-20 at 15:36 UTC and
backported a guard that refuses to check out fork pull-request code from a
`pull_request_target` workflow. The previous v4 release, v4.3.1, was from
2025-11-17. Any workflow pinned to the floating `@v4` tag picked the new
behaviour up with no change on its own side.

The error:

```
Refusing to check out fork pull request code from a 'pull_request_target'
workflow. ... To opt in ... set 'allow-unsafe-pr-checkout: true' on the
actions/checkout step.
```

## The four runs

The workflow is the same in all four. Only one variable changes at a time.

| # | Fork PR? | checkout ref | opt-in | Result | Run |
|---|---|---|---|---|---|
| 1 | yes | `@v4` (now v4.4.0) | no | **failure** | [https://github.com/test-readme/checkout-guard-repro/actions/runs/29776897571](https://github.com/test-readme/checkout-guard-repro/actions/runs/29776897571) |
| 2 | yes | `@v4.3.1` pinned | no | success | [https://github.com/test-readme/checkout-guard-repro/actions/runs/29776935662](https://github.com/test-readme/checkout-guard-repro/actions/runs/29776935662) |
| 3 | yes | `@v4` | **yes** | success | [https://github.com/test-readme/checkout-guard-repro/actions/runs/29776977611](https://github.com/test-readme/checkout-guard-repro/actions/runs/29776977611) |
| 4 | no | `@v4` | no | success | [https://github.com/test-readme/checkout-guard-repro/actions/runs/29777030983](https://github.com/test-readme/checkout-guard-repro/actions/runs/29777030983) |

Reading it:

- **1 vs 2** isolates the cause. Same pull request, same content, same workflow.
  Only the action version differs, and that alone flips the outcome.
- **1 vs 4** isolates the trigger. Same floating `@v4`, no opt-in either time.
  Only the fork differs. A branch inside the base repository is unaffected.
- **3** validates the fix on the current `@v4`.

## The detail that matters most

In run 1 the job dies at step 3. The `Run validation` step is **skipped**, so
the workflow never reads the pull request's content at all. A submission cannot
influence this outcome, whatever it contains.

## The fix

Add one line to the step that checks out the fork's head:

```yaml
      - name: Checkout PR head (untrusted data, never executed)
        uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          repository: ${{ github.event.pull_request.head.repo.full_name }}
          path: pr
          persist-credentials: false
          allow-unsafe-pr-checkout: true
```

For a workflow that only reads the checked-out files as data and never executes
them, this restores the previous behaviour without widening what the workflow
trusts. Pinning to `@v4.3.1` also works but leaves the guard off for good.
