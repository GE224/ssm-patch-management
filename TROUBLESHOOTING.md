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
