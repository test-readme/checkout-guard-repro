# checkout-guard-repro

Minimal reproduction of the `actions/checkout` v4.4.0 change that refuses to check
out fork pull-request code from a `pull_request_target` workflow.

The workflow here is the same shape as `logos-co/lambda-prize`'s submission
validator: a trusted checkout of the base, then a second checkout of the fork's
head, read as data and never executed.

Watch the Actions tab. The point to notice is that when the guard fires, the
`Run validation` step is **skipped** — the job dies before it ever reads the
pull request's content.
