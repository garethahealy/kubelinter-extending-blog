[![License](https://img.shields.io/hexpm/l/plug.svg?maxAge=2592000)]()
[![Generate README.html](https://github.com/garethahealy/kubelinter-extending-blog/actions/workflows/docs.yaml/badge.svg)](https://github.com/garethahealy/kubelinter-extending-blog/actions/workflows/docs.yaml)

# Extending kube-linter To Build A Custom Template

Over the past few years, I have taken a keen interest in DevSecOps and software supply chain security in the Kubernetes space.
One area that I have looked at from multiple tools is how to automate security practices and policies based on the resources being
deployed. These investigations have led me to write multiple blogs on OPA [[1]](https://cloud.redhat.com/blog/automate-your-security-practices-and-policies-on-openshift-with-open-policy-agent) and
Kyverno [[2]](https://cloud.redhat.com/blog/automate-your-security-practices-and-policies-on-openshift-with-kyverno), 
[[3]](https://cloud.redhat.com/blog/software-supply-chain-security-on-openshift-with-kyverno-and-cosign), 
[[4]](https://cloud.redhat.com/blog/guide-to-mutations-of-a-resource-on-openshift-with-kyverno).

For this blog post, I will take a look at kube-linter and how a developer can write a custom template that will be used to
check if a `Route` has TLS configured.

## What Is kube-linter?
For those new to kube-linter, let's firstly start off with a quick introduction and show an example.

So what is kube-linter? straight from the docs:

> KubeLinter analyzes Kubernetes YAML files and Helm charts, and checks them against a variety of best practices, 
> with a focus on production readiness and security.

> KubeLinter runs sensible default checks, designed to give you useful information about your Kubernetes YAML files and Helm charts. 
> This is to help teams check early and often for security misconfigurations and DevOps best practices. 
> Some common examples of these include running containers as a non-root user, enforcing least privilege, 
> and storing sensitive information only in secrets.

> KubeLinter is configurable, so you can enable and disable checks, as well as create your own custom checks, 
> depending on the policies you want to follow within your organization.

> When a lint check fails, KubeLinter reports recommendations for how to resolve any potential issues and returns a non-zero exit code.

In simple terms; it's a CLI tool that can be run against your `Helm` chart to validate if you are following well-defined best practices.

## Let's Give It A Little Play
Firstly, let's install it. If a `go install` method is offered, I typically use it. But, if you haven't got `go` installed and configured correctly
there are a number of [different options](https://docs.kubelinter.io/#/?id=installing-kubelinter), such as a `brew` or `docker`.

The below presumes you've already installed `go` and configured `$GOBIN` as part of your `$PATH` correctly.

```bash
GO111MODULE=on go install golang.stackrox.io/kube-linter/cmd/kube-linter@0.2.6
kube-linter version
```

Now, let's run an example that only runs the `minimum-three-replicas` check:

```bash
git clone https://github.com/garethahealy/kubelinter-extending-blog.git
cd kubelinter-extending-blog
kube-linter --config policy/kube-linter-config.yaml lint policy/pod-replicas-below-one/test_data/unit/list.yml
```

Which should show the below:

```bash
policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) object has 1 replica but minimum required replicas is 3 (check: minimum-three-replicas, remediation: Increase be number of replicas in the deployment to at least three to increase the fault tolerancy of the deployment.)

Error: found 1 lint errors
```

Hopefully, from what `kube-linter` outputted you've worked out that the `Deployment` in our example only has `.spec.replicas: 1`
which if you want to run a highly available workload then this isn't the best idea.

Since `kube-linter` comes with a variety of policies out of the box, we can run all the checks against our example:

```bash
kube-linter lint policy/pod-replicas-below-one/test_data/unit/list.yml
```

Which should show the below:

```bash
policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) The container "bar" is using an invalid container image, "". Please use images that are not blocked by the `BlockList` criteria : [".*:(latest)$" "^[^:]*$" "(.*/[^:]+)$"] (check: latest-tag, remediation: Use a container image with a specific tag other than latest.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) object has no selector specified (check: mismatching-selector, remediation: Confirm that your deployment selector correctly matches the labels in its pod template.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) container "bar" does not have a read-only root file system (check: no-read-only-root-fs, remediation: Set readOnlyRootFilesystem to true in the container securityContext.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) container "bar" is not set to runAsNonRoot (check: run-as-non-root, remediation: Set runAsUser to a non-zero number and runAsNonRoot to true in your pod or container securityContext. Refer to https://kubernetes.io/docs/tasks/configure-pod-container/security-context/ for details.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) container "bar" has cpu request 0 (check: unset-cpu-requirements, remediation: Set CPU requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) container "bar" has cpu limit 0 (check: unset-cpu-requirements, remediation: Set CPU requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) container "bar" has memory request 0 (check: unset-memory-requirements, remediation: Set memory requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)

policy/pod-replicas-below-one/test_data/unit/list.yml: (object: <no namespace>/replicaisone apps/v1, Kind=Deployment) container "bar" has memory limit 0 (check: unset-memory-requirements, remediation: Set memory requests and limits for your container based on its requirements. Refer to https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#requests-and-limits for details.)
```

Again, from the output we can see that we don't have any requests or limits set, which again, could cause problems for the stability of the platform
if the application starts to become overloaded or thrash the system and starve the `kubelet` process.

## So What Are Checks And Templates? Huh?!
Currently, `kube-linter` supports two concepts. A `check` and a `template`.

- a `check` is an input to a `template`.
- a `template` is the parser that works on the resource to validate the `check`.

OK, but what does that look like in the real world? A [check](https://docs.kubelinter.io/#/generated/checks?id=access-to-create-pods) 
could be that you only want to allow a `Role` to be able to `create` certain resources, such as a `Deployment`. 
The [template](https://docs.kubelinter.io/#/generated/templates?id=access-to-resources) then looks at the specific `Role`
and checks that the input (i.e.: the check) is not being violated.

Currently, all templates need to be written in `go` so developers who want to write policies for `kube-linter` need to follow
a fork, commit, build and carry model. In the [future](https://github.com/stackrox/kube-linter/issues/47), 
it is planned to allow users to write policies using other methods.

If you want to see what comes out of the box, you can simply run:
```bash
kube-linter checks list
```

And:

```bash
kube-linter templates list
```

## Let's Get Into The Coding!
In the intro, I said we'd extend `kube-linter` to support the OCP `Route` resource so that we could write a policy to check that TLS
was configured. For those that just want to jump into the repository and have a dig around themselves, all the code is pushed to:
- https://github.com/garethahealy/kube-linter/tree/routetermination

For those that are new to `go` or unsure of the `kube-linter` codebase, I'll step through the changes to enable you to
write your own policies.

So let's get into it!

1. As we are adding a new schema resource to `kube-linter`, we need to tell it how to parse the `YAML` of the `Route`. So let's add the `go`
package to the schema builder via [commit/516bcf3c3d407177bd0f995d4b72fdf04b8bc65d](https://github.com/garethahealy/kube-linter/commit/516bcf3c3d407177bd0f995d4b72fdf04b8bc65d).

2. `kube-linter` makes use of [object kind](https://github.com/stackrox/kube-linter/tree/main/pkg/objectkinds) 
matcher functions that decides if the resource input matches what the template can work against.
For our template, we'll create an `IngressLike` matcher via 
[commit/4335e829d570a81ef8a9069cd9a4a91077b9b772](https://github.com/garethahealy/kube-linter/commit/4335e829d570a81ef8a9069cd9a4a91077b9b772).
Currently, it only supports a `Route` but in the future, it could be extended to support `networking.k8s.io/v1/Ingress` 
or if you are on the cutting edge, a `gateway.networking.k8s.io/v1alpha2/Gateway`.

3. Now the meat of the policy; the template that implements the check. We need to create two objects, the `params` and the `template` via 
[commit/112b7f2beb09714a1ce971ee894ec1f6b1e03307](https://github.com/garethahealy/kube-linter/commit/112b7f2beb09714a1ce971ee894ec1f6b1e03307).
Currently, we don't have any parameters to pass into our template as our policy is binary; we have TLS defined or not. 
The template follows a standard pattern of implementing `check.Template`. The main body of code goes in the `Instantiate` parameter.
As you can see from the `WrapInstantiateFunc` body, we:
   1. [Retrieve](https://github.com/garethahealy/kube-linter/commit/112b7f2beb09714a1ce971ee894ec1f6b1e03307#diff-8cde8ff009e5d38004a1f35be006ba12d40a9e075a9ef38f2a3237ef4c4d4de0R29) the `Route` object.
   2. [Check](https://github.com/garethahealy/kube-linter/commit/112b7f2beb09714a1ce971ee894ec1f6b1e03307#diff-8cde8ff009e5d38004a1f35be006ba12d40a9e075a9ef38f2a3237ef4c4d4de0R31) if TLS is null/nil and Termination is empty.
   3. [Return](https://github.com/garethahealy/kube-linter/commit/112b7f2beb09714a1ce971ee894ec1f6b1e03307#diff-8cde8ff009e5d38004a1f35be006ba12d40a9e075a9ef38f2a3237ef4c4d4de0R32) message.
   
4. OK, so that looks quite simple so far. But how do we know it works correctly? we write tests! The unit tests via 
[commit/f662b2968d51a42163301083d0d8ba274879c41b](https://github.com/garethahealy/kube-linter/commit/f662b2968d51a42163301083d0d8ba274879c41b)
check for matching and non-matching inputs, as we don't want the policy firing incorrectly.

5. We are now at the point of knowing our new template works correctly but if we were to compile the binary and attempt to run the template via
`kube-linter lint --include route-termination --do-not-auto-add-defaults route-termination.yml`, `kube-linter` would complain it does not include a `check`
called `route-termination`. There is some minor plumbing work required via 
[commit/b811a542fba36a7d99ab24abfba6b3c8cc4cbb01](https://github.com/garethahealy/kube-linter/commit/b811a542fba36a7d99ab24abfba6b3c8cc4cbb01)
to let the CLI know if a user wants to run `route-termination` then it is for the template we have just written.

6. Now the bulk of the code is done, we just need to write a simple `BATS` integration test to validate the end-to-end process and that if we provide
a YAML file containing a `Route` without TLS defined, we get the expected error message back via 
[commit/37d267cfaa7ea2b2c95e7ec85a88a86d26eb20cd](https://github.com/garethahealy/kube-linter/commit/37d267cfaa7ea2b2c95e7ec85a88a86d26eb20cd).

7. And finally, let's build our new binary and execute the tests!
   1. `make go-generated-srcs`
   2. `make test`
   3. `make e2e-test`
   4. `make e2e-bats`

At this point, you've built a new `template` that has both unit and integration tests and is ready to be used within your DevSecOps
pipelines.

## What's Next?
Have a policy that is not currently implemented by `kube-linter` that you think could be of benefit to the wider OCP/k8s community?
get [contributing](https://github.com/garethahealy/kube-linter#community)!

### Thanks For Reviewing
- [@janisz](https://github.com/janisz)