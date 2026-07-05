# kargo-promotion-patterns

One application, two promotion patterns, side by side. Both pipelines deploy
`ghcr.io/jake-painter/guestbook` through dev, staging, and prod on an RKE2
homelab cluster, driven by Kargo and delivered by Argo CD. The point of this
repo is the comparison: the pipelines share a kustomize base and a Warehouse
subscription, and differ only in what a promotion writes and where.

To see the entire difference in one command:

```
diff kargo/classic/stages.yaml kargo/rendered/stages.yaml
```

## The two patterns

**Classic (branch: main).** A promotion commits a one-line `newTag` bump into
`classic/<stage>/kustomization.yaml`. Argo CD applications track main and run
kustomize at sync time. Git history on main is the promotion history. What the
cluster runs is whatever the overlay renders to at sync time, which you can
only see by rendering it yourself.

**Rendered (branches: env/dev, env/staging, env/prod).** A promotion checks
out main, sets the image tag in memory, renders the overlay to plain
manifests, and commits the result to the stage's env branch. main never
changes. Argo CD applications track the env branch and apply exactly what is
there, with no client-side rendering.

## Tradeoffs, honestly

The classic pattern is simpler to operate: one branch, no branch sprawl, and
the overlay remains the single source of truth. Its cost is opacity at review
time. A PR or commit shows intent (a tag changed) rather than effect (what
YAML will actually hit the cluster), and a kustomize or helm upgrade can
change rendered output with no visible diff in the overlay at all.

The rendered pattern makes the applied state first-class. The env branch IS
what the cluster runs, byte for byte: auditors can read it, `git diff
env/staging env/prod` answers "what is different between environments" exactly,
rollback is a branch revert, and Argo CD syncs faster because there is nothing
to render. Its costs are more promotion machinery, N extra branches to reason
about, and a subtle truth-splitting: main holds the sources, env branches hold
the materialized outcome, and only Kargo should ever write the latter.

Rendered tends to win as environment count, compliance burden, or rendering
complexity grows. Classic tends to win for small teams and few environments.

## Layout

```
kargo-promotion-patterns/
|-- base/                 shared kustomize base (deployment + service)
|-- classic/<stage>/      overlays Kargo EDITS on main (classic pipeline)
|-- rendered/<stage>/     overlays Kargo RENDERS FROM (rendered pipeline; main untouched)
|-- kargo/classic/        Project, Warehouse, Stages for the classic pipeline
`-- kargo/rendered/       Project, Warehouse, Stages for the rendered pipeline
```

Kargo resources are synced by an Argo CD Application in a separate private
GitOps repo, alongside the six workload Applications (one per stage per
pattern). Env branches are Kargo-managed: never commit to them by hand.

## Operating notes

Freight moves dev -> staging -> prod; staging and prod only accept freight
verified in the stage upstream of them. Promotions write directly (no PR
gate); a PR-gated variant of the same machinery runs elsewhere in the lab
using the git-open-pr and git-wait-for-pr promotion steps. The Warehouse
expects semver tags; if image tags are not semver, switch the subscription to
`imageSelectionStrategy: NewestBuild`. Each project namespace needs a git
credential Secret (label `kargo.akuity.io/cred-type: git`) so promotions can
push to this repo. The sealed files under `kargo/*/creds.sealed.yaml` are
SealedSecrets: ciphertext encrypted to one specific cluster's controller,
safe to publish and useless anywhere else.

This is a personal homelab reference, not an official Akuity or Kargo project.
