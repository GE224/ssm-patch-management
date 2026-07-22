# Troubleshooting: SSM Connection Lost After IAM Instance Profile Removal

## Scenario

As a deliberate break/debug exercise, the IAM instance profile (`SSM-EC2-Profile`)
was detached from `i-0c87cc8e8238ae7bc` (`SSM-EC2-Role`) to simulate a common
real-world failure: an instance that's healthy at the EC2 level but has silently
lost its ability to communicate with Systems Manager.

## Symptom

Attempting to start a new Session Manager session failed immediately:

```
SessionId: root-hel79chyfv4robyf327gekqs88 :
----------ERROR-------
Setting up data channel with id root-hel79chyfv4robyf327gekqs88 failed:
failed to create websocket for datachannel with error: CreateDataChannel
failed with no output or error: createDataChannel request failed: failed
to sign the request: failed to retrieve instance profile role credentials.
Err: EC2RoleRequestError: no EC2 instance role found
```

No commands were failing on an already-open session, no application errors,
no alarms — just this one specific failure the moment a new session was requested.

## Diagnostic steps

1. **Confirm the instance itself is healthy.**
   `describe-instances` / EC2 console showed the instance `running` with
   passing status checks. This rules out a crashed instance, OS-level hang,
   or network/security-group issue — the problem is scoped to SSM specifically.

2. **Check SSM's own view of the instance — and don't trust it blindly.**
   ```
   aws ssm describe-instance-information \
     --filters "Key=InstanceIds,Values=i-0c87cc8e8238ae7bc"
   ```
   `PingStatus` showed `Online` with a recent `LastPingDateTime`. This turned
   out to be a **false positive** — background heartbeat pings were still
   succeeding on a previously-cached, not-yet-expired credential, even though
   the instance could no longer authenticate for *new* actions like opening a
   session. Relying on `PingStatus` alone would have pointed the investigation
   in the wrong direction.

3. **Read the actual error text.**
   The error explicitly named the failure: `EC2RoleRequestError: no EC2
   instance role found`. This pointed directly at IAM credential retrieval,
   not network connectivity, not agent health, not DNS.

4. **Confirm the hypothesis.**
   ```
   aws ec2 describe-iam-instance-profile-associations \
     --filters "Name=instance-id,Values=i-0c87cc8e8238ae7bc"
   ```
   Returned no active association — the instance profile had in fact been
   detached. Root cause confirmed.

## Root cause

The IAM instance profile providing the instance's role (and therefore its
only path to short-lived credentials via IMDS) had been detached. The SSM
agent could not obtain fresh credentials to authenticate a new session,
even though existing cached credentials kept the heartbeat/ping mechanism
looking healthy for a period after the detachment.

## Fix

Re-attach the instance profile:

```
aws ec2 associate-iam-instance-profile \
  --instance-id i-0c87cc8e8238ae7bc \
  --iam-instance-profile Arn=arn:aws:iam::889168907206:instance-profile/SSM-EC2-Profile
```

## Verification

Verification was done by actually re-establishing a session, not by
re-checking `PingStatus` alone:

```
aws ssm start-session --target i-0c87cc8e8238ae7bc
```

Session connected successfully, confirming the fix.

## Key lessons

- **A "healthy" status metric isn't proof of a working capability.**
  `PingStatus: Online` remained true for several minutes into the outage
  because the heartbeat used already-cached credentials. The real signal
  came from attempting the actual operation (opening a session), not from
  the dashboard/status check. Always verify the capability you actually
  need, not just an adjacent health signal.
- **Detaching an instance profile does not revoke already-issued STS
  credentials.** Existing temporary credentials remain valid until their
  natural expiration; only *new* credential requests are blocked. This
  explains the delay between the break and the first observable failure.
- **Commands run inside the instance use the instance's own IAM role**,
  not the operator's local credentials. That role is intentionally scoped
  to only what the SSM agent needs (checking in, executing commands) —
  seeing `AccessDenied` for broad SSM control-plane calls (e.g.
  `DescribeInstanceInformation`) from inside the instance is expected,
  least-privilege behavior, not a bug.

---

# Troubleshooting: Bad Parameter on AWS-ConfigureAWSPackage

## Scenario

A second deliberate break/debug exercise, this time targeting command-level
failure instead of connectivity. Rather than breaking the instance's ability
to reach SSM, a `send-command` was issued against a healthy, fully-online
instance using `AWS-ConfigureAWSPackage` with an invalid `name` parameter
(`NotARealPackage123`) — simulating a common real mistake: a typo'd or
non-existent package reference in an otherwise valid command.

## Symptom

The first attempt was actually rejected before this scenario even started:

