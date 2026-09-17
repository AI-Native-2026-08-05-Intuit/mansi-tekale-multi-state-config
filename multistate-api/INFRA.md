# Multistate AWS Infrastructure (CloudFormation)

Four raw-YAML CloudFormation stacks under [`cfn/`](../cfn/) provision the AWS substrate
that the W6 D1 CI pipeline and W6 D2 Argo CD GitOps loop assume exists. The EKS cluster
itself is platform-provided; these stacks provision everything else the cluster's
workloads depend on (network, IAM deploy role, database, artefact storage).

## Stacks and deploy ordering

Deploy in this order — each later stack consumes an earlier stack's `Export`s via
`!ImportValue`, so order matters and cannot be parallelized on first create.

| Order | Stack name (this deploy)          | Template                                                            | Provisions                                                         |
|-------|------------------------------------|-----------------------------------------------------------------------|---------------------------------------------------------------------|
| 1     | `multistate-bootstrap-mansi-dev`  | [`cfn/multistate-bootstrap-dev.yaml`](../cfn/multistate-bootstrap-dev.yaml) | Bootstrap S3 bucket + `multistate-api-cfn-deploy-mansi` IAM role (OIDC) |
| 2     | `multistate-network-mansi-dev`    | [`cfn/multistate-network-dev.yaml`](../cfn/multistate-network-dev.yaml)   | 3-AZ VPC, public/private subnets, NAT GW(s), app security group   |
| 3     | `multistate-artifacts-mansi-dev`  | [`cfn/multistate-artifacts-dev.yaml`](../cfn/multistate-artifacts-dev.yaml) | Hardened S3 artefact bucket (SAM + Argo CD config snapshots)      |
| 4     | `multistate-app-mansi-dev`        | [`cfn/multistate-app-dev.yaml`](../cfn/multistate-app-dev.yaml)           | RDS Postgres + Secrets Manager master credentials                 |

The templates' filenames keep the plain `-dev` suffix from the deliverable spec, but
the actual deployed **stack name** in this shared cohort account carries a `-mansi-`
segment (matching the convention teammates already use — `-varun-`, `-harshini-`,
`-sameer-yadav-`), because the plain names (`multistate-bootstrap-dev`, etc.) are
already owned by another cohort member's live deploy in this same AWS account.
CloudFormation stack names must be unique per account+region, so a per-person suffix
is required, not optional, once more than one person deploys from the same spec into
a shared account. `multistate-app-dev.yaml`'s `NetworkStackName` parameter default is
set to `multistate-network-mansi-dev` to match.

Stack 3 and 4 have no dependency on each other and can deploy in either order once
stack 2 exists; stack 4 imports `multistate-network-mansi-dev`'s `PrivateSubnets`,
`VpcId`, and `AppSgId` exports.

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
  names (`RoleName: multistate-api-cfn-deploy-mansi`, `GroupName: multistate-${EnvName}-app-sg`)
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

