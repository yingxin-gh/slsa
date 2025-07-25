---
title: Threats & mitigations
description: A comprehensive technical analysis of supply chain threats and their corresponding mitigations in SLSA.
---

What follows is a comprehensive technical analysis of supply chain threats and
their corresponding mitigations with SLSA and other best practices. For an
introduction to the supply chain threats that SLSA is aiming to protect
against, see [Supply chain threats].

The examples on this page are meant to:

-   Explain the reasons for each of the SLSA [build](build-requirements.md) and
    [source](source-requirements.md) requirements.
-   Increase confidence that the SLSA requirements are sufficient to achieve the
    desired [level](about#how-slsa-works) of integrity protection.
-   Help implementers better understand what they are protecting against so that
    they can better design and implement controls.

<!--
**TODO:** Expand this threat model to also cover "unknowns". Not sure if that is
a "threat" or a "risk". Example: If libFoo is compromised, how do you know if
you are compromised? At a first level, if you don't even know whether you
include libFoo or not, that's a big risk. But even then, it might be that you
don't use libFoo in a way that makes your product vulnerable. We should capture
that somehow. This isn't specific to dependencies - it applies to the entire
diagram.
([discussion](https://github.com/slsa-framework/slsa/pull/1046/files/ebf34a8f9e874b219f152bad62673eae0b3ba2c3#r1585440922))
-->

<article class="threats">

## Overview

![Supply Chain Threats](images/supply-chain-threats.svg)

This threat model covers the *software supply chain*, meaning the process by
which software is produced and consumed. We describe and cluster threats based
on where in the software development pipeline those threats occur, labeled (A)
through (I). This is useful because priorities and mitigations mostly cluster
along those same lines. Keep in mind that dependencies are
[highly recursive](#dependency-threats), so each dependency has its own threats
(A) through (I), and the same for *their* dependencies, and so on. For a more
detailed explanation of the supply chain model, see
[Terminology](terminology.md).

Importantly, producers and consumers face *aggregate* risk across all of the
software they produce and consume, respectively. Many organizations produce
and/or consume thousands of software packages, both first- and third-party, and
it is not practical to rely on every individual team in the organization to do
the right thing. For this reason, SLSA prioritizes mitigations that can be
broadly adopted in an automated fashion, minimizing the chance of mistakes.

## Source threats

A source integrity threat is a potential for an adversary to introduce a change
to the source code that does not reflect the intent of the software producer.
This includes modification of the source data at rest as well as insider threats,
when an authorized individual introduces an unauthorized change.

The SLSA Source track mitigates these threats when the consumer
[verifies source](verifying-source.md) against expectations, confirming
that the revision they received was created in the expected manner.

### (A) Producer

The producer of the software intentionally produces code that harms the
consumer, or the producer otherwise uses practices that are not deserving of the
consumer's trust.

<details><summary>Software producer intentionally creates a malicious revision of the source</summary>

*Threat:* A producer intentionally creates a malicious revision with the intent of harming their consumers.

*Mitigation:*
This kind of attack cannot be directly mitigated through SLSA controls.
Consumers must establish some basis to trust the organizations from which they consume software.
That basis may be:

-   The repo is open source with an active user-base. High numbers of engaged users may increase the likelihood that bad code is detected during code review and reduce the time-to-detection when bad revisions are accepted.
-   The organization has sufficient legal or reputational incentives to dissuade it from making malicious changes.

Ultimately this is a judgement call with no straightforward answer.

*Example:* A producer with an otherwise good reputation decides suddenly to produce a malicious artifact with the intent to harm their consumers.

</details>

### (B) Modifying the source

An adversary without any special administrator privileges attempts to introduce a change counter to the declared intent of the source by following the producer's official source control process.

Threats in this category can be mitigated by following source control management best practices.

#### (B1) Submit change without review

<details><summary>Directly submit without review<span>(Source L4)</span></summary>

*Threat:* Malicious code submitted to the source repository.

*Mitigation:* Require approval of all changes before they are accepted.

*Example:* Adversary directly pushes a change to a git repo's `main` branch.
Solution: The Source Control System is configured to require two party review for
contributions to the `main` branch.

</details>
<details><summary>Single actor controls multiple accounts</summary>

*Threat:* An actor is able to control multiple account and effectively approve their own code changes.

*Mitigation:* The producer must ensure that no actor is able to control or influence multiple accounts with review privileges.

*Example:* Adversary creates a pull request using a secondary account and approves it using their primary account.
Solution: The producer must track all actors who have both explicit review permissions and the independent ability to control
a privileged bot. A common vector for this attack is to influence a robot account with the permission to review or contribute
code. Control of the robot account and an actor's own personal account is enough to exploit this vulnerability. A common
solution to this flow is to deny bot accounts from contributing or reviewing code, or to require more human reviews in those
cases.

</details>
<details><summary>Use a robot account to submit change</summary>

*Threat:* Exploit a robot account that has the ability to submit changes without
two-person review.

*Mitigation:* All changes require review by two people, even changes authored by
robots.

*Example:* A file within the source repository is automatically generated by a
robot, which is allowed to submit without review.
Adversary compromises the robot and submits a malicious change.
Solution: Require two-person review for such changes, ignoring the robot.

</details>
<details><summary>Abuse of rule exceptions</summary>

*Threat:* Rule exceptions provide vector for abuse.

*Mitigation:* Remove rule exceptions.

*Example:* A producer intends to require two-person review on "all changes except for documentation changes," defined as those only modifying `.md` files.
Adversary submits a malicious executable masquerading as a documentation file, `help.md`.
This avoids the two-person review rule due to the exception.
In the future, a user (or another workflow) can be induced to *execute* `help.md` and become compromised.
Technically the malicious code change met all defined policies yet the intent of the organization was defeated.
Solution: The producer adjusts the rules to prohibit such exceptions.

</details>

<details><summary>Highly-permissioned actor bypasses or disables controls<span>(verification)</span></summary>

*Threat:* Trusted actor with "admin" privileges in a repository submits code by disabling existing controls.

*Mitigation:* The Source Control System must have controls in place to prevent
and detect abusive behavior from administrators (e.g. two-person approvals,
audit logging).

*Example:* GitHub repository-level admin removes a branch protection requirement, pushes
their change, then re-enables the requirement to cover their tracks.
Solution: Consumers do not accept claims from the Source Control System unless
they trust sufficient controls are in place to prevent repo admins from
abusing privileges.

</details>

#### (B2) Evade change management process

<details><summary>Alter change history<span>(Source L2+)</span></summary>

*Threat:* Adversary alters branch history to hide malicious activity.

*Mitigation:* The Source Control System prevents branch history from being
altered.

*Example:* Adversary submits a malicious commit `X` to the `main` branch. A
release is built and published from `X`. The adversary then "force pushes"
to `main` erasing the record of the malicious commit.  Solution: The Source
Control System is configured to prevent force pushes to `main`.

</details>
<details><summary>Replace tagged content with malicious content<span>(Source L2+)</span></summary>

*Threat:* Adversary alters a tag to point at malicious content.

*Mitigation:* The Source Control System does not allow protected tags to be updated.

*Example:* Adversary crafts a malicious commit `X` on a development branch which
does enforce any controls. They then update the `release_1.2` tag to point to
`X`. Consumers of `release_1.2` will get the malicious revision. Solution: The
Source Control System does not allow protected tags to be updated.

</details>
<details><summary>Skip required checks<span>(Source L2+)</span></summary>

*Threat:* Code is submitted without following the producers documented
development process, introducing unintended behavior.

*Mitigation:* The producer uses the Source Control System to implement technical
controls ensuring adherence to the development process.

*Example:* An engineer submits a new feature that has a critical flaw on an
untested code path, in violation of the producer's documented process of having
high test coverage. Solution: The producer implements a technical control in the
SCS that requires 95%+ test coverage.

</details>
<details><summary>Modify code after review<span>(Source L4)</span></summary>

*Threat:* Modify the code after it has been reviewed but before submission.

*Mitigation:* The Source Control System invalidates approvals whenever the proposed change is modified.

*Example:* Source repository requires two-person review on all changes.
Adversary sends an initial "good" pull request to a peer, who approves it.
Adversary then modifies their proposal to contain "bad" code.

Solution: Configure the code review rules to require review of the most recent revision before submission.

</details>
<details><summary>Submit a change that is unreviewable<span>(Source L4)</span></summary>

*Threat:* Adversary crafts a change that looks benign to a reviewer but is actually malicious.

*Mitigation:* Code review system ensures that all reviews are informed and
meaningful to the extent possible. For example the system could show
& resolve symlinks, render images, or verify & display provenance.

*Example:* A proposed change updates a JPEG file to include a malicious
message, but the reviewer is only presented with a diff of the binary
file contents. The reviewer is unable to parse the contents themselves
so they do not have enough context to provide a meaningful review.
Solution: the code review system should present the reviewer with a
rendering of the image and the [embedded
metadata](https://en.wikipedia.org/wiki/Exif), allowing them to make an
informed decision.

</details>
<details><summary>Copy a reviewed change to another context<span>(Source L4)</span></summary>

*Threat:* Get a change reviewed in one context and then transfer it to a
different context.

*Mitigation:* Approvals are context-specific.

*Example:* MyPackage's source repository requires two-person review. Adversary
forks the repo, submits a change in the fork with review from a colluding
colleague (who is not trusted by MyPackage), then proposes the change to
the upstream repo.
Solution: The proposed change still requires two-person review in the upstream
context even though it received two-person review in another context.

</details>

</details>
<details><summary>Commit graph attacks</summary>

*Threat:* A malicious commit can be included in a sequence of commits such that it does not appear malicious in the net change presented to reviewers.

*Mitigation:* The producer ensures that all revisions in the protected context followed the same contribution process.

*Example:* Adversary sends a pull request containing malicious commit X and a commit Y that undoes X.
The combined change of X + Y displays zero lines of malicious code and the reviewer cannot tell that X is malicious unless they review it individually.
If X is allowed to become reachable from the protected branch, the content may become available in secured environments such as developer machines.

Solution: Each revision in the protected context must have followed the intended process.
Ultimately, this means that either each code review results in at most a single new commit or that the full process is followed for each constituent commit in a proposed sequence.

</details>

#### (B3) Render code review ineffective

<details><summary>Collude with another trusted person</summary>

*Threat:* Two trusted persons collude to author and approve a bad change.

*Mitigation:* This threat is not currently addressed by SLSA, but the producer can arbitrarily increase friction of their policies to reduce risk, such as requiring additional, or more senior reviewers.
The goal of policy here is to ensure that the approved changes match the intention of the producer for the source.
Increasing the friction of the policies may make it harder to circumvent, but doing so has diminishing returns.
Ultimately the producer will need to land upon a balanced risk profile that makes sense for their security posture.

</details>
<details><summary>Trick reviewer into approving bad code</summary>

*Threat:* Construct a change that looks benign but is actually malicious, a.k.a.
"bugdoor."

*Mitigation:* This threat is not currently addressed by SLSA.

</details>
<details><summary>Reviewer blindly approves changes</summary>

*Threat:* Reviewer approves changes without actually reviewing, a.k.a. "rubber
stamping."

*Mitigation:* This threat is not currently addressed by SLSA.

</details>

#### (B4) Render change metadata ineffective

<details><summary>Forge change metadata<span>(Source L2+)</span></summary>

*Threat:* Forge the change metadata to alter attribution, timestamp, or
discoverability of a change.

*Mitigation:* The Source Control System only attributes changes to authenticated
identities and, at Source L3+, records contemporaneous evidence of changes in
signed source provenance attestations.

*Example:* Adversary 'X' creates a commit with unauthenticated metadata claiming
it was authored by 'Y'. Solution: The Source Control System records the identity
of 'X' when 'X' submits the commit to the repository.

</details>

### (C) Source code management

An adversary introduces a change to the source control repository through an
administrative interface, or through a compromise of the underlying
infrastructure.

<details><summary>Platform admin abuses privileges<span>(verification)</span></summary>

*Threat:* Platform administrator abuses their privileges to bypass controls or
to push a malicious version of the software.

*Mitigation:* The source platform must have controls in place to prevent and
detect abusive behavior from administrators (e.g. two-person approvals for
changes to the infrastructure, audit logging). A future [Platform
Operations Track](future-directions#platform-operations-track) may provide
more specific guidance on how to secure the underlying platform.

*Example 1:* GitHostingService employee uses an internal tool to push changes to
the MyPackage source repo.

*Example 2:* GitHostingService employee uses an internal tool to push a
malicious version of the server to serve malicious versions of MyPackage sources
to a specific CI/CD client but the regular version to everyone else, in order to
hide tracks.

*Example 3:* GitHostingService employee uses an internal tool to push a
malicious version of the server that includes a backdoor allowing specific users
to bypass branch protections. Adversary then uses this backdoor to submit a
change to MyPackage without review.

*Solution:* Consumers do not accept claims from the Source Control System unless
they trust sufficient controls are in place to prevent repo admins from
abusing privileges.

</details>
<details><summary>Exploit vulnerability in SCM</summary>

*Threat:* Exploit a vulnerability in the implementation of the source code
management system to bypass controls.

*Mitigation:* This threat is not currently addressed by SLSA.

</details>

## Build threats

A build integrity threat is a potential for an adversary to introduce behavior
to an artifact without changing its source code, or to build from a
source, dependency, and/or process that is not intended by the software
producer.

The SLSA Build track mitigates these threats when the consumer
[verifies artifacts](verifying-artifacts.md) against expectations, confirming
that the artifact they received was built in the expected manner.

### (D) External build parameters

An adversary builds from a version of the source code that does not match the
official source control repository, or changes the build parameters to inject
behavior that was not intended by the official source.

The mitigation here is to compare the provenance against expectations for the
package, which depends on SLSA Build L1 for provenance. (Threats against the
provenance itself are covered by (E) and (F).)

<details><summary>Build from unofficial fork of code <span>(expectations)</span></summary>

*Threat:* Build using the expected CI/CD process but from an unofficial fork of
the code that may contain unauthorized changes.

*Mitigation:* Verifier requires the provenance's source location to match an
expected value.

*Example:* MyPackage is supposed to be built from GitHub repo `good/my-package`.
Instead, it is built from `evilfork/my-package`. Solution: Verifier rejects
because the source location does not match.

</details>
<details><summary>Build from unofficial branch or tag <span>(expectations)</span></summary>

*Threat:* Build using the expected CI/CD process and source location, but
checking out an "experimental" branch or similar that may contain code not
intended for release.

*Mitigation:* Verifier requires that the provenance's source branch/tag matches
an expected value, or that the source revision is reachable from an expected
branch.

*Example:* MyPackage's releases are tagged from the `main` branch, which has
branch protections. Adversary builds from the unprotected `experimental` branch
containing unofficial changes. Solution: Verifier rejects because the source
revision is not reachable from `main`.

</details>
<details><summary>Build from unofficial build steps <span>(expectations)</span></summary>

*Threat:* Build the package using the proper CI/CD platform but with unofficial
build steps.

*Mitigation:* Verifier requires that the provenance's build configuration source
matches an expected value.

*Example:* MyPackage is expected to be built by Google Cloud Build using the
build steps defined in the source's `cloudbuild.yaml` file. Adversary builds
with Google Cloud Build, but using custom build steps provided over RPC.
Solution: Verifier rejects because the build steps did not come from the
expected source.

</details>
<details><summary>Build from unofficial parameters <span>(expectations)</span></summary>

*Threat:* Build using the expected CI/CD process, source location, and
branch/tag, but using a parameter that injects unofficial behavior.

*Mitigation:* Verifier requires that the provenance's external parameters all
match expected values.

*Example 1:* MyPackage is supposed to be built from the `release.yml` workflow.
Adversary builds from the `debug.yml` workflow. Solution: Verifier rejects
because the workflow parameter does not match the expected value.

*Example 2:* MyPackage's GitHub Actions Workflow uses `github.event.inputs` to
allow users to specify custom compiler flags per invocation. Adversary sets a
compiler flag that overrides a macro to inject malicious behavior into the
output binary. Solution: Verifier rejects because the `inputs` parameter was not
expected.

</details>
<details><summary>Build from modified version of code modified after checkout <span>(expectations)</span></summary>

*Threat:* Build from a version of the code that includes modifications after
checkout.

*Mitigation:* Build platform pulls directly from the source repository and
accurately records the source location in provenance.

*Example:* Adversary fetches from MyPackage's source repo, makes a local commit,
then requests a build from that local commit. Builder records the fact that it
did not pull from the official source repo. Solution: Verifier rejects because
the source repo does not match the expected value.

</details>

### (E) Build process

An adversary introduces an unauthorized change to a build output through
tampering of the build process; or introduces false information into the
provenance.

These threats are directly addressed by the SLSA Build track.

<details><summary>Forge values of the provenance (other than output digest) <span>(Build L2+)</span></summary>

*Threat:* Generate false provenance and get the trusted control plane to sign
it.

*Mitigation:* At Build L2+, the trusted control plane [generates][authentic] all
information that goes in the provenance, except (optionally) the output artifact
hash. At Build L3+, this is [hardened][unforgeable] to prevent compromise even
by determined adversaries.

*Example 1 (Build L2):* Provenance is generated on the build worker, which the
adversary has control over. Adversary uses a malicious process to get the build
platform to claim that it was built from source repo `good/my-package` when it
was really built from `evil/my-package`. Solution: Builder generates and signs
the provenance in the trusted control plane; the worker reports the output
artifacts but otherwise has no influence over the provenance.

*Example 2 (Build L3):* Provenance is generated in the trusted control plane,
but workers can break out of the container to access the signing material.
Solution: Builder is hardened to provide strong isolation against tenant
projects.

</details>
<details id="forged-digest"><summary>Forge output digest of the provenance <span>(n/a)</span></summary>

*Threat:* The tenant-controlled build process sets output artifact digest
(`subject` in SLSA Provenance) without the trusted control plane verifying that
such an artifact was actually produced.

*Mitigation:* None; this is not a problem. Any build claiming to produce a given
artifact could have actually produced it by copying it verbatim from input to
output.[^preimage] (Reminder: Provenance is only a claim that a particular
artifact was *built*, not that it was *published* to a particular registry.)

*Example:* A legitimate MyPackage artifact has digest `abcdef` and is built
from source repo `good/my-package`. A malicious build from source repo
`evil/my-package` claims that it built artifact `abcdef` when it did not.
Solution: Verifier rejects because the source location does not match; the
forged digest is irrelevant.

[^preimage]: Technically this requires the artifact to be known to the
    adversary. If they only know the digest but not the actual contents, they
    cannot actually build the artifact without a [preimage attack] on the digest
    algorithm. However, even still there are no known concerns where this is a
    problem.

[preimage attack]: https://en.wikipedia.org/wiki/Preimage_attack

</details>
<details><summary>Compromise project owner <span>(Build L2+)</span></summary>

*Threat:* An adversary gains owner permissions for the artifact's build project.

*Mitigation:* The build project owner must not have the ability to influence the
build process or provenance generation.

*Example:* MyPackage is built on Awesome Builder under the project "mypackage".
Adversary is an owner of the "mypackage" project. Awesome Builder allows
owners to debug the build environment via SSH. An adversary uses this feature
to alter a build in progress. Solution: Build L3 requires the external parameters
to be complete in the provenance. The attackers access and/or actions within the
SSH connection would be enumerated within the external parameters. The updated
external parameters will not match the declared expectations causing verification
to fail.

</details>
<details><summary>Compromise other build <span>(Build L3)</span></summary>

*Threat:* Perform a malicious build that alters the behavior of a benign
build running in parallel or subsequent environments.

*Mitigation:* Builds are [isolated] from one another, with no way for one to
affect the other or persist changes.

*Example 1:* A build platform runs all builds for project MyPackage on
the same machine as the same Linux user. An adversary starts a malicious build
that listens for another build and swaps out source files, then starts a benign
build. The benign build uses the malicious build's source files, but its
provenance says it used benign source files. Solution: The build platform
changes architecture to isolate each build in a separate VM or similar.

*Example 2:* A build platform uses the same machine for subsequent
builds. An adversary first runs a build that replaces the `make` binary with a
malicious version, then subsequently runs an otherwise benign build. Solution:
The builder changes architecture to start each build with a clean machine image.

</details>
<details><summary>Steal cryptographic secrets <span>(Build L3)</span></summary>

*Threat:* Use or exfiltrate the provenance signing key or some other
cryptographic secret that should only be available to the build platform.

*Mitigation:* Builds are [isolated] from the trusted build platform control
plane, and only the control plane has [access][unforgeable] to cryptographic
secrets.

*Example:* Provenance is signed on the build worker, which the adversary has
control over. Adversary uses a malicious process that generates false provenance
and signs it using the provenance signing key. Solution: Builder generates and
signs provenance in the trusted control plane; the worker has no access to the
key.

</details>
<details><summary>Poison the build cache <span>(Build L3)</span></summary>

*Threat:* Add a malicious artifact to a build cache that is later picked up by a
benign build process ([example][build-cache-poisoning-example]).

*Mitigation:* Build caches must be [isolated][isolated] between builds to prevent
such cache poisoning attacks. In particular, the cache SHOULD be keyed by the
transitive closure of all inputs to the cached artifact, and the cache must
either be only writable by the trusted control plane or have SLSA Build L3
provenance for each cache entry.

*Example 1:* The cache key does not fully cover the transitive closure of all
inputs and instead only uses the digest of the source file itself. Adversary runs
a build over `auth.cc` with command line flags to gcc that define a marco
replacing `CheckAuth(ctx)` with `true`. When subsequent builds build `auth.cc`
they will get the attacker's poisoned instance that does not call `CheckAuth`.
Solution: Build cache is keyed by digest of `auth.cc`, command line, and digest of
gcc so changing the command line flags results in a different cache entry.

*Example 2:* The tenant controlled build process has full write access to the
cache. Adversary observes a legitimate build of `auth.cc` which covers the
transitive closure of all inputs and notes the digest used for caching. The
adversary builds a malicious version of `auth.o` and directly writes it to the
build cache using the observed digest. Subsequent legitimate builds will use
the malicious version of `auth.o`. Solution: Each cache entry is keyed by the
transitive closure of the inputs, and the cache entry is itself a SLSA Build L3
build with its own provenance that corresponds to the key.

</details>
<details><summary>Compromise build platform admin <span>(verification)</span></summary>

*Threat:* An adversary gains admin permissions for the artifact's build platform.

*Mitigation:* The build platform must have controls in place to prevent and
detect abusive behavior from administrators (e.g. two-person approvals, audit
logging).

*Example:* MyPackage is built on Awesome Builder. Awesome Builder allows
engineers on-call to SSH into build machines to debug production issues. An
adversary uses this access to modify a build in progress. Solution: Consumers
do not accept provenance from the build platform unless they trust sufficient
controls are in place to prevent abusing admin privileges.

</details>

### (F) Artifact publication

An adversary uploads a package artifact that does not reflect the intent of the
package's official source control repository.

This is the most direct threat because it is the easiest to pull off. If there
are no mitigations for this threat, then (D) and (E) are often indistinguishable
from this threat.

<!--
**TODO:** We need to define "official source control repository". Its meaning is
not obvious. The gist is that each package theoretically has some "official" or
"canonical" repository from which it "should" be built, and the attack here is
that you either build from a different source repository or otherwise do
something that doesn't reflect that source repository. But we need to nail down
this concept.
-->

<details><summary>Build with untrusted CI/CD <span>(expectations)</span></summary>

*Threat:* Build using an unofficial CI/CD pipeline that does not build in the
correct way.

*Mitigation:* Verifier requires provenance showing that the builder matched an
expected value.

*Example:* MyPackage is expected to be built on Google Cloud Build, which is
trusted up to Build L3. Adversary builds on SomeOtherBuildPlatform, which is only
trusted up to Build L2, and then exploits SomeOtherBuildPlatform to inject
malicious behavior. Solution: Verifier rejects because builder is not as
expected.

</details>
<details><summary>Upload package without provenance <span>(Build L1)</span></summary>

*Threat:* Upload a package without provenance.

*Mitigation:* Verifier requires provenance before accepting the package.

*Example:* Adversary uploads a malicious version of MyPackage to the package
repository without provenance. Solution: Verifier rejects because provenance is
missing.

</details>
<details><summary>Tamper with artifact after CI/CD <span>(Build L1)</span></summary>

*Threat:* Take a benign version of the package, modify it in some way, then
re-upload it using the original provenance.

*Mitigation:* Verifier checks that the provenance's `subject` matches the hash
of the package.

*Example:* Adversary performs a proper build, modifies the artifact, then
uploads the modified version of the package to the repository along with the
provenance. Solution: Verifier rejects because the hash of the artifact does not
match the `subject` found within the provenance.

</details>
<details><summary>Tamper with provenance <span>(Build L2)</span></summary>

*Threat:* Perform a build that would not meet expectations, then modify the
provenance to make the expectations checks pass.

*Mitigation:* Verifier only accepts provenance with a valid [cryptographic
signature][authentic] or equivalent proving that the provenance came from an
acceptable builder.

*Example:* MyPackage is expected to be built by GitHub Actions from the
`good/my-package` repo. Adversary builds with GitHub Actions from the
`evil/my-package` repo and then modifies the provenance so that the source looks
like it came from `good/my-package`. Solution: Verifier rejects because the
cryptographic signature is no longer valid.

</details>

### (G) Distribution channel

An adversary modifies the package on the package registry using an
administrative interface or through a compromise of the infrastructure
including modification of the package in transit to the consumer.

The distribution channel threats and mitigations look very similar to the
Artifact Publication (F) threats and mitigations with the main difference
being that these threats are mitigated by having the *consumer* perform
verification.

The consumer's actions may be simplified if (F) produces a [VSA][vsa].
In this case the consumer may replace provenance verification with
[VSA verification][vsa_verification].

<details><summary>Build with untrusted CI/CD <span>(expectations)</span></summary>

*Threat:* Replace the package with one built using an unofficial CI/CD pipeline
that does not build in the correct way.

*Mitigation:* Verifier requires provenance showing that the builder matched an
expected value or a VSA for corresponding `resourceUri`.

*Example:* MyPackage is expected to be built on Google Cloud Build, which is
trusted up to Build L3. Adversary builds on SomeOtherBuildPlatform, which is only
trusted up to Build L2, and then exploits SomeOtherBuildPlatform to inject
malicious behavior. Adversary then replaces the original package within the
repository with the malicious package. Solution: Verifier rejects because
builder is not as expected.

</details>
<details><summary>Issue VSA from untrusted intermediary <span>(expectations)</span></summary>

*Threat:* Have an unofficial intermediary issue a VSA for a malicious package.

*Mitigation*: Verifier requires VSAs to be issued by a trusted intermediary.

*Example:* Verifier expects VSAs to be issued by TheRepository. Adversary
builds a malicious package and then issues a VSA of their own for the malicious
package. Solution: Verifier rejects because they only accept VSAs from
TheRepository which the adversary cannot issue since they do not have the
corresponding signing key.

</details>
<details><summary>Upload package without provenance or VSA <span>(Build L1)</span></summary>

*Threat:* Replace the original package with a malicious one without provenance.

*Mitigation:* Verifier requires provenance or a VSA before accepting the package.

*Example:* Adversary replaces MyPackage with a malicious version of MyPackage
on the package repository and deletes existing provenance. Solution: Verifier
rejects because provenance is missing.

</details>
<details><summary>Replace package and VSA with another <span>(expectations)</span></summary>

*Threat:* Replace a package and its VSA with a malicious package and its valid VSA.

*Mitigation*: Consumer ensures that the VSA matches the package they've requested (not just the package they received) by following the [verification process](verification_summary#how-to-verify).

*Example:* Adversary uploads a malicious package to `repo/evil-package`,
getting a valid VSA for `repo/evil-package`. Adversary then replaces
`repo/my-package` and its VSA with `repo/evil-package` and its VSA.
Solution: Verifier rejects because the VSA `resourceUri` field lists
`repo/evil-package` and not the expected `repo/my-package`.

</details>
<details><summary>Tamper with artifact after upload <span>(Build L1)</span></summary>

*Threat:* Take a benign version of the package, modify it in some way, then
replace it while retaining the original provenance or VSA.

*Mitigation:* Verifier checks that the provenance or VSA's `subject` matches
the hash of the package.

*Example:* Adversary performs a proper build, modifies the artifact, then
replaces the modified version of the package in the repository and retains the
original provenance. Solution: Verifier rejects because the hash of the
artifact does not match the `subject` found within the provenance.

</details>
<details><summary>Tamper with provenance or VSA <span>(Build L2)</span></summary>

*Threat:* Perform a build that would not meet expectations, then modify the
provenance or VSA to make the expectations checks pass.

*Mitigation:* Verifier only accepts provenance or VSA with a valid [cryptographic
signature][authentic] or equivalent proving that the provenance came from an
acceptable builder or the VSA came from an expected verifier.

*Example 1:* MyPackage is expected to be built by GitHub Actions from the
`good/my-package` repo. Adversary builds with GitHub Actions from the
`evil/my-package` repo and then modifies the provenance so that the source looks
like it came from `good/my-package`. Solution: Verifier rejects because the
cryptographic signature is no longer valid.

*Example 2:* Verifier expects VSAs to be issued by TheRepository. Adversary
builds a malicious package and then modifies the original VSA's `subject`
field to match the digest of the malicious package. Solution: Verifier rejects
because the cryptographic signature is no longer valid.

</details>

## Usage threats

A usage threat is a potential for an adversary to exploit behavior of the
consumer.

### (H) Package selection

The consumer requests a package that it did not intend.

<details><summary>Dependency confusion</summary>

*Threat:* Register a package name in a public registry that shadows a name used
on the victim's internal registry, and wait for a misconfigured victim to fetch
from the public registry instead of the internal one.

*Mitigation:* The mitigation is for the software producer to build internal
packages on a SLSA Level 2+ compliant build system and define expectations for
build provenance. Expectations must be verified on installation of the internal
packages. If a misconfigured victim attempts to install attacker's package with
an internal name but from the public registry, then verification against
expectations will fail.

For more information see [Verifying artifacts](verifying-artifacts.md)
and [Defender's Perspective: Dependency Confusion and Typosquatting Attacks](/blog/2024/08/dep-confusion-and-typosquatting).

</details>

<details><summary>Typosquatting</summary>

*Threat:* Register a package name that is similar looking to a popular package
and get users to use your malicious package instead of the benign one.

*Mitigation:* This threat is not currently addressed by SLSA. That said, the
requirement to make the source available can be a mild deterrent, can aid
investigation or ad-hoc analysis, and can complement source-based typosquatting
solutions.

</details>

### (I) Usage

The consumer uses a package in an unsafe manner.

<details><summary>Improper usage</summary>

*Threat:* The software can be used in an insecure manner, allowing an
adversary to compromise the consumer.

*Mitigation:* This threat is not addressed by SLSA, but may be addressed by
efforts like [Secure by Design][secure-by-design].

</details>

## Dependency threats

A dependency threat is a potential for an adversary to introduce unintended
behavior in one artifact by compromising some other artifact that the former
depends on at build time. (Runtime dependencies are excluded from the model, as
[noted below](#runtime-dep).)

Unlike other threat categories, dependency threats develop recursively through
the supply chain and can only be exploited indirectly. For example, if
application *A* includes library *B* as part of its build process, then a build
or source threat to *B* is also a dependency threat to *A*. Furthermore, if
library *B* uses build tool *C*, then a source or build threat to *C* is also a
dependency threat to both *A* and *B*.

This version of SLSA does not explicitly address dependency threats, but we
expect that a future version will. In the meantime, you can [apply SLSA
recursively] to your dependencies in order to reduce the risk of dependency
threats.

<!--
-   **TODO:** Should we distinguish 1P vs 3P boundaries in the diagram, or
    otherwise visualize 1P/3P?
-   **TODO:** Expand to cover typosquatting, dependency confusion, and other
    "dependency" threats.
-   **TODO:** The word "compromised" is a bit too restrictive. If the publisher
    intends to do harm, either because they tricked you into using a dependency
    (typosquatting or dependency confusion), or because they were good and now
    do something bad, that's not really "compromised" per se.
-   **TODO:** Should we expand this to cover "transitive SLSA verification"?
-   **TODO:** Update the Terminology page to show "build time" vs "runtime",
    since the latter term results in confusion. Also consider the term "deploy
    time" as an alternative.
-->

[apply SLSA recursively]: verifying-artifacts.md#step-3-optional-check-dependencies-recursively

### Build dependency

An adversary compromises the target artifact through one of its build
dependencies. Any artifact that is present in the build environment and has the
ability to influence the output is considered a build dependency.

<details id="included-dep"><summary>Include a vulnerable dependency (library, base image, bundled file, etc.)</summary>

*Threat:* Statically link, bundle, or otherwise include an artifact that is
compromised or has some vulnerability, causing the output artifact to have the
same vulnerability.

*Example:* The C++ program MyPackage statically links libDep at build time. A
contributor accidentally introduces a security vulnerability into libDep. The
next time MyPackage is built, it picks up and includes the vulnerable version of
libDep, resulting in MyPackage also having the security vulnerability.

*Mitigation:* A future
[Dependency track](../../current-activities#dependency-track) may
provide more comprehensive guidance on how to address more specfiic
aspects of this threat.

</details>
<details id="build-tool"><summary>Use a compromised build tool (compiler, utility, interpreter, OS package, etc.)</summary>

*Threat:* Use a compromised tool or other software artifact during the build
process, which alters the build process and injects unintended behavior into the
output artifact.

*Mitigation:* This can be partially mitigated by treating build tooling,
including OS images, as any other artifact to be verified prior to use.
The threats described in this document apply recursively to build tooling
as do the mitigations and examples.  A future
[Build Environment track](../../current-activities#build-environment-track) may
provide more comprehensive guidance on how to address more specfiic
aspects of this threat.

*Example:* MyPackage is a tarball containing an ELF executable, created by
running `/usr/bin/tar` during its build process. An adversary compromises the
`tar` OS package such that `/usr/bin/tar` injects a backdoor into every ELF
executable it writes. The next time MyPackage is built, the build picks up the
vulnerable `tar` package, which injects the backdoor into the resulting
MyPackage artifact.  Solution: [apply SLSA recursively] to all build tools
prior to the build.  The build platform verifies the disk image,
or the individual components on the disk image, against the associated
provenance or VSAs prior to running a build.  Depending on where the initial
compromise took place (i.e. before/during vs *after* the build of the build tool itself), the modified `/usr/bin/tar` will fail this verification.

</details>

<details id="runtime-dep-at-build-time"><summary>Use a compromised runtime dependency during the build (for tests, dynamic linking, etc.)</summary>

*Threat:* During the build process, use a compromised runtime dependency (such
as during testing or dynamic linking), which alters the build process and
injects unwanted behavior into the output.

**NOTE:** This is technically the same case as [Use a compromised build
tool](#build-tool). We call it out to remind the reader that
[runtime dependencies](#runtime-dep) can become build dependencies if they are
loaded during the build.

*Example:* MyPackage has a runtime dependency on package Dep, meaning that Dep
is not included in MyPackage but required to be installed on the user's machine
at the time MyPackage is run. However, Dep is also loaded during the build
process of MyPackage as part of a test. An adversary compromises Dep such that,
when run during a build, it injects a backdoor into the output artifact. The
next time MyPackage is built, it picks up and loads Dep during the build
process. The malicious code then injects the backdoor into the new MyPackage
artifact.

*Mitigation:* In addition to all the mitigations for build tools, you can often
avoid runtime dependencies becoming build dependencies by isolating tests to a
separate environment that does not have write access to the output artifact.

</details>

### Related threats

The following threats are related to "dependencies" but are not modeled as
"dependency threats".

<details id="runtime-dep"><summary>Use a compromised dependency at runtime <span>(modeled separately)</span></summary>

*Threat:* Load a compromised artifact at runtime, thereby compromising the user
or environment where the software ran.

*Example:* MyPackage lists package Dep as a runtime dependency. Adversary
publishes a compromised version of Dep that runs malicious code on the user's
machine when Dep is loaded at runtime. An end user installs MyPackage, which in
turn installs the compromised version of Dep. When the user runs MyPackage, it
loads and executes the malicious code from Dep.

*Mitigation:* N/A - SLSA's
threat model does not explicitly model runtime dependencies. Instead, each
runtime dependency is considered a distinct artifact with its own threats.

</details>

## Availability threats

<!--
**TODO:** Merge into the list above rather than having a separate section.
-->

An availability threat is a potential for an adversary to deny someone from
reading a source and its associated change history, or from building a package.

SLSA does not currently address availability threats, though future versions might.

<details><summary>Delete the code</summary>

*Threat:* Perform a build from a particular source revision and then delete that
revision or cause it to get garbage collected, preventing anyone from inspecting
the code.

*Mitigation:* This threat is not currently addressed by SLSA.

</details>
<details><summary>A dependency becomes temporarily or permanently unavailable to the build process</summary>

*Threat:* Unable to perform a build with the intended dependencies.

*Mitigation:* This threat is not currently addressed by SLSA. That said, some
solutions to support hermetic and reproducible builds may also reduce the
impact of this threat.

</details>
<details><summary>De-list artifact</summary>

*Threat:* The package registry stops serving the artifact.

*Mitigation:* This threat is not currently addressed by SLSA.

</details>
<details><summary>De-list provenance</summary>

*Threat:* The package registry stops serving the provenance.

*Mitigation:* This threat is not currently addressed by SLSA.

</details>

## Verification threats

Threats that can compromise the ability to prevent or detect the supply chain
security threats above.

<details><summary>Tamper with recorded expectations</summary>

*Threat:* Modify the verifier's recorded expectations, causing the verifier to
accept an unofficial package artifact.

*Mitigation:* Changes to recorded expectations requires some form of
authorization, such as two-party review.

*Example:* The package ecosystem records its expectations for a given package
name in a configuration file that is modifiable by that package's producer. The
configuration for MyPackage expects the source repository to be
`good/my-package`. The adversary modifies the configuration to also accept
`evil/my-package`, and then builds from that repository and uploads a malicious
version of the package. Solution: Changes to the recorded expectations require
two-party review.

</details>
<details><summary>Exploit cryptographic hash collisions</summary>

*Threat:* Exploit a cryptographic hash collision weakness to bypass one of the
other controls.

*Mitigation:* Choose secure algorithms when using cryptographic digests, such
as SHA-256.

*Examples:* Attacker crafts a malicious file with the same MD5 hash as a target
benign file. Attacker replaces the benign file with the malicious file.
Solution: Only accept cryptographic hashes with strong collision resistance.

</details>

</article>

<!-- Links -->

[apply SLSA recursively]: verifying-artifacts.md#step-3-optional-check-dependencies-recursively
[authentic]: build-requirements.md#provenance-authentic
[build-cache-poisoning-example]: https://adnanthekhan.com/2024/05/06/the-monsters-in-your-build-cache-github-actions-cache-poisoning/
[exists]: build-requirements.md#provenance-exists
[isolated]: build-requirements.md#isolated
[unforgeable]: build-requirements.md#provenance-unforgeable
[secure-by-design]: https://www.cisa.gov/securebydesign
[supply chain threats]: threats-overview
[vsa]: verification_summary
[vsa_verification]: verification_summary#how-to-verify
