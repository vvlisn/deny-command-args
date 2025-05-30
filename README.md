[![Stable](https://img.shields.io/badge/status-stable-brightgreen?style=for-the-badge)](https://github.com/kubewarden/community/blob/main/REPOSITORIES.md#stable)

# container-command-control

A Kubewarden policy that provides fine-grained control over container `args` and `command` fields in Kubernetes Deployments.

## Introduction

This policy enhances security by controlling how containers can be configured within Kubernetes Deployments. By default, it prevents the use of `args` and `command` fields in container specifications, which helps maintain consistency and security across your cluster.

The policy is configurable via runtime settings, allowing administrators to:
- Enforce strict control by denying any use of `args` and `command` (default behavior)
- Allow the use of these fields when needed for specific use cases

You can configure the policy using this structure:

```json
{
  "allow_args_and_command": false
}
```

When `allow_args_and_command` is:
- `false` (default): Rejects any Deployment where containers specify `args` or `command`
- `true`: Allows containers to use `args` and `command` fields

## Code organization

The policy is organized into several key files:

- `settings.go`: Handles policy configuration parsing and validation
- `validate.go`: Contains the core logic for validating Deployment resources
- `main.go`: Registers the policy entry points with the Kubewarden SDK

## Examples

### Default Deny Configuration
```bash
kubectl apply -f - <<EOF
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: deny-command-args
spec:
  module: registry://ghcr.io/vvlisn/policies/deny-command-args:latest
  rules:
  - apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    operations:
    - CREATE
  mutating: false
  settings:
    allow_args_and_command: false
EOF
```




### Allow Configuration
```bash
kubectl apply -f - <<EOF
apiVersion: policies.kubewarden.io/v1
kind: ClusterAdmissionPolicy
metadata:
  name: deny-command-args
spec:
  module: registry://ghcr.io/vvlisn/policies/deny-command-args:latest
  rules:
  - apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments"]
    operations:
    - CREATE
  mutating: false
  settings:
    allow_args_and_command: true
EOF
```

## Implementation details

> **DISCLAIMER:** WebAssembly is a constantly evolving area.
> This document describes the status of the Go ecosystem as of July 2023.

Currently, the official Go compiler can't produce WebAssembly binaries that can run **outside** the browser.
Because of that, you can only build Kubewarden Go policies with the [TinyGo](https://tinygo.org/) compiler.

Kubewarden policies need to process JSON data.
For example, the policy settings and the actual request received by Kubernetes.
TinyGo doesn't yet support the full Go Standard Library,
it has limited support of Go reflection.
Because of that, it's impossible to import the official Kubernetes Go library from upstream (e.g.: `k8s.io/api/core/v1`).
Importing these official Kubernetes types results in a compilation failure.

However, it's still possible to write a Kubewarden policy by using certain third party libraries.

> **Warning:**
> Using an older version of TinyGo results in runtime errors due to the limited support for Go reflection.

This list of libraries can be useful when writing a Kubewarden policy:

- [Kubernetes Go types](https://github.com/kubewarden/k8s-objects) for TinyGo:
the official Kubernetes Go Types can't be used with TinyGo.
This module provides all the Kubernetes Types in a TinyGo-friendly way.
- Parsing JSON: Queries against JSON documents can be written using the
[gjson](https://github.com/tidwall/gjson) library.
The library features a powerful query language that allows quick navigation of JSON documents and data retrieval.
- Generic `set` implementation: Using [Set](https://en.wikipedia.org/wiki/Set_(abstract_data_type)) data types can reduce the amount of code in a policy,
see the `union`, `intersection`, `difference`, and other operations provided by a Set implementation.
The [mapset](https://github.com/deckarep/golang-set) can be used when writing policies.

Last, but not least, this policy takes advantage of helper functions provided by
[Kubewarden's Go SDK](https://github.com/kubewarden/policy-sdk-go).

## Testing

This policy comes with unit tests implemented using the Go testing
framework.

As usual, the tests are defined in `_test.go` files.
As these tests aren't part of the final WebAssembly binary, the official Go compiler can be used to run them.

The unit tests can be run via a simple command:

```console
make test
```

It's also important to test the final result of the TinyGo compilation:
the actual WebAssembly module.

This is done with a second set of end-to-end tests.
These tests use the `kwctl` cli provided by the Kubewarden project to load and execute the policy.

The e2e tests are implemented using
[bats](https://github.com/bats-core/bats-core),
the Bash Automated Testing System.

The end-to-end tests are defined in the `e2e.bats` file and can be run using:

```console
make e2e-tests
```

## Automation

This project has the following [GitHub Actions](https://docs.github.com/en/actions):

- `e2e-tests`: this action builds the WebAssembly policy,
installs the `bats` utility and then runs the end-to-end test.
- `unit-tests`: this action runs the Go unit tests.
- `release`: this action builds the WebAssembly policy and pushes it to a user defined OCI registry
([ghcr](https://ghcr.io) is a good candidate).
