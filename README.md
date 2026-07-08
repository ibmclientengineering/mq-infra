# mq-infra

The IBM MQ **Tekton factory** — the reusable machinery for running IBM MQ as code: one parameterized task library and a set of pipelines that build, prove, and deliver queue managers and their apps through GitOps.

> Part of the **IBM Client Engineering Cloud Pak for Integration production-deployment demo** — this is the MQ build-and-promotion engine that feeds the shared GitOps repos every environment reconciles from.

## What this is

Change the params, not the tasks. One pipeline serves every queue manager: point it at a QM repo, and it builds the image, smoke-tests it against a real queue manager, scans it, and delivers a `QueueManager` custom resource as a kustomize overlay to the GitOps apps repo — ready for a pull request. A separate promotion pipeline gates on functional evidence, then copies the proven overlay to the next environment and opens the PR. The machine assembles the evidence; a human merges; Argo CD deploys.

## What's inside

- **`tekton/pipelines/`** — three pipelines, all `tekton.dev/v1`:
  - `mq-qm-dev` — build the queue-manager image, smoke-test it against a live QM, scan it, and deliver the `QueueManager` overlay to [multi-tenancy-gitops-apps](https://github.com/ibmclientengineering/multi-tenancy-gitops-apps).
  - `mq-spring-app-dev` — build the JMS app image and prove it round-trips a message (send → recv) through a live queue manager.
  - `mq-qm-promote` — a functional **put/get gate** on a dedicated queue, then promote the overlay to the next environment as a PR.
- **`tekton/tasks/`** — the shared task library the pipelines compose: `ibm-setup`, `ibm-build-tag-push`, `ibm-smoke-tests-mq`, `ibm-img-scan`, `ibm-tag-release`, `ibm-img-release`, `ibm-gitops-kustomize-mq` (writes the overlay), `ibm-mq-gate-test` (the promotion gate), `ibm-gitops-promote` (copy-forward + PR), and `ibm-app-smoke-test`.
- **`Dockerfile`** — the queue-manager image, `FROM` the entitled `ibm-mqadvanced-server-integration` base (pinned by digest), ready for MQSC to be baked in.
- **`chart/base/`** — a Helm chart the smoke stage renders to stand up a throwaway `QueueManager` for testing, with `security` and `ha` (Native HA) flavors and MQSC config maps.
- **`tekton/kustomization.yaml`** — apply the whole factory with one `oc apply -k tekton/`, or reference it as a remote kustomize base from GitOps.

## Why it's built this way

- **Factory, not copy-paste.** The tasks are generic and parameterized, so a hundred queue managers share one audited pipeline. New QM? New params — no forked YAML to drift.
- **Nothing ships unproven.** `mq-qm-dev` won't deliver an image the smoke test didn't stand up and connect to; `mq-qm-promote` won't advance an overlay the put/get gate didn't pass. The gate uses a **dedicated queue** — it never drains an application queue to prove liveness.
- **Promotion is a pull request.** A machine proposes the change (copy the proven overlay forward, rebuild the target kustomization, open the PR); a human reviews the diff and merges; Argo CD reconciles. That's separation of duties and a full Git audit trail for every environment hop.
- **The 2026 CR model.** Delivery writes a first-class `QueueManager` custom resource for the IBM MQ Operator as a kustomize overlay — not a deprecated Helm-release-into-Artifactory handoff. The overlay pins the exact tested image and MQSC, so what you reviewed is what deploys.
- **Modernized and hardened.** Ported from the Cloud Native Toolkit's `ibm-garage-tekton-tasks` (Apache-2.0; license kept at `tekton/LICENSE`) and rebuilt against a live OpenShift 4.18 / Pipelines 1.22 cluster: `v1` APIs, current buildah/skopeo/tool images, the pod-security fixes that turn red validation runs green, and gate/promote scripts that pass params as inert env vars (never shell-interpolated).

## How it fits the bigger picture

`mq-infra` is the MQ half of a pair of build factories. Its sibling, **ace-infra**, does the same for App Connect Enterprise — `ibm-gitops-promote` is deliberately product-generic (point `product-path` at `ace/environments` and it promotes ACE too). Both factories deliver into **[multi-tenancy-gitops-apps](https://github.com/ibmclientengineering/multi-tenancy-gitops-apps)**, the workload layer of the four-repo **multi-tenancy-gitops** family (root `multi-tenancy-gitops` → `-infra` → `-services` → `-apps`) that Argo CD reconciles onto the cluster. `mq-spring-app-dev` builds the JMS application from **mq-spring-app** and proves it against a running queue manager, closing the loop from broker to app.

Maintained by IBM Client Engineering.