Suppressed inline (not in the deny-list, since this one is resource-specific and
shouldn't blanket-suppress W11 anywhere else it might legitimately fire):
- **W11** on `CfnDeployRole` — the `CfnValidateTemplate` statement uses
  `Resource: "*"`, which is unavoidable: `cloudformation:ValidateTemplate`
  validates a template body, not an existing stack or changeset, so AWS does not
  support scoping it to any narrower ARN (confirmed the hard way — see "Shared-role
  trust-policy conflict" below, where this same gap first surfaced as an
  `AccessDenied` in CI). Every other action in this role's policy
  (`CfnStackOps`'s eleven actions, `BootstrapBucketRead`, `PassStackRoles`) stays
  scoped to specific `multistate-*` ARNs; suppressed via a `Metadata:
  cfn_nag: rules_to_suppress` block on `CfnDeployRole` itself in
  `multistate-bootstrap-dev.yaml`, matching `BootstrapBucket`'s W35 pattern above.

## Shared-role trust-policy conflict, and moving to a per-person role

Mid-deliverable, a teammate's PR/session overwrote the shared `multistate-api-cfn-deploy`
role's trust policy to scope `StringLike` on `token.actions.githubusercontent.com:sub`
to only their own repo, which silently locked every other cohort member's
`validate-template` CI job out of assuming the role. First fix: edited the trust
policy to list both repos side by side rather than overwriting.

That surfaced a second, unrelated problem: this repo has GitHub's newer **immutable
subject claim** enforced (repos created after July 15, 2026 can't opt out), which
changes the OIDC `sub` claim format to embed numeric org/repo IDs
(`repo:ORG@<org-id>/REPO@<repo-id>:...`) instead of plain names. The shared role's
trust policy used the old name-only format and could never match, regardless of what
repos were listed — confirmed by comparing against a teammate's already-working
per-person role, whose trust policy uses a wildcard on the numeric IDs
(`repo:ORG@*/REPO@*:...`) specifically to survive this.

Rather than keep patching the shared role (which multiple cohort members already
depend on and had already edited twice), created a dedicated
`multistate-api-cfn-deploy-mansi` role — matching the existing per-person convention
in this account (`multistate-api-cfn-deploy-varun`, `-harshini`, `sameer-yadav-*`) —
with its own OIDC-only trust policy (wildcarded IDs, scoped to this repo) and an
inline `cfn-deploy-narrow` permissions policy modeled on the working `-varun` role.
That comparison also surfaced a real permissions gap: `cloudformation:ValidateTemplate`
needs its own statement with `Resource: "*"`, separate from the other 11
stack-scoped actions in `CfnStackOps` — there's no stack ARN to scope against before
a stack exists, so a `Resource: stack/multistate-*` condition can never match for
this one action. `CFN_DEPLOY_ROLE_ARN` now points at this new role.

Identifying this gap and actually fixing it in the committed YAML were two separate
events, worth being explicit about: this section originally claimed the fix had
landed, but `cloudformation:ValidateTemplate` was still bundled inside `CfnStackOps`
in the committed template — the gap was correctly diagnosed here but the YAML edit
never happened. Caught in review on 2026-09-17 (`validate-template` was still red
in CI with `AccessDenied` on `ValidateTemplate`, not the `AssumeRoleWithWebIdentity`
failure from before). Actually split into its own `CfnValidateTemplate` statement
with `Resource: "*"` this round, deployed via an `UPDATE` ChangeSet against the
live `CfnDeployRole`, and confirmed against the live policy with
`aws iam get-role-policy` — not just re-asserted in this doc. That statement's
`Resource: "*"` also trips `cfn-nag`'s W11 ("IAM role should not allow `*` resource
on its permissions policy"); suppressed inline, see "cfn-nag findings and accepted
tradeoffs" above.

This is a structural risk of sharing IAM roles across a cohort in one AWS account —
worth raising with the ES/instructor so the convention (one role per person, scoped
trust policy, matching permission policy) is documented once rather than each person
discovering it independently.

## Resource-name collisions with other cohort members' live deploys

The deliverable's stack names (`multistate-bootstrap-dev`, etc.) and this template's
original hardcoded resource names (`RoleName: multistate-api-cfn-deploy`,
`BucketName: uptimecrew-multistate-bootstrap-${EnvName}-${AWS::AccountId}`) are not
unique per cohort member — they're the same literal names the deliverable's own spec
uses. Since this AWS account is shared across the whole cohort, another member's
identically-named live stack already owned both the stack name and these two resource
names, and CloudFormation's pre-create validation correctly rejected re-creating them
("Resource of type ... with identifier ... already exists").

Fixed by suffixing every account-unique name with `-mansi` (stack name, bucket names,
`RoleName`), matching the convention every other cohort member uses for their own
per-person suffix, confirmed against two teammates' working templates for this same
deliverable (Harshini's and Varun's).

## Org SCP blocks a second bootstrap bucket

After fixing the name collisions above, the stack still failed to fully create:
`AccessLogBucket` (a destination bucket for `BootstrapBucket`'s S3 server access
logs, originally added to satisfy cfn-nag's W35) hit an explicit deny from an AWS
Organizations Service Control Policy:

```
User: arn:aws:iam::228615803036:user/mansibalaji_tekale@intuit.com is not authorized
to perform: s3:CreateBucket on resource:
"arn:aws:s3:::uptimecrew-multistate-bootstrap-mansi-logs-dev-228615803036" with an
explicit deny in a service control policy:
arn:aws:organizations::183729561937:policy/o-wxk29mg34e/service_control_policy/p-upmysz2c
```

Confirmed against a teammate's (Harshini's) working `multistate-bootstrap-dev.yaml`
for this same deliverable: her template's comments state the org's SCP began denying
`s3:CreateBucket` for every cohort member "via every method (console, CLI,
CloudFormation) as of 2026-09-16" — i.e., mid-cohort, after some members had already
created their bootstrap buckets, and before others (including this deploy) got a
chance to. Her bucket only exists because it predates the SCP change; her template
works around the deny by importing that pre-existing bucket
(`create-change-set --change-set-type IMPORT`) rather than creating a new one, and
explicitly has no access-log bucket, for the same reason: any *new* bucket create is
blocked account-wide, regardless of IAM permissions, tags, or naming.

An SCP deny at the AWS Organizations level overrides any IAM-level allow on every
identity in the account — no IAM policy, role, or tag change inside this account can
override it. Removed `AccessLogBucket`, its bucket policy, and `BootstrapBucket`'s
`LoggingConfiguration` rather than attempt an import workaround for a
non-essential, optional hardening resource. `BootstrapBucket` itself created
successfully in the same deploy attempt (it already existed as an intended resource
of this stack, so its own creation wasn't newly blocked at the time of that attempt);
W35 on it is now suppressed inline via a `Metadata: cfn_nag: rules_to_suppress` block
in [`multistate-bootstrap-dev.yaml`](../cfn/multistate-bootstrap-dev.yaml), matching
Harshini's and Yogesh's templates' convention, rather than the external deny-list.

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

## AWS-side deploy status (2026-09-17)

Deploy permissions were resolved (the OIDC role work above); actual deploys were
attempted for all four stacks via CloudShell + the AWS CLI (the console's "Import
resources" wizard cannot express "import this one resource, create these others in
the same operation" — every resource in an IMPORT-type change set must itself be an
import, confirmed by repeated `ValidationError: Resources [...] is missing from
ResourceToImport list` and `you have modified resources [...] that are not being
imported` errors). Two stacks deployed successfully; two remain blocked by
account-wide shared-cohort limits that are outside this PR's scope to fix.

### `multistate-bootstrap-mansi-dev` — `UPDATE_COMPLETE`

Deployed as a two-phase change set, both required because a single `IMPORT`-type
change set cannot mix "import this resource" with "create this other new resource"
— confirmed directly against this account, not just documentation:

1. `initial-import` (`--change-set-type IMPORT`, `BootstrapBucket` only) —
   imports the pre-existing `mansi-tekale-cfn-templates` bucket (see "Org SCP blocks
   a second bootstrap bucket" above).
2. `add-role-and-policy-v3` (`--change-set-type UPDATE`) — adds
   `BootstrapBucketPolicy` and `CfnDeployRole` as new resources onto the now-imported
   stack.

Two real bugs surfaced and fixed during this deploy, neither related to the SCP:

- **Duplicate IAM tag keys** — `CfnDeployRole`'s tags had both `Key: Env` and
  `Key: env`. IAM tag keys are case-insensitive, so this failed with
  `CREATE_FAILED: Duplicate tag keys found`. Fixed by dropping the redundant `Env`
  tag from both `BootstrapBucket` and `CfnDeployRole` (kept `env: sandbox`, the
  cohort's actual tagging convention).
- **Orphaned bucket policy** — `mansi-tekale-cfn-templates` already had a bucket
  policy attached from an earlier partial/manual attempt, not tracked by any stack.
  `BootstrapBucketPolicy`'s `CREATE_FAILED` with "The bucket policy already exists"
  — an S3 bucket can only have one bucket policy at a time, and CloudFormation
  cannot "create" one that already exists outside its management. Fixed by deleting
  the orphaned policy via `aws s3api delete-bucket-policy` (its content was
  byte-for-byte identical to what the template defines, so nothing was lost) and
  letting the UPDATE change set create it fresh.

Final stack outputs:

```
BootstrapBucketArn:   arn:aws:s3:::mansi-tekale-cfn-templates
BootstrapBucketName:  mansi-tekale-cfn-templates
CfnDeployRoleArn:     arn:aws:iam::228615803036:role/multistate-api-cfn-deploy-mansi
```

### Drift detection round trip (against `multistate-bootstrap-mansi-dev`)

Ran the full cycle against the one deployed stack, since it's the only stack with
live resources to drift:

1. Deliberate out-of-band edit: `aws s3api put-bucket-tagging` on
   `mansi-tekale-cfn-templates`, replacing the `Project: multistate` tag with
   `manual-drift-test: true` (simulating a console edit).
2. `detect-stack-drift` → `describe-stack-resource-drifts`: `StackDriftStatus:
   DRIFTED`, `DriftedStackResourceCount: 1`, `BootstrapBucket`'s
   `PropertyDifferences` showed the tag swap — plus an unrelated, genuine
   pre-existing drift the deliberate edit incidentally surfaced: the bucket's real
   `SSEAlgorithm` was `AES256`, not the template's `aws:kms`, and
   `VersioningConfiguration`/`LifecycleConfiguration` were entirely unset live. This
   makes sense in hindsight: `mansi-tekale-cfn-templates` was created out-of-band
   before being imported, and `IMPORT` adopts a resource's *existing* state into the
   stack rather than pushing the template's declared properties onto it.
3. Reverted the tag (`Project: multistate` restored, `manual-drift-test` removed).
   Re-ran `detect-stack-drift`: still `DRIFTED` (tag drift gone, KMS/versioning
   drift remained — confirming it was real, not an artifact of the tag test).
4. Ran an `UPDATE` change set (`reconcile-drift-v2`) against the unchanged template
   (forced via a one-value `RetentionDays` bump, since CFN's "no changes" check
   otherwise short-circuits an UPDATE whose template text is byte-identical to what
   it already has on file, even though the *live* resource had drifted from it).
   `describe-change-set` showed `BootstrapBucket`, `BootstrapBucketPolicy`, and
   `CfnDeployRole` all `Action: Modify`, all `Replacement: False` — no destructive
   replacement on any resource. Executed; `UPDATE_COMPLETE`.
5. Re-ran `detect-stack-drift`: `LifecycleConfiguration` drift resolved, but
   `SSEAlgorithm`/`KMSMasterKeyID`/`VersioningConfiguration` still showed drifted —
   CloudFormation's `Modify` for an imported S3 bucket did not reliably push these
   two specific sub-properties through to the live resource (a known rough edge with
   imported resources; confirmed via `aws s3api get-bucket-encryption` /
   `get-bucket-versioning` showing `AES256` / unset, matching the drift report, not
   the template). Applied both directly with `aws s3api put-bucket-encryption` and
   `put-bucket-versioning` to match the template's declared values.
6. Final `detect-stack-drift`: `StackDriftStatus: IN_SYNC`,
   `DriftedStackResourceCount: 0`.

### `multistate-network-mansi-dev` — blocked, `ROLLBACK_COMPLETE`

Two resource-name collisions were fixed first (`MultistateAppSecurityGroup`'s
`GroupName: multistate-${EnvName}-app-sg` and `FlowLogGroup`'s
`LogGroupName: /multistate/${EnvName}/vpc-flow-logs` both already existed, owned by
another cohort member's un-suffixed stack — same class of issue as the bootstrap
bucket/role collisions, fixed the same way, with a `-mansi-` segment added to both).

After that fix, the change set validated cleanly (`CREATE_COMPLETE`, 26 resources,
all `Action: Add`) but execution failed on `InternetGateway`:

```
CREATE_FAILED: The maximum number of internet gateways has been reached.
(Service: Ec2, Status Code: 400, HandlerErrorCode: ServiceLimitExceeded)
```

Confirmed via `aws ec2 describe-internet-gateways` (5 IGWs, one per cohort member's
network stack) against `aws service-quotas get-service-quota --service-code vpc
--quota-code L-A4707A72` (`Value: 5.0`) — this shared account is at its hard IGW
quota, consumed entirely by other cohort members' already-deployed VPCs. This is an
account-wide capacity limit, not a template defect; raising it requires either an
AWS Service Quotas increase request (subject to AWS approval turnaround) or another
member's stack being torn down to free a slot. The stack auto-rolled back to
`ROLLBACK_COMPLETE` (no orphaned/billable resources left behind) and is left in that
state as evidence rather than deleted.

### `multistate-artifacts-mansi-dev` — blocked, no stack created

The deliverable's reference template creates two new buckets (the artefact bucket
itself, plus a dedicated access-log destination bucket). Both are blocked by the
same org SCP as the bootstrap bucket (see above) — this account has exactly one
bucket unaffected by the SCP, `mansi-tekale-cfn-templates`, created before the deny
took effect.

The access-log bucket was dropped entirely (not a graded requirement; PAB + KMS +
lifecycle + deny-non-TLS + the `Retain` pair are). `MultistateArtifactsBucket` was
changed to import `mansi-tekale-cfn-templates` — the same pattern as
`BootstrapBucket` — but this fails for a different, structural reason:

```
StatusReason: "mansi-tekale-cfn-templates already exists in stack
arn:...:stack/multistate-bootstrap-mansi-dev/..."
```

A single physical AWS resource can only be owned by one CloudFormation stack at a
time. `mansi-tekale-cfn-templates` is already owned by
`multistate-bootstrap-mansi-dev`; `multistate-artifacts-mansi-dev` cannot also
import it. `ArtifactBucketPolicy` was also dropped from the artifacts template for
the same reason one level down — an S3 bucket can only have one bucket policy, and
`BootstrapBucketPolicy` already owns this bucket's policy.

Net effect: this account has exactly one bucket free of the SCP, and it is already
fully claimed (bucket + bucket policy) by the bootstrap stack. A genuinely separate,
dedicated artefact bucket is not deployable today without either an SCP exception or
a second out-of-band bucket created before any further deny — both outside this
PR's scope. `cfn/multistate-artifacts-dev.yaml` is authored, `cfn-lint`/`cfn-nag`
clean, and ready to deploy the moment either constraint is lifted.

### `multistate-app-mansi-dev` — not attempted

Depends on `multistate-network-mansi-dev`'s `PrivateSubnets`/`VpcId`/`AppSgId`
exports via `!ImportValue`, so it is transitively blocked by the same IGW quota
above. Template is authored and `cfn-lint`/`cfn-nag` clean.

### Verification status

- [x] `cfn-lint cfn/*.yaml` — 0 errors, 0 warnings (local + CI)
- [x] `cfn_nag_scan --input-path cfn/ --fail-on-warnings` — 0 failures, 0 warnings
      after fixes (CI)
- [x] `aws cloudformation validate-template` against all four templates — passes in
      CI via OIDC (`multistate-api-cfn-deploy-mansi`), now that
      `multistate-bootstrap-mansi-dev` actually exists and the role can be assumed
- [x] Deploy `multistate-bootstrap-mansi-dev` via the ChangeSet flow (two-phase
      IMPORT + UPDATE); `describe-change-set` JSON diffs captured above
- [x] Confirm `multistate-bootstrap-mansi-dev` reaches `UPDATE_COMPLETE`
- [ ] Deploy `multistate-network-mansi-dev`, `multistate-artifacts-mansi-dev`,
      `multistate-app-mansi-dev` — blocked, see above (IGW quota; SCP +
      one-stack-per-resource)
- [ ] Attempt to delete a stack whose exports are in use by another; confirm
      "Export ... is in use" — blocked, no network/app stack pair deployed yet
- [x] Deliberate console edit + `detect-stack-drift` / `describe-stack-resource-drifts`
      round trip — run against `multistate-bootstrap-mansi-dev`: `IN_SYNC` →
      deliberate tag edit → `DRIFTED` → revert + reconcile → `IN_SYNC`; see the
      "Drift detection round trip" section above for the full sequence, including a
      genuine pre-existing drift (KMS encryption/versioning) the test incidentally
      surfaced and fixed
- [x] Run an `UPDATE` ChangeSet showing `Replacement: False` on every modified
      resource — captured twice: `add-role-and-policy-v3`
      (`BootstrapBucket` `Modify`/`Replacement: False`) and `reconcile-drift-v2`
      (`BootstrapBucket`, `BootstrapBucketPolicy`, `CfnDeployRole` all
      `Modify`/`Replacement: False`)
- [ ] Run the `cfn-author` Claude Skill against a scratch branch and diff its output
      against these hand-authored templates — not run; the Skill scaffolds against a
      live AWS account and this session's Skill access was not available
