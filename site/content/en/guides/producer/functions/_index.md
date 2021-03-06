---
title: "Functions"
linkTitle: "Functions"
weight: 4
type: docs
description: >
   Writing config functions to generated, transform, and validate resources.
---

## Functions Developer Guide

Config functions are conceptually similar to Kubernetes *controllers* and
*validating webhooks* -- they are programs which read resources as input, then
write resources as output (creating, modifying, deleting, or validating
resources).

Unlike controllers and validating webhooks, config functions can be run outside
of the Kubernetes control plane.  This allows them to run in more contexts or
embedded in other systems. For example, functions could be:

- manually run locally
- automatically run locally as part of *make*, *mvn*, *go generate*, etc
- automatically run in CICD systems
- run by controllers as reconcile implementations

{{< svg src="images/fn" >}}

{{% pageinfo color="primary" %}}
Unlikely pure-templating and DSL approaches, functions must be able to both
*read* and *write* resources, and specifically should be able to read resources
they have previously written -- updating the inputs rather generating new
resources.

This mirrors the level-triggered controller architecture used in the Kubernetes
control plane. A function reconciles the desired state of the resource
configuration to match the declared state specified by the functionConfig.

Functions that implement abstractions should update resources they have
generated in the past by reading them from the input.
{{% /pageinfo %}}

The following function runtimes are available in kpt:

| Runtime    | Read Resources From | Write Resources To | Write Error Messages To | Validation Failure | Maturity |
|------------|---------------------|--------------------|-------------------------|--------------------|----------|
| Containers | STDIN               | STDOUT             | STDERR                  | Exit Code          | Beta     |
| Starlark   | `ctx.resource_list` | `ctx.resource_list`| `log`                   | Exit Code          | Alpha    |

## Input / Output

Functions read a `ResourceList`, modify it, and write it back out.

### ResourceList.items

The ResourceList contains:

- (input+output) a list of resource `items`
- (input) configuration for the function
- (output) validation results

Items are resources read from some source -- such as a package directory.

After a function adds, deletes or modifies items, the items will be written to
a sink. In most cases the sink will be the same as the source (directory).

```yaml
kind: ResourceList
items:
- apiVersion: apps/v1
  kind: Deployment
  spec:
  ...
- apiVersion: v1
  kind: Service
  spec:
  ...
```

### ResourceList.functionConfig

Functions may optionally be configured using the `ResourceList.functionConfig`
field. `functionConfig` is analogous to a Deployment, and `items` is analogous
to the set of all resources in the Deployment controller in-memory cache (e.g.
all the resources in the cluster) -- this includes the ReplicaSets created,
updated and deleted for that Deployment.

*Functions are responsible for identifying which resources from the `items` that they own.*

```yaml
kind: ResourceList
functionConfig:
  apiVersion: example.com/v1alpha1
  kind: Foo
  spec:
    foo: bar
    ...
items:
  ...
```

{{% pageinfo color="primary" %}}
Some functions use a ConfigMap as the functionConfig instead of introducing a
new resource type bespoke to the function.
{{% /pageinfo %}}

## Running Functions

Functions may be run either imperatively using the form
`kpt fn run DIR/ --image fn`, or they may be run declaratively using the form
`kpt fn run DIR/`.

Either way, `kpt fn run` will

1. read the package directory as input
2. encapsulate the package resources in a `ResourceList`
3. run the function(s), providing the ResourceList as input
4. write the function(s) output back to the package directory; creating,
   deleting, or updating resources

### Imperative Run

Functions can be run imperatively by specifying the `--image` flag.

`kpt fn run DIR/ --image some-image:version` will:

- create a container from the image
- read all resources from the package directory
- provide the resources as input to the function (container)
- write the output items back to the package directory

```sh
kpt pkg get https://github.com/GoogleContainerTools/kpt-functions-sdk.git/example-configs example-configs
mkdir results/
kpt fn run example-configs/ --results-dir results/ --image gcr.io/kpt-functions/validate-rolebinding:results -- subject_name=bob@foo-corp.com
```

