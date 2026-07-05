# mq-infra

This project is the shared MQ build infrastructure — the Tekton pipeline and task library every queue manager repo builds with, plus the MQ image Dockerfile and Helm chart template.

## Table of Contents

* [Introduction](#introduction)
* [Pipeline (vendored + modernized)](#pipeline-vendored--modernized)
* [Pre-requisites](#pre-requisites)
* [Queuemanager Details](#queuemanager-details)

## Introduction

This guide provides a walkthrough on how to set up an Queuemanager.  The Github repository contains a Dockerfile and Helm Chart template, and — under `tekton/` — the vendored pipeline and task library (originally from the [Cloud Native Toolkit](https://cloudnativetoolkit.dev/)) used to build a Queuemanager image and deliver it through GitOps on a containerized instance of IBM MQ.

This repo contains the below artifacts.


```
.
├── Dockerfile
├── README.md
├── chart
    └── base
        ├── Chart.yaml
        ├── config
        │   └── config.mqsc
        ├── security
        │   └── config.mqsc
        ├── templates
        │   ├── NOTES.txt
        │   ├── _helpers.tpl
        │   ├── configmap.yaml
        │   └── qm-template.yaml
        └── values.yaml
└── tekton
    ├── LICENSE                        # Apache-2.0 (upstream license for the vendored files)
    ├── kustomization.yaml             # apply the whole factory with one kustomize build
    ├── pipelines
    │   └── mq-qm-dev.yaml             # the queue manager pipeline
    └── tasks                          # the task library the pipeline references
        ├── ibm-setup.yaml
        ├── ibm-build-tag-push.yaml
        ├── ibm-smoke-tests-mq.yaml
        ├── ibm-tag-release.yaml
        ├── ibm-img-release.yaml
        ├── ibm-img-scan.yaml
        └── ibm-gitops-kustomize-mq.yaml
```

- `ibm-mqadvanced-server-integration` docker image that comes with CloudPaks. This image can be further customized if we need additional configurations that are part of queuemanager.
- `Helm Charts` - Currently, we are using quickstart template as our base and building additional things on top of it for deploying the queuemanager.
- `Configurations` - Like mentioned earlier, the configurations can be embedded as part of Dockerfile. Alternatively, they can also be injected as configmaps.


## Pipeline (vendored + modernized)

**Provenance:** the pipeline and tasks under `tekton/` are ported from the Cloud Native Toolkit's [IBM/ibm-garage-tekton-tasks](https://github.com/IBM/ibm-garage-tekton-tasks) (pipeline `ibm-mq` and the `ibm-*` tasks it references), licensed Apache-2.0 — the upstream license text is kept at `tekton/LICENSE`. They were vendored here in July 2026 so this repo actually owns the factory role, and modernized against a live OpenShift 4.18 / OpenShift Pipelines 1.22 cluster:

- **Tekton `tekton.dev/v1`** for the Pipeline and all Tasks (upstream is `v1beta1`; it still converts today, but v1 is the API current clusters serve).
- **Pod-security fixes** — the real 2021 rot. The buildah/skopeo steps of `ibm-build-tag-push` and `ibm-img-release` carry `securityContext: {capabilities: {add: [SETFCAP]}}` (without it buildah dies writing `uid_map` under current SCCs), and the `ibm-smoke-tests-mq` deploy/health-check/cleanup steps run with `runAsUser: 0` so they can write the shared clone volume. Both fixes are exactly the patches that turned red validation PipelineRuns green.
- **Kustomize delivery** — the upstream `ibm-helm-release` (Artifactory) + `ibm-gitops` (Helm-chart reference) stages are replaced by one `ibm-gitops-kustomize-mq` task that writes the tested image + MQSC as a kustomize overlay (`queuemanager.yaml`, `kustomization.yaml`, `static-definitions.mqsc`) to a delivery branch of [multi-tenancy-gitops-apps](https://github.com/ibmclientengineering/multi-tenancy-gitops-apps), ready for a PR.
- **Maintained step images** — the EOL `node:lts-stretch`, `ibmcloud-dev` (yq v3) and 2021 buildah/skopeo pins are replaced with current images (buildah/skopeo at the digests OpenShift Pipelines itself ships; `alpine/k8s` as the kubectl+helm+yq tools image, scripts ported to yq v4; `node:22` for release tagging; pinned Trivy).

The pipeline is named **`mq-qm-dev`** and exposes: `git-url`, `git-revision`, `scan-image`, `image-url` (registry override; empty = the internal registry `image-registry.openshift-image-registry.svc:5000/<namespace>/<repo>:<short-sha>`), `security` and `ha` (which chart/kustomize flavor the smoke and delivery stages use), and `gitops-repo-url` (where the delivery stage pushes). Before the first run the pipeline namespace needs the `ibm-entitled-registry-credentials` (or `registry-auth`) secret so buildah can pull the entitled MQ base image, and a `git-credentials` secret (username/password) so the tag-release and gitops stages can push. The cluster's integrated image registry must be enabled (`managementState: Managed`) — it is the only in-cluster registry nodes can pull from.

The intended consumption path is GitOps: `mq/environments/ci/pipelines` in your `multi-tenancy-gitops-apps` fork references `tekton/` here as a remote kustomize base, so an Argo CD Application delivers the Pipelines and Tasks into the `ci` namespace by commit. `oc apply -k tekton/` works for a manual bootstrap.

## Pre-requisites

- [IBM Catalog Operator](https://www.ibm.com/docs/en/app-connect/11.0.0?topic=iicia-enabling-operator-catalog-cloud-pak-foundational-services-operator)
- [IBM Common Services](https://github.com/IBM/ibm-common-service-operator)
- [IBM MQ Operator](https://www.ibm.com/docs/en/ibm-mq/9.2?topic=integration-using-mq-in-cloud-pak-openshift)

## Queuemanager Details

- Intially, security and native HA are disabled.
- To enable security, set `security` to true in `Values.yaml`.
- To enable high availability, set `ha` to true in `Values.yaml`.

Note: This project demonstrates how to add in the `mqsc` configuration files. Similarly, if you want to configure an `qm.ini`, please create a configMap for the same and inject it under `spec.queueManager` in the `qm-template.yaml` using the below snippet.

```yaml
ini:
- configMap:
    name: {{ .Values.ini.configmap }}
    items:
    - {{ .Values.ini.name }}
```

### Configuration

- Create required queues to store info.
- Create channel to provide necessary communication links.
- For this queuemanager, channel authentication is disabled.

## Enable Security

If you want to enable the queuemanager to use security, we need to set the `security` flag to `true` in `Values.yaml`. By default, it is always `false`.

### Configuration

- Create required queues to store info.
- Except for the channel used by MQ Explorer, all channels with inbound connections are blocked.
- Create channel to provide necessary communication links.
- Allow access to the LDAP channel.
- Privileged user IDs asserted by a client-connection channel are blocked by means of the special value *MQADMIN* by default. We are removing this default rule.
- For authentication, our queuemanage is using LDAP. We define necessary LDAP configurations. Our sample configurations as follows.

  ```
    DEFINE AUTHINFO(USE.LDAP) +
    AUTHTYPE(IDPWLDAP) +
    CONNAME('openldap.openldap(389)') +
    LDAPUSER('cn=admin,dc=ibm,dc=com') LDAPPWD('admin') +
    SECCOMM(NO) +
    USRFIELD('uid') +
    SHORTUSR('uid') +
    BASEDNU('ou=people,dc=ibm,dc=com') +
    AUTHORMD(SEARCHGRP) +
    BASEDNG('ou=groups,dc=ibm,dc=com') +
    GRPFIELD('cn') +
    CLASSGRP('groupOfUniqueNames') +
    FINDGRP('uniqueMember') +
    CHCKCLNT(REQUIRED) +
    REPLACE
    ALTER QMGR CONNAUTH(USE.LDAP)
  ```
- Enable TLS protocol security. We can specify the cryptographic algorithms that are used by the TLS protocol by supplying a CipherSpec as part of the channel definition along with the authenticated user information.
- This privileged user will be allowed to access the queue manager and interact with it.

## Enable Native HA

If you want to enable the queuemanager to use native high avaibility capability, we need to set the `ha` flag to `true` in `Values.yaml`. By default, it is always `false`.

- A Native HA configuration provides a highly available queue manager where the recoverable MQ data (for example, the messages)  are replicated across multiple sets of storage, preventing loss from storage failures.
- This is suitable for use with cloud block storage.
