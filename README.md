# GIT OPS CONFIGURATION FOR OPENSHIFT 4 CLUSTERS

## Installing Red Hat Openshift GitOps

In the spirit of gitops best practices, this [installation instructions](https://docs.openshift.com/container-platform/4.7/operators/user/olm-installing-operators-in-namespace.html#olm-installing-operator-from-operatorhub-using-cli_olm-installing-operators-in-namespace) use the CLI instead of the web console.

The operator will be available to all namespaces so there is no need to create an Operator group.

A subscription object needs to be created in the __openshift-operators__ namespace, the yaml file __rhogo-subscription.yaml__ contains its definition, it uses the stable channel and an automatic approval strategy for updates.  The name of the operator, the source and the source namespace, are all extracted from the package manifest, keep in mind that these can vary between OCP versions:

```
$ oc describe packagemanifest/openshift-gitops-operator -n openshift-marketplace
```
Create the subcription in the openshift cluster:

```
$ oc create -f rhogo-subscription.yaml
```

## Accessing the ArgoCD web

Upon installation a link to the ArgoCD web is added to the "Rubic's Kube" icon in the Openshift web console.

The user to access is __admin__, the password can be obtained from a secret in the __openshift-gitops__ namespace:

```
# oc extract secret/openshift-gitops-cluster -n openshift-gitops --to -
# admin.password
zAx8oVwakQsq5JitMFIp53ecGd14rSlv
```