{{% pageinfo color="primary" %}}
If key-value pairs are provided after a `--` argument, then a ConfigMap will be
generated with these values as `data` elements, and the ConfigMap will be set
as the functionConfig field.
{{% /pageinfo %}}

### Declarative Run

Functions can be specified declaratively using the
`config.kubernetes.io/function` annotation on a resource serving as the
functionConfig.

`kpt fn run DIR/` (without `--image`) will:

- read all resources from the package directory
- for each resource with this annotation, kpt will run the specified function
  (using the resource as the functionConfig)
  - functions are run sequentially, with the output of each function provided
    as input to the next
- write the output items back to the package directory

{{% pageinfo color="primary" %}}
If multiple functions are specified in the same yaml file (e.g. separated by
`---`), then they are run in the order they are specified.
{{% /pageinfo %}}

## Declaring Functions

Functions may be run by declaratively through the
`config.kubernetes.io/function` annotation.

The following is a declaration to run the function implemented by the
`gcr.io/example.com/image:v1.0.0` image, and to provide this ExampleFunction as
the functionConfig.

```yaml
apiVersion: example.com/v1alpha1
kind: ExampleFunction
metadata:
  annotations:
    config.kubernetes.io/function: |
      container:
        image: gcr.io/example.com/image:v1.0.0
```

## Function Scoping

Functions are scoped to resources that live in their same directory, or
subdirectories of their directory.

`kpt fn run DIR/` will recursively traverse DIR/ looking for declared functions
and invoking them -- passing in only those resources scoped to the function.

## Validation

### ResourceList.results

Functions may define validation results through the `results` field. When
functions are run using the `--results-dir`, each function's results field will
be written to a file under the specified directory.

```yaml
kind: ResourceList
functionConfig:
  apiVersion: example.com/v1alpha1
  kind: Foo
  spec:
    foo: bar
    ...
results:
- name: "kubeval"
  items:  
  - severity: error # one of ["error", "warn", "info"] -- error code should be non-0 if there are 1 or more errors
    tags: # arbitrary metadata about the result
      error-type: "field"
    message: "Value exceeds the namespace quota, reduce the value to make the pod schedulable"
    resourceRef: # key to lookup the resource
      apiVersion: apps/v1
      kind: Deployment
      name: foo
      namespace: bar
    file:
      # optional if present as annotation
      path: deploy.yaml # read from annotation if present
      # optional if present as annotation
      index: 0 # read from annotation if present
    field:
      path: "spec.template.spec.containers[3].resources.limits.cpu"
      currentValue: "200" # number | string | boolean
      suggestedValue: "2" # number | string | boolean
  - severity: warn
    ...
- name: "something else"
  items:
  - severity: info
     ...
```

## Deferring Failure

When running multiple validation functions, it may be desired to defer failures
until all functions have been run so that the results of all functions are
written to the results directory. Functions can specify that failures should be
deferred by specifying `deferFailure` in the declaration.

```yaml
apiVersion: example.com/v1alpha1
kind: ExampleFunction
metadata:
  annotations:
    config.kubernetes.io/function: |
      container:
        image: gcr.io/example.com/image:v1.0.0
      deferFailure: true # continue running functions if this fails, and fail at the end
```

## FunctionConfig for imperative runs

When functions are run imperatively, the functionConfig may be generated from
commandline arguments.

When functions are run using the form
`kpt fn run DIR/ --image foo:v1 -- a=b c=d`, the arguments following `--` are
parsed into a ConfigMap which is set as the functionConfig.

```yaml
kind: ResourceList
functionConfig:
  apiVersion: v1
  kind: ConfigMap
  spec:
    a: b
    ...
items:
  ...
```

If the first argument after `--` is *not* a key=value pair, it will be used as
the functionConfig type.

`kpt fn run DIR/ --image foo:v1 -- Foo a=b c=d`

```yaml
kind: ResourceList
functionConfig:
  kind: Foo
  spec:
    a: b
    ...
items:
  ...
```

## Next Steps

- Learn how to [run functions].
- Find out how to structure a pipeline of functions from the
  [functions concepts] page.
- Consult the [fn command reference].

[run functions]: ../../consumer/function/
[functions concepts]: ../../../concepts/functions/
[fn command reference]: ../../../reference/fn/
