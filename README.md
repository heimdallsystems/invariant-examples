# Invariant examples

Runnable examples of [Invariant](https://github.com/heimdallsystems/invariant)
checking AWS SCP, RCP and IAM policy changes in CI.

## LZA Universal Configuration

[`lza-universal-configuration/`](lza-universal-configuration/) is an unmodified
copy of AWS's [LZA Universal Configuration](https://github.com/aws/lza-universal-configuration)
v1.4.0, used here as a realistic landing zone. Its policies are under
`modules/base/default/`. Heimdall Systems is not affiliated with AWS.

The [workflow](.github/workflows/invariant.yml) runs two jobs on every pull
request and every push to `main`:

| Job | What it checks | How it gates |
| --- | --- | --- |
| `guardrail-baseline` | Every SCP and RCP against the LZA guardrails bundled with Invariant, and every IAM policy for privilege-escalation actions | Reports to code scanning. A pull request that adds a new error-level alert fails the **Code scanning results** check |
| `guardrail-regression` | Each changed policy against its reviewed copy in [`reference/`](reference/): SCPs and RCPs must still deny everything they denied, and IAM policies must grant nothing new | Fails the job on any failed check |

### Findings on `main`

The unmodified configuration already has findings, shown as code scanning
alerts rather than hidden:

- `INV_AWS_003` on `iam-policies/sample-end-user-policy.json`: the policy
  grants IAM actions such as `iam:PassRole` and `iam:CreatePolicyVersion` on
  some resources. Its grants are scoped to specific role and policy ARNs, so
  a reviewer decides whether that scope is acceptable.
- `INV_AWS_004` on several guardrails: statements that use condition operators
  the solver doesn't model differ from the guardrails bundled with Invariant,
  which come from an earlier release of this configuration. They need review
  by hand.
- `lza-deployment-guardrails-1.json` and `-2.json` have no bundled counterpart,
  so the baseline job skips them. The regression job still covers them.

### Example pull requests

Each example is an open pull request, so you can see the real checks:

- **Allow deleting AWS Config rules**: removes `config:DeleteConfigRule` from
  the `GRCFGR` guardrail. Expected: the regression job fails with a
  counterexample naming that action.
- **Add a CI deploy role policy**: a new IAM policy granting `iam:PassRole`
  on every resource. Expected: a new `INV_AWS_003` alert fails the code
  scanning check.
- **Remove a duplicate S3 resource**: removes a repeated ARN from
  `ssm-s3-policy.json`. Expected: both jobs pass.

## Run it yourself

```bash
python -m pip install invariant-sast==0.3.5
invariant scan --directory lza-universal-configuration/modules/base/default
```

## What a result means

A failed check comes with a concrete request or a named action. A pass checks
individual policy documents against Invariant's model; it isn't proof of
account-wide security, and it doesn't compute effective permissions. See
[how the solver works](https://heimdallsystems.github.io/invariant/how-it-works/).

## Licence

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
