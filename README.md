# GIT OPS CONFIGURATION FOR OPENSHIFT 4 CLUSTERS

## Installing Red Hat Openshift GitOps

In the spirit of gitops this [installation instructions](https://docs.openshift.com/container-platform/4.7/operators/user/olm-installing-operators-in-namespace.html#olm-installing-operator-from-operatorhub-using-cli_olm-installing-operators-in-namespace) use the CLI instead of the web console.

The operator will be available to all namespaces so there is no need to create an Operator group.

A subscription object needs to be created in the __openshift-operators__ namespace.  The yaml file __rhogo-subscription.yaml__ contains the subscription object definition, it uses the stable channel and an automatic approval strategy for updates.  The name of the operator, the source and the source namespace, are all extracted from the package manifest, keep in mind that these can between OCP versions:

```
$ oc describe packagemanifest/openshift-gitops-operator -n openshift-marketplace
```
Create the subcription in the openshift cluster:

```
$ oc create -f rhogo-subscription.yaml
```