```
aws ssm send-command --document-name "AWS-RunPatchBaseline" \
  --parameters '{"Operation":["Foo"]}'

An error occurred (InvalidParameters) when calling the SendCommand operation:
Parameters provided in document are invalid or not supported.
```

`AWS-RunPatchBaseline`'s `Operation` parameter has a closed `AllowedValues`
list (`Scan`/`Install`) enforced by the document's JSON schema, so SSM's
control plane rejected the request before it ever reached the instance —
no CommandId was even created. This is a real and useful failure mode, but
a different one than intended: nothing to diagnose via command output,
since nothing was dispatched.

The scenario was switched to `AWS-ConfigureAWSPackage`, whose `name`
parameter is a free-form string — it passes schema validation and
actually dispatches to the instance:

```
aws ssm send-command \
  --instance-ids i-0c87cc8e8238ae7bc \
  --document-name "AWS-ConfigureAWSPackage" \
  --parameters '{"action":["Install"],"name":["NotARealPackage123"]}'
```

This returned a real CommandId, and `SSM PingStatus` stayed `Online` the
entire time — the instance itself was never at risk. Only this one command
invocation failed:

```
"Status": "Failed",
"StatusDetails": "Failed"
```

## Diagnostic steps

1. **Check `get-command-invocation` at the top level first — it wasn't enough.**
   With no `--plugin-name` specified, `PluginName`, `StandardOutputContent`,
   and `StandardErrorContent` all came back `null`. Multi-step documents
   don't surface plugin-level output at the top level.

2. **List the individual plugin steps and their output.**
   ```
   aws ssm list-command-invocations \
     --command-id 390370ed-0f5c-4492-ab03-33327942da75 \
     --instance-id i-0c87cc8e8238ae7bc \
     --details \
     --query "CommandInvocations[0].CommandPlugins[*].{Name:Name,Status:Status,Output:Output}"
   ```
   This surfaced the actual error, scoped to the `configurePackage` plugin:
   ```
   ----------ERROR-------
   failed to download manifest - failed to retrieve package document
   description: InvalidDocument: Document with name NotARealPackage123
   does not exist.
   ```

## Root cause

`AWS-ConfigureAWSPackage` does not search OS package managers (`yum`/`dnf`).
The `name` parameter refers to a package published in **SSM Distributor** —
AWS's own package catalog — and the document's first step is to fetch that
package's manifest (internally, a `GetDocument`-style lookup) before any
install logic runs on the instance. `NotARealPackage123` doesn't exist in
Distributor, so the manifest fetch failed immediately. The instance-side
agent never got as far as attempting an install — the failure happened at
the reference-resolution stage, not execution.

## Fix

Re-sent the same document with a real Distributor package name
(`AmazonCloudWatchAgent`):

```
aws ssm send-command \
  --instance-ids i-0c87cc8e8238ae7bc \
  --document-name "AWS-ConfigureAWSPackage" \
  --parameters '{"action":["Install"],"name":["AmazonCloudWatchAgent"]}'
```

## Verification

```
aws ssm list-command-invocations \
  --command-id 99c5e427-ccaa-4acf-a1b8-3f991c076f50 \
  --instance-id i-0c87cc8e8238ae7bc \
  --details \
  --query "CommandInvocations[0].CommandPlugins[*].{Name:Name,Status:Status,Output:Output}"
```

`configurePackage` returned `Status: Success`, with output confirming the
CloudWatch agent package was installed (`cwagent` user/group created,
`Successfully installed arn:aws:ssm:::package/AmazonCloudWatchAgent`).
Note: installing the package does not start or configure it — no agent
config was applied and no data was sent to CloudWatch, so this incurred
no charges.

## Key lessons

- **A bad parameter can be schema-valid but still reference something
  that doesn't exist.** `AWS-RunPatchBaseline`'s `Operation` param is a
  closed enum, so a bad value never left the control plane. `AWS-
  ConfigureAWSPackage`'s `name` param is just a string — AWS can't validate
  it against Distributor's catalog until the command actually runs, so the
  failure only shows up after dispatch, at execution time.
- **Top-level `get-command-invocation` output is unreliable for multi-step
  documents.** Always drop to `list-command-invocations --details` and
  read per-plugin `Output` when the top-level fields come back `null`.
- **This is a fundamentally different failure class than the IAM break.**
  There, the instance was unreachable and `PingStatus` lied about it.
  Here, `PingStatus` was accurate the whole time — the instance was
  completely healthy — and the fault was entirely within a single command's
  parameters. Same symptom category ("something with SSM failed"), two
  unrelated root causes and two different diagnostic paths.
