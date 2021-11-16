---
layout: blog
title: 'Pod Security Admission (PSA) Graduates to Beta'
date: 2021-12-15
slug: pod-security-admission-psa-beta
---

**Authors:** Jim Angel (Google), Lachlan Evenson (Microsoft)

_Editor's note: [Jim Angel](https://twitter.com/JimmAngel) and [Lachlan Evenson](https://twitter.com/LachlanEvenson) are both [CNCF Cloud Ambassadors](https://www.cncf.io/people/ambassadors/) and wear many hats in the open source Kubernetes community. Jim is a Cloud Consultant at Google and Lachlan is a Principal Program Manager on the Azure Core Upstream._

## Introduction

With the release of Kubernetes v1.23, [Pod Security Admission (PSA)](https://kubernetes.io/docs/concepts/security/pod-security-admission/) has now entered beta. PSA is a [built-in](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) admission controller that evaluates pod specifications against a predefined set of [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) and determines whether to `admit` or `deny` the pod from running. PSA is the successor to [Pod Security Policy](https://kubernetes.io/docs/concepts/policy/pod-security-policy/) which was deprecated in v1.21 and is removed in v1.25. Let's explore how Pod Security Admission works in the hope that cluster administrators and developers alike will use this to enforce secure defaults for their workloads. In this blog we will cover the key concepts of Pod Security Admission along with how to use it. 

## Why Pod Security Admission

Pod Security Admission overcomes the shortcomings of, deprecated, Pod Security Policy (PSP):

   * Policy authorization model — challenging to deploy with controllers.
   * Challenging to enable — no dry-run/audit capabilities made it hard to enable.
   * Inconsistent and Unbounded API — security constraints were ever-changing which led to the API needing changes.

The shortcomings of PSP made it very difficult to use which led the community to reevaluate whether or not a better implementation could achieve the same goals. One of those goals was to provide an out-of-the-box solution to apply security best practices. Pod Security Admission ships with predefined Pod Security levels that a cluster administrator can configure to meet the desired security posture.

It's important to note that Pod Security Admission doesn't have complete feature parity with the deprecated Pod Security Policy. Specifically, it doesn't have the ability to `mutate` or change Kubernetes resources to auto-remediate a policy violation on behalf of the user. Additionally, it doesn't provide fine-grain control over each allowed field and value within a pod specification or any other Kubernetes resource that you may wish to evaluate. If you need more fine-grained policy control then take a look at the [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) project which supports such use cases.

Pod Security Admission also adheres to Kubernetes best practices of declarative object management by denying resources that violate the policy. This requires resources to be updated in source repositories, and tooling to be updated prior to being deployed to Kubernetes.

## How Does Pod Security Admission Work?
    
Pod Security Admission is a built-in [admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/). Admission controllers function by intercepting requests in the Kubernetes API server prior to persistence to storage. They can either `admit` or `deny` a request. In the case of Pod Security Admission, pod specifications will be evaluated against a configured policy in the form of a Pod Security Standard. This means that security sensitive fields in a pod specification will only be allowed to have [specific](https://kubernetes.io/docs/concepts/security/pod-security-standards/#profile-details) values. 

## Configuring Pod Security Admission
    
### Pod Security Standards
    
In order to use Pod Security Admission we first need to understand [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/). These standards define three different policy levels that range from permissive to restrictive. These levels are as follows:
   * Privileged — open and unrestricted
   * Baseline — Covers most common known privilege escalations and provides an easier onboarding
   * Restricted — Highly restricted, covering best practices. May cause compatibility issues

Each of these policy levels define which fields are restricted within a pod specification and the allowed values. Some of the fields restricted by these policies include:
   * `spec.containers[*].ports`
   * `spec.volumes[*].hostPath`
   * `spec.securityContext`
   * `spec.containers[*].securityContext`
    
These policy levels can be applied in two different ways. Either via namespace labels or an AdmissionConfiguration resource. Using namespace labels allows for granular per namespace policy selection whereas using AdmissionConfiguration allows cluster-level defaulting along with exemptions.

### Policy modes

Policies are applied using modes. Here is a list of modes:
   * `enforce` — Any Pods that violate the policy will be rejected
   * `audit` — Pods with violations will be allowed and an audit annotation will be added
   * `warn` — Pods that violate the policy will be allowed and a warning message will be sent back to the user.

In addition to modes you can also pin the policy to a specific version for example v1.22. Pinning to a specific version allows the behavior to remain consistent as policy changes happen over Kubernetes releases.

## Hands on demo

### Prerequisites

- [kind CLI installed](https://kind.sigs.k8s.io/docs/user/quick-start/#installation)
- [Docker](https://docs.docker.com/get-docker/) or [Podman](https://podman.io/getting-started/installation) container runtime & CLI

### Deploy a kind cluster

```shell
kind create cluster --image kindest/node:v1.23.0
```

It might take awhile to start and once it's started it might take a minute or so before the node becomes ready.

```shell
kubectl cluster-info --context kind-kind
```

Wait for the node STATUS to become ready.

```shell
kubectl get nodes
```

The output is similar to this:

```shell
NAME                 STATUS   ROLES                  AGE   VERSION
kind-control-plane   Ready    control-plane,master   54m   v1.23.0
```

### Confirm PSA is enabled

The best way to [confirm the API's default enabled plugins](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#which-plugins-are-enabled-by-default) is to check the Kubernetes API container's help arguments.

```shell
kubectl -n kube-system exec kube-apiserver-kind-control-plane -it -- kube-apiserver -h | grep "default enabled ones"
```

The output is similar to this:

```shell
...
      --enable-admission-plugins strings
admission plugins that should be enabled in addition
to default enabled ones (NamespaceLifecycle, LimitRanger,
ServiceAccount, TaintNodesByCondition, PodSecurity, Priority,
DefaultTolerationSeconds, DefaultStorageClass,
StorageObjectInUseProtection, PersistentVolumeClaimResize,
RuntimeClass, CertificateApproval, CertificateSigning,
CertificateSubjectRestriction, DefaultIngressClass,
MutatingAdmissionWebhook, ValidatingAdmissionWebhook,
ResourceQuota).
...
```

`PodSecurity` is listed in the group of default enabled admission plugins.

### Configure Pod Security Admission

Policies are applied to a namespace via labels. These labels are as follows:
   * `pod-security.kubernetes.io/<MODE>: <LEVEL>` (required)
   * `pod-security.kubernetes.io/<MODE>-version: <VERSION>` (*optional*, defaults to latest)

A specific version can be supplied for each enforcement mode. The version pins the policy to the version that was defined at that Kubernetes version. This is to increase flexibility for future  Kubernetes enhancements and unknown security issues.

From earlier, the modes are:
   * `enforce` — Any Pods that violate the policy will be rejected
   * `audit` — Pods with violations will be allowed and an audit annotation will be added
   * `warn` — Pods that violate the policy will be allowed and a warning message will be sent back to the user.

> Note: The default for enforce/warn/audit is `privileged` if not specified

When to `warn`?

Don't use`warn` for the exact same level+version of the policy as `enforce`. In the admission sequence, if `enforce` fails, the entire sequence fails before evaluating the `warn`.

The typical uses for `warn` are:

* `warn` at the same level but a different version (e.g. pin `enforce` to *restricted+v1.23* and `warn` at *restricted+latest*)
* `warn` at a stricter level (e.g. `enforce` baseline, `warn` restricted)


Let's create 3 namespaces and apply all 3 PSA security profiles. First, create the namespaces:

```shell
kubectl create namespace kube-test-privileged
kubectl create namespace kube-test-baseline
kubectl create namespace kube-test-restricted
```

Add the respective labels.

```shell
# kube-test-privileged enforces a "privileged" security policy and warns / audits on baseline
kubectl label --overwrite ns kube-test-privileged \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=baseline \
  pod-security.kubernetes.io/audit=baseline

# kube-test-baseline enforces a "baseline" security policy and warns / audits on restricted
kubectl label --overwrite ns kube-test-baseline \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted

# kube-test-restricted enforces a "restricted" security policy and audits on restricted
kubectl label --overwrite ns kube-test-restricted \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted
```

### Deploy demo workloads

Each workload represents a higher level of security that would not pass the profile that comes after it.

**Privileged level and workload**

For the privileged pod, allow privilege escalation. This allows the process inside a container to gain new processes and can be dangerous if untrusted ([more info](https://kubernetes.io/docs/concepts/security/pod-security-standards/#privileged)).

First, let's try to deploy it in the restricted namespace:

```shell
cat <<EOF | kubectl -n kube-test-restricted apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: true
EOF
```

The output is similar to this:

```shell
Error from server (Forbidden): error when creating "STDIN": pods "nginx-privileged" is forbidden: violates PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

Now let's try the `kube-test-privileged` namespace

```shell
cat <<EOF | kubectl -n kube-test-privileged apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: true
EOF
```

The output is similar to this:

```shell
pod/nginx-privileged created
```

We can run `kubectl -n kube-test-privileged get pods` to verify it is running.

**Baseline level and workload**

The baseline pod demonstrates sensible defaults but does not allow privilege escalation, so the pod above would fail here too ([more info](https://kubernetes.io/docs/concepts/security/pod-security-standards/#baseline)). Baseline accepts null / no values for unset keys.

First, let's try to deploy it in the restricted namespace:

```shell
cat <<EOF | kubectl -n kube-test-restricted apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-baseline
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        add:
          - NET_BIND_SERVICE
          - CHOWN
EOF
```

The output is similar to this:

```shell
Error from server (Forbidden): error when creating "STDIN": pods "nginx-baseline" is forbidden: violates PodSecurity "restricted:latest": unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]; container "nginx" must not include "CHOWN" in securityContext.capabilities.add), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

Now let's try the `kube-test-baseline` namespace

```shell
cat <<EOF | kubectl -n kube-test-baseline apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-baseline
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        add:
          - NET_BIND_SERVICE
          - CHOWN
EOF
```

The output is similar to this:

```shell
Warning: would violate PodSecurity "restricted:latest": unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]; container "nginx" must not include "CHOWN" in securityContext.capabilities.add), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/nginx-baseline created
```

Remember, we set the `kube-test-baseline` namespace to `warn` based on the restricted profile. We can run `kubectl -n kube-test-baseline get pods` to verify it is running.

**Restricted level and workload**

Restricted requires rejection of all privileged parameters. It is the most secure with a trade-off for complexity ([more info](https://kubernetes.io/docs/concepts/security/pod-security-standards/#restricted)). The restricted policy allows containers to add the `NET_BIND_SERVICE` capability.

```shell
cat <<EOF | kubectl -n kube-test-restricted apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-restricted
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        add:
          - NET_BIND_SERVICE
EOF
```

The output is similar to this:

```shell
Error from server (Forbidden): error when creating "STDIN": pods "nginx-restricted" is forbidden: violates PodSecurity "restricted:latest": unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

This is because the restricted profile explicitly requires that certain values are set to the most secure parameters.

By requiring explicit values, manifests become more declarative and your entire security model can shift left. With the `restricted` level of enforcement, a company could audit their cluster's compliance based on permitted manifests.

Let's fix each warning resulting in the following file:

```shell
cat <<EOF | kubectl -n kube-test-restricted apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-restricted
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    securityContext:
      seccompProfile:
        type: RuntimeDefault
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
        add:
          - NET_BIND_SERVICE
EOF
```

The output is similar to this:

```shell
pod/nginx-restricted created
```

At this point, if you wanted to dive deeper into permissions and what is or isn't enabled on a certain workload, exec into the control plane and play around `containerd`+`crictl inspect` with:

```shell
docker exec -it kind-control-plane bash

# list running containers
crictl ps

# inspect each one by container ID
crictl inspect <CONTAINER ID>
```

### Applying a cluster-wide policy

In addition to applying labels to namespaces to configure policy you can also configure cluster-wide policies and exemptions using the AdmissionConfiguration resource. 

Using this resource, policy definitions are applied cluster-wide by default and any policy that is applied via namespace labels will take precedence. 

There is no runtime configurable API for this resource and a cluster administrator would need to specify a path to the file below via the `--admission-control-config-file` flag on the API server. 

In the following resource we are enforcing the baseline policy and warning and auditing the baseline policy. We are also making the kube-system namespace exempt from this policy.

It's not recommended to alter control plane / clusters after install, so let's build a new cluster with a default policy on all namespaces.

First, delete the current cluster.

```shell
kind delete cluster
```

Create a Pod Security configuration. 

<!-- TODO: link to the official docs after posted: https://github.com/kubernetes/website/blob/dev-1.23/content/en/docs/tasks/configure-pod-container/enforce-standards-admission-controller.md?plain=1#L25-L55
--->

<!-- TODO: is this a BUG?:
It errors with runtimeClassNames: [] and runs without it, tested on both alphav1/betav1 apis.

2021-11-16T02:59:09.121870759Z stderr F E1116 02:59:09.121285       1 run.go:120] "command failed" err="failed to initialize admission: couldn't init admission plugin \"PodSecurity\": strict decoding error: unknown field \"runtimeClassNames\""
 --->


```shell
cat <<EOF > pod-security.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1beta1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      enforce-version: "latest"
      audit: "baseline"
      audit-version: "latest"
      warn: "restricted"
      warn-version: "latest"
    exemptions:
      # Array of authenticated usernames to exempt.
      usernames: []
      # Array of runtime class names to exempt.
      runtimeClasses: []
      # Array of namespaces to exempt.
      namespaces: [kube-system]
EOF
```

We now have a default baseline policy, next pass it to the kind configuration to enable the `--admission-control-config-file` API server argument and pass the policy file.

```shell
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
        # enable admission-control-config flag on the API server
        extraArgs:
          admission-control-config-file: /etc/kubernetes/policies/pod-security.yaml
        # mount new file / directories on the control plane
        extraVolumes:
          - name: policies
            hostPath: /etc/kubernetes/policies
            mountPath: /etc/kubernetes/policies
            readOnly: true
            pathType: "DirectoryOrCreate"
  # mount the local file on the control plane
  extraMounts:
  - hostPath: ./pod-security.yaml
    containerPath: /etc/kubernetes/policies/pod-security.yaml
    readOnly: true
EOF
```

```shell
kind create cluster --image kindest/node:v1.23.0 --config kind-config.yaml
```

Let's look at the default namespace.

```shell
kubectl describe namespace default
```

The output is similar to this:

```shell
Name:         default
Labels:       kubernetes.io/metadata.name=default
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

Let's create a new namespace and see if the labels apply there.

```shell
kubectl create namespace test-defaults
kubectl describe namespace test-defaults
```

Same.

```shell
Name:         test-defaults
Labels:       kubernetes.io/metadata.name=test-defaults
Annotations:  <none>
Status:       Active

No resource quota.

No LimitRange resource.
```

Can we deploy a workload that is privileged?

```shell
cat <<EOF | kubectl -n test-defaults apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: true
EOF
```

Hmm... yep. Our default `warn` level is working at least.

```shell
Warning: would violate PodSecurity "restricted:latest": allowPrivilegeEscalation != false (container "nginx" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container "nginx" must set securityContext.capabilities.drop=["ALL"]), runAsNonRoot != true (pod or container "nginx" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container "nginx" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
pod/nginx-privileged created
```

Let's delete the pod with `kubectl -n test-defaults delete pod/nginx-privileged`.

Is my config even working?

```shell
# shell into the control plane
docker exec -it kind-control-plane bash

# cat out the file we mounted
cat /etc/kubernetes/policies/pod-security.yaml

# check the api server logs
cat /var/log/containers/kube-apiserver*.log 

# check the api server config
cat /etc/kubernetes/manifests/kube-apiserver.yaml 
```

**UPDATE:** The baseline policy permits `allowPrivilegeEscalation`. While I cannot see the Pod Security default levels of enforcement, they are there. Let's try to provide a manifest that violates the baseline by requesting hostNetwork access.

```shell
cat <<EOF | kubectl -n test-defaults apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-privileged
spec:
  containers:
  - name: nginx
    image: nginx
  hostNetwork: true
EOF
```

The output is similar to this:

```shell
Error from server (Forbidden): error when creating "STDIN": pods "nginx-privileged" is forbidden: violates PodSecurity "baseline:latest": host namespaces (hostNetwork=true)
```

#### Yes!!! It worked! 🎉🎉🎉

I later found out, another way to check if things are operating as intended is to check the raw API server metrics endpoint.

Run the following command:

```
kubectl get --raw /metrics | grep pod_security_evaluations_total
```

The output is similar to this:

```
# HELP pod_security_evaluations_total [ALPHA] Number of policy evaluations that occurred, not counting ignored or exempt requests.
# TYPE pod_security_evaluations_total counter
pod_security_evaluations_total{decision="allow",mode="enforce",policy_level="baseline",policy_version="latest",request_operation="create",resource="pod",subresource=""} 2
pod_security_evaluations_total{decision="allow",mode="enforce",policy_level="privileged",policy_version="latest",request_operation="create",resource="pod",subresource=""} 0
pod_security_evaluations_total{decision="allow",mode="enforce",policy_level="privileged",policy_version="latest",request_operation="update",resource="pod",subresource=""} 0
pod_security_evaluations_total{decision="deny",mode="audit",policy_level="baseline",policy_version="latest",request_operation="create",resource="pod",subresource=""} 1
pod_security_evaluations_total{decision="deny",mode="enforce",policy_level="baseline",policy_version="latest",request_operation="create",resource="pod",subresource=""} 1
pod_security_evaluations_total{decision="deny",mode="warn",policy_level="restricted",policy_version="latest",request_operation="create",resource="controller",subresource=""} 2
pod_security_evaluations_total{decision="deny",mode="warn",policy_level="restricted",policy_version="latest",request_operation="create",resource="pod",subresource=""} 2
```

A monitoring tool could ingest these metrics too for reporting, assessments, or measuring trends.

### Auditing

Auditing is another way to track what policies are being enforced in your cluster. Auditing is out of scope for this blog, but if you're interested in learning how to set up auditing with kind you can review their [official docs](https://kind.sigs.k8s.io/docs/user/auditing/). As of [version 1.11](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.11.md#sig-auth), Kubernetes audit logs include two annotations that indicate whether or not a request was authorized (`authorization.k8s.io/decision`) and the reason for the decision (`authorization.k8s.io/reason`). Audit events can be streamed to a webhook for monitoring, tracking, or alerting.

The audit events look similar to the following:

```shell
{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":"","pod-security.kubernetes.io/audit":"allowPrivilegeEscalation != false (container \"nginx\" must set securityContext.allowPrivilegeEscalation=false), unrestricted capabilities (container \"nginx\" must set securityContext.capabilities.drop=[\"ALL\"]), runAsNonRoot != true (pod or container \"nginx\" must set securityContext.runAsNonRoot=true), seccompProfile (pod or container \"nginx\" must set securityContext.seccompProfile.type to \"RuntimeDefault\" or \"Localhost\")"}}
```

Auditing is also a good first step in evaluating your cluster's current compliance with Pod Security. The Kubernetes Enhancement Proposal (KEP) hints at a future where `baseline` [could be the default for unlabeled namespaces](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/2579-psp-replacement/README.md#rollout-of-baseline-by-default-for-unlabeled-namespaces).

# PSP migrations

If you're already using PSP, SIG Auth has created a guide and [published the steps to migrate off of PSP](https://kubernetes.io/docs/tasks/configure-pod-container/migrate-from-psp/).

To summarize the process:
- Update all existing PSPs to be non-mutating
- Apply PSA policies in `warn` or `audit` mode
- Upgrade PSA policies to `enforce` mode
- Remove `PodSecurityPolicy` from `--enable-admission-plugins`

Listed as "optional future extensions" and currently out of scope, SIG Auth has kicked around the idea of providing a tool to assist with migrations. More [details in the KEP](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/2579-psp-replacement/README.md#automated-psp-migration-tooling).

## Wrap up

Pod Security Admission is a promising new feature that provides an out-of-the-box way to allow users to improve the security posture of their workloads. Like any new enhancement that has matured to beta, we ask that you try it out, provide feedback, or share your experience via either raising a Github issue or joining SIG-auth community meetings. It's our hope that Pod Security Admission will be deployed on every cluster in our ongoing pursuit as a community to make Kubernetes security a priority.

## Additional resources

- [Official Pod Security Admission Docs](/docs/concepts/security/pod-security-admission/)
- [Enforce Pod Security Standards with Namespace Labels](/docs/tasks/configure-pod-container/enforce-standards-namespace-labels/)
- [Enforce Pod Security Standards by Configuring the Built-in Admission Controller](/docs/tasks/configure-pod-container/enforce-standards-admission-controller/)
- [Official Kubernetes Enhancement Proposal](https://github.com/kubernetes/enhancements/blob/master/keps/sig-auth/2579-psp-replacement/README.md) (KEP)
- [PodSecurityPolicy Deprecation: Past, Present, and Future](/blog/2021/04/06/podsecuritypolicy-deprecation-past-present-and-future/)
- [Hands on with Kubernetes Pod Security Admission](https://medium.com/@LachlanEvenson/hands-on-with-kubernetes-pod-security-admission-b6cac495cd11)
- [Applying Pod Security Standards at Cluster level](/docs/tutorials/tutorials/pod-security/cluster-level-pss.md)
