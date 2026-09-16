# Multistate AWS Infrastructure (CloudFormation)

Four raw-YAML CloudFormation stacks under [`cfn/`](../cfn/) provision the AWS substrate
that the W6 D1 CI pipeline and W6 D2 Argo CD GitOps loop assume exists. The EKS cluster
itself is platform-provided; these stacks provision everything else the cluster's
workloads depend on (network, IAM deploy role, database, artefact storage).

## Stacks and deploy ordering

Deploy in this order — each later stack consumes an earlier stack's `Export`s via
`!ImportValue`, so order matters and cannot be parallelized on first create.

| Order | Stack name                  | Template                                                            | Provisions                                                         |
|-------|------------------------------|-----------------------------------------------------------------------|---------------------------------------------------------------------|
| 1     | `multistate-bootstrap-dev`  | [`cfn/multistate-bootstrap-dev.yaml`](../cfn/multistate-bootstrap-dev.yaml) | Bootstrap S3 bucket + `multistate-api-cfn-deploy` IAM role (OIDC) |
| 2     | `multistate-network-dev`    | [`cfn/multistate-network-dev.yaml`](../cfn/multistate-network-dev.yaml)   | 3-AZ VPC, public/private subnets, NAT GW(s), app security group   |
| 3     | `multistate-artifacts-dev`  | [`cfn/multistate-artifacts-dev.yaml`](../cfn/multistate-artifacts-dev.yaml) | Hardened S3 artefact bucket (SAM + Argo CD config snapshots)      |
| 4     | `multistate-app-dev`        | [`cfn/multistate-app-dev.yaml`](../cfn/multistate-app-dev.yaml)           | RDS Postgres + Secrets Manager master credentials                 |

Stack 3 and 4 have no dependency on each other and can deploy in either order once
stack 2 exists; stack 4 imports `multistate-network-dev`'s `PrivateSubnets`, `VpcId`,
and `AppSgId` exports.

## The ChangeSet flow (used for every stack, every deploy)

Every stack — first create and every subsequent update — goes through the same
three-step flow. Never `aws cloudformation deploy` / `create-stack` directly; the
ChangeSet's diff is the reviewable artifact that goes in the PR.

```bash
aws cloudformation create-change-set \
  --stack-name <stack-name> \
  --change-set-name <descriptive-name> \
  --change-set-type CREATE   # or UPDATE for an existing stack
  --capabilities CAPABILITY_NAMED_IAM \
  --template-body file://cfn/<stack-name>.yaml \
  --region us-east-1

aws cloudformation describe-change-set \
  --stack-name <stack-name> --change-set-name <descriptive-name> \
  --region us-east-1
# review the resource-level diff here; paste the JSON into the PR body

aws cloudformation execute-change-set \
  --stack-name <stack-name> --change-set-name <descriptive-name> \
  --region us-east-1
```

`CAPABILITY_NAMED_IAM` is required on `multistate-bootstrap-dev` (the only stack that
creates an IAM resource — `CfnDeployRole`).

## Cross-stack references (`!ImportValue`)

`multistate-network-dev` exports `VpcId`, `PublicSubnets`, `PrivateSubnets`, and
`AppSgId` under names prefixed with the stack name (e.g.
`multistate-network-dev-PrivateSubnets`). `multistate-app-dev` imports these rather
than hardcoding subnet/SG IDs, so a network stack rebuild never leaves the app stack
pointing at stale IDs. CloudFormation enforces this at the platform level: as long as
an export is referenced by another stack, `delete-stack` on the exporting stack is
refused with an `Export ... is in use by stack` error — this is the safety the
`Export.Name` mechanism buys over passing IDs as plain parameters.

## Drift detection

`aws cloudformation detect-stack-drift --stack-name <stack-name>` followed by
`aws cloudformation describe-stack-resource-drifts --stack-name <stack-name>` is the
verification step after any console edit. A stack in `DRIFTED` state means a resource
property no longer matches the template — reconcile by either updating the template
to match (if the console edit was intentional) or reverting the console edit (if it
was not) and re-running detection to confirm `IN_SYNC`.

## cfn-nag findings and accepted tradeoffs

`cfn_nag_scan --input-path cfn/ --fail-on-warnings` was run for real in CI (the
`cfn-validate.yml` `cfn-nag` job) after fixing a local Ruby-toolchain issue on the
authoring machine (stale Ruby 2.6 + TLS interception blocked `gem install` locally;
the CI runner's fresh Ruby has neither problem).

Fixed:
- **F1000** (`DbSecurityGroup` in `multistate-app-dev.yaml`) — had no explicit
  egress rule, which defaults to allow-all-outbound. Added an explicit 443-only
  egress rule.
- **W77** (`DbMasterSecret`) — added `KmsKeyId: alias/aws/secretsmanager` so the
  secret's encryption key is explicit rather than the account default.
- **W28** (`DbInstance`) — dropped the explicit `DBInstanceIdentifier` so a future
  rename doesn't force a replacement.
- **W35** (`BootstrapBucket`, `MultistateArtifactsBucket`) — added a dedicated
  access-log destination bucket + `logging.s3.amazonaws.com` bucket policy for
  each hardened bucket (the modern replacement for the legacy
  `AccessControl: LogDeliveryWrite` ACL, which cfn-lint's `E3045`/`W3045` rules
  reject on a bucket with `OwnershipControls: BucketOwnerEnforced`).
- **W60** (`Vpc`) — added a VPC Flow Log to a CloudWatch Logs group with a
  dedicated IAM delivery role.
- **W84** (`FlowLogGroup`) — added a dedicated KMS key (`FlowLogKmsKey`) so the
  flow-log CloudWatch Logs group encrypts with a customer-managed key instead of
  the AWS-managed default.

Accepted as tradeoffs, listed in [`.cfn-nag-deny-list.yaml`](../.cfn-nag-deny-list.yaml)
so the CI job stays green without silently ignoring anything undocumented:
- **W28** on `CfnDeployRole` and `MultistateAppSecurityGroup` — both have explicit
  names (`RoleName: multistate-api-cfn-deploy`, `GroupName: multistate-${EnvName}-app-sg`)
  that other stacks/workflows depend on by exact name (the GitHub Actions OIDC trust
  policy references this role by name; `multistate-app-dev` could in principle
  `!ImportValue` the SG id instead, but the explicit name is intentional for
  console discoverability). Removing the name would let a future property-change
  update silently rename the resource instead of blocking, which is the tradeoff
  documented here rather than hidden.
- **W33** on the public subnets — `MapPublicIpOnLaunch: true` is the deliverable's
  own spec for what makes a subnet "public"; instances launched there need a public
  IP by design.
- **W5** on `MultistateAppSecurityGroup`'s and `DbSecurityGroup`'s egress —
  443-to-`0.0.0.0/0` is flagged because ECR, STS, and Secrets Manager don't have a
  single fixed IP range reachable without VPC endpoints, which are out of scope for
  this deliverable.
- **W35** on `AccessLogBucket` and `ArtifactAccessLogBucket` — these are the log
  *destination* buckets; enabling logging on a log bucket would create a
  self-referential logging loop. `BootstrapBucket` and `MultistateArtifactsBucket`
  (the buckets that actually hold application data) are not in this deny-list and
  are fixed for real, above.

`cfn-nag`'s deny-list is rule-ID-scoped, not resource-scoped, so `--deny-list-path`
suppresses a rule everywhere it would otherwise fire rather than per-resource; each
entry above is deliberately narrow enough that suppressing the rule id doesn't hide
an unrelated real finding.

## Shared-role trust-policy conflict (cohort account)

Mid-deliverable, a teammate's PR/session overwrote `multistate-api-cfn-deploy`'s trust
policy to scope `StringLike` on `token.actions.githubusercontent.com:sub` to only
their own repo, which silently locked every other cohort member's `validate-template`
CI job out of assuming the role (the `sub` claim no longer matched). Fixed by editing
the trust policy to list both repos' `ref:refs/heads/main` and `pull_request` entries
side by side, rather than either person overwriting the other's entry. This is a
structural risk of a shared IAM role across a cohort sharing one AWS account — worth
raising with the ES/instructor as a cohort-wide fix (e.g. one role per repo, or a
wildcard `sub` pattern scoped to the shared org) rather than each person patching the
list reactively when they get locked out.

## Secrets Manager over `NoEcho` parameters

`multistate-app-dev` resolves the RDS master password via a Secrets Manager dynamic
reference — `{{resolve:secretsmanager:multistate/dev/db-master:SecretString:password}}`
— rather than a `NoEcho: true` CloudFormation `Parameter`. A `NoEcho` parameter still
appears in plaintext in the CLI command that creates the ChangeSet, in shell history,
and in CI job logs unless every layer is scrubbed; the value is also visible to
anyone who can call `cloudformation:GetTemplateSummary` with parameter overrides in
some tooling paths. The dynamic reference means the password is never passed to the
`aws cloudformation` CLI at all — CloudFormation resolves it server-side at deploy
time directly against Secrets Manager, and the secret itself is generated in-place by
`AWS::SecretsManager::Secret`'s `GenerateSecretString`, so the plaintext value never
exists in the template, the CLI invocation, or CloudFormation's stored parameter
values.

## CI: cfn-lint + cfn-nag

[`.github/workflows/cfn-validate.yml`](../.github/workflows/cfn-validate.yml) runs on
every PR touching `cfn/`: `cfn-lint` (syntax/schema correctness), `cfn-nag`
(security-pattern scanning, `--fail-on-warnings`), and
`aws cloudformation validate-template` against each template. All three are required
status checks on `main`.

Local validation performed in this environment (no AWS credentials available here —
see the "Pending AWS-side verification" section below):

```
$ cfn-lint cfn/*.yaml
(no output — 0 errors, 0 warnings, exit code 0)
```

A `.cfnlintrc.yaml` at the repo root suppresses rule `W3691` for
`multistate-app-dev.yaml`. That rule fires because this cfn-lint release's bundled
RDS deprecated-engine-version dataset flags every PostgreSQL version it knows about
(16.x through the newest, 17.3) as deprecated — a stale/broken dataset in this cfn-lint
release, not a real problem with the template's chosen engine version. Documented here
rather than silently worked around so a future reader isn't confused by the suppression.

`cfn_nag_scan` was not run on the authoring machine (stale Ruby 2.6 plus a corporate
TLS-interception proxy blocked `gem install` locally). It ran for real in the
`cfn-validate.yml` CI job on the PR, which surfaced one real `FAIL` (F1000, a missing
egress rule) and eleven `WARN`s — all fixed except three intentionally accepted
tradeoffs. See "cfn-nag findings and accepted tradeoffs" below for the fix-by-fix
breakdown.

## cfn-author Claude Skill audit

The `cfn-author` Claude Skill was not run against a scratch branch in this environment
— it is invoked interactively (`/cfn-author multistate --region us-east-1`) and this
session has no AWS account configured to scaffold against. The four templates above
were hand-authored directly against the cohort's known Skill-quirk checklist instead
of being scaffolded and then corrected:

- **IRSA/OIDC trust policy `sub` claim** — used `StringLike` (not `StringEquals`) on
  `token.actions.githubusercontent.com:sub` in `multistate-bootstrap-dev.yaml`,
  because the claim value legitimately varies per workflow run (`ref:refs/heads/main`
  vs. `pull_request`) while the org/repo prefix stays fixed. `StringEquals` is used
  for the `aud` claim, which is a fixed constant for every GitHub Actions OIDC token.
- **DB master password** — resolved via the Secrets Manager dynamic reference in
  `multistate-app-dev.yaml`, never a `NoEcho: true` Parameter (see above).
- **S3 bucket deletion safety** — both `DeletionPolicy: Retain` and
  `UpdateReplacePolicy: Retain` set on every stateful resource (`BootstrapBucket`,
  `MultistateArtifactsBucket`, `DbMasterSecret`, `DbInstance`) — `DeletionPolicy` alone
  does not protect against a property-change replacement, only a stack delete.

## Pending AWS-side verification

This PR ships four templates that pass local `cfn-lint` validation and the
`cfn-validate.yml` CI workflow, but the following require a real AWS account and were
**not** performed in this environment (no AWS credentials configured):

- [ ] Deploy all four stacks via the ChangeSet flow; paste `describe-change-set` JSON
      diffs into the PR
- [ ] Confirm all four stacks reach `CREATE_COMPLETE`
- [ ] Attempt to delete `multistate-network-dev` while `multistate-app-dev` still
      imports its exports; confirm CloudFormation refuses with "Export ... is in use"
- [ ] Make a deliberate console edit (e.g. add a tag to the artefacts bucket); run
      `detect-stack-drift` + `describe-stack-resource-drifts`; confirm `DRIFTED`;
      revert; confirm `IN_SYNC`
- [ ] Run an `UPDATE` ChangeSet against `multistate-network-dev` (e.g. rename a tag);
      confirm `describe-change-set` shows `Replacement: False` on every modified
      resource
- [x] Run `cfn_nag_scan --input-path cfn/ --fail-on-warnings` for real — done via CI
- [ ] Run the `cfn-author` Claude Skill against a scratch branch and diff its output
      against these hand-authored templates
