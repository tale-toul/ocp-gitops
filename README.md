# GITOPS CONFIGURATION FOR OPENSHIFT 4 CLUSTERS

## Installing the Red Hat Openshift GitOps Operator

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

After creating the subscription, the GitOps and Pipelines operators are installed.  An ArgoCd instance is created in the __openshift-gitops__ namespace.  

Initially the service account running ArgoCD: __openshift-gitops-applicationset-controller__, contains a restricted set of permissions that limit its use when configuring the Openshift cluster.  For full cluster configuration capabilites, add the __cluster-admin__ cluster role to the service account:
```
$ oc adm policy add-cluster-role-to-user cluster-admin -z openshift-gitops-argocd-application-controller -n openshift-gitops
```

## Accessing the ArgoCD web

Upon installation a link to the ArgoCD web is added to the "Rubic's Kube" icon in the Openshift web console.

The user to access is __admin__, the password can be obtained from a secret in the __openshift-gitops__ namespace:

```
# oc extract secret/openshift-gitops-cluster -n openshift-gitops --to -
# admin.password
zAx8oVwakQsq5JitMFIp53ecGd14rSlv
```

## Openshift cluster configuration

The namespaces directory contains additional directories for the creation of projects (namespaces) in the Openshift cluster.  Each directory contains a yaml definition for the namespace object, and a resources subdirectory for the resources to be created in it.  The resources subdirectory exists to make sure that ArgoCD creates the namespace before attempting to create its resources. 

## ArgoCD component manifests

The directory CRDs contains yaml definitions for the _applications_ created inside ArgoCD.

## Sealed Secrets

This section explains how to use [Bitnami's sealed secrets](https://github.com/bitnami-labs/sealed-secrets) to manage kubernetes secrets in a public git repository.  

Kubernetes secrets are not encrypted, the information they contain is base 64 encoded, so if a secret contains sensitive information like passwords, certificates, tokens, etc. They should not be pushed to a git repository, but if we want to use ArgoCD to deploy applications and apply configuration changes to an Openshift cluster we need a way to store secrets in git.

Bitnami's sealed secret is a combination of a client side utility (kubeseal) and a cluster side controller.  The client side utility is used to encrypt the kubernetes secrets which can then be pushed to the git repository, while the server side controller is used to decrypt the secrets when they are deployed to the cluster with ArgoCD for example.

### Installation

To Install the client just download the binary from github and install it to a directory in your path:
```
$ wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.16.0/kubeseal-linux-amd64 -O kubeseal
$ sudo install -m 755 kubeseal /usr/local/bin/kubeseal
```

To install the controller there are different options, in this case the __Sealed Secrets Operator__ from the operator hub will be used.  All components are installed by default in the namespace called sealed-secrets, if the namespace does not exist it is created during the installation process.  
Go to the Operator Hub section in the Openshift cluster and look for _secrets_ in the search box, select __Sealed Secrets Operator_ (Helm)__ and click Install.  When the operator is installed go to SealedSecretController and click on __Create SealedSecretController__, the name given to the new controller must be all in small caps.

To install the operator manually in the namespace __sealed-secrets__:
* Create the sealed-secrets namespace:
```
$ oc new-project sealed-secrets
```
* Create the operator group, the yaml definition can be found at ocp-gitops/Operators/SealedSecrets/operatorgroup.yaml 
```
$ oc apply -f operatorgroup.yaml 
operatorgroup.operators.coreos.com/sealed-secrets-s8jxr created
```
* Create the subscription, the yaml definition can be found at ocp-gitops/Operators/SealedSecrets/subscription.yaml. 
   
   The subscription uses the alpha channel, the only one available at the moment, and a automatic update strategy.
```
$ oc apply -f subscription.yaml 
subscription.operators.coreos.com/sealed-secrets-operator-helm created
```
* Create the SealedSecretController CR, the yaml definition can be found at ocp-gitops/Operators/SealedSecrets/sealedsecretcontrollerCR.yaml
```
$ oc apply -f sealedsecretcontrollerCR.yaml
```
The operator, controller and the secret containing the x509 certificate use to encrypt/unencrypt the secrets are created in the sealed-secrets namespace:
```
$ oc get pods
NAME                                                    READY   STATUS    RESTARTS   AGE
sealed-secrets-operator-helm-76bd999f7f-84vvw           1/1     Running   0          13m
sealedsecretcontroller-sealed-secrets-8b848ccc7-4b996   1/1     Running   0          3m30s
$ oc get secrets|grep tls
sealed-secrets-key7kvdp                                 kubernetes.io/tls                     2      3m30s
```

### Certificate management

When the SealedSecretController is created, it will generate a new TLS certificate that is used to encrypt/decrypt secrets (sealing key).  The certificate is stored in a secret of type __kubernetes.io/tls__ in the same namespace and will be called something like __sealed-secrets-keyxff2g__.  As with any other normal secret, the certificate is not encrypted.

The sealing key is automatically renewed every 30 days. Which means a new sealing key is created and appended to the set of active sealing keys the controller can use to unseal Sealed Secret resources.  The latest sealing key is used to encrypt new secrets, the old sealing keys are kept so that secrets encrypted with them can still be used.

Since every SealedSecretController creates its own sealing key, this means that a secret encrypted in one cluster cannot be used/decrypted on a different cluster.  This is a security meassure to avoid revealing a secret by simply deploying it on a different cluster.  However this makes it difficult to have generic configuration definitions in git that can be applyed to any new installed cluster.  

To avoid this limitation it is possible to inject additional decrypting certificates to the SealedSecretController. So the certificate used to encrypt a secret in one cluster can be used to decrypt it in another:

* Extract the certificate used to encrypt the secret:
```
$ oc get secret sealed-secrets-keyxff2g -n sealed-secrets -o yaml > sealedkey-secret.yaml
```
* Import the secret in the new cluster, in the namespace where the SealedSecretController is running.  Make sure that the secret contains the label __sealedsecrets.bitnami.com/sealed-secrets-key=active__:
```
$ oc create -f sealedkey-secret.yaml
```
* Restart the SealedSecretController pod
```
$ oc delete pod -l app.kubernetes.io/instance=sealedsecretscontroller
```
The new pod logs will show the sealed keys it has picked up:
```
$ oc logs -l app.kubernetes.io/instance=sealedsecretscontroller
```

### Usage

The __kubeseal__ client is used to encrypt secrets, it needs access to the cluster so it can communicate with the SealedSecretController which is the one actually encrypting the secret with the current sealing key.  This means that previously we need to log into the cluster with the __oc login__ command.

Let's see how to encrypt a secret following an example.  

* Create a secret definition 

```
$ oc create secret generic petruxio --from-literal carta=pacio --from-literal duck=cover --dry-run=client -o yaml >secret1.yaml
```
Sealed secrets are by default bound to a particular namespace and the sealing key in a cluster to avoid the secret being deployed and revealed into another cluster.

If no namespace is specified in the yaml definition the SealedSecretController will assign the current active namespace when the secret is encrypted, to avoid this it is better to explictly specify the namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  creationTimestamp: null
  name: petruxio
  namespace: garrido
data:
  carta: cGFjaW8=
  nigro: bWFudGU=
```

* Encrypt the secret with kubeseal.  The name of the namespace where the SealedSecretController is running and the name of the service it provides need to be specified in the command line since the operator installation does not use the default ones:

```
$ kubeseal -o yaml --controller-name sealedsecretcontroller-sealed-secrets --controller-namespace sealed-secrets  < secret1.yaml >secret1-sealed.yaml
```
The resulting yaml file looks like:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: petruxio
  namespace: garrido
spec:
  encryptedData:
    carta: AgCCT9ThqudYoGn7cRq6HXldM6/6q/SMRRCX1+iqofktaqRBKc66Ok7snlmjUNZ1ft0zVmOLsYei7qUlkISnqlJcoCN6YCPkKq4QwdTPIp6NoevsYEzdaMFv3CAeXDkiFAxCfWUW9RkWf72lklIBcLvpMBP8jtvdlFwohUq7+svS7T2nEtxOjgffa0NwmT0le7La8qFEfbqWdPTA5wUMZBoGHzGYczLr5cG7+jSH5IvKmE6135owADjafGqMzZa3VeGi/3RzO7bw18Vq8Vh2xrcAXO+MB5x6OhZTesgb5SngcI0/l/ule0bH3q9U9lV91UrasszdWul+cjqiqhlLzdqaLZRfWdri9FzR0D8X3HjtikU989VCULX3lKygmRuPiLSGXkIe65cfbaPR9axrN3vS38BGlPHc8DWM+5I97UeHPu/BXY+kjD6ODXnopoJm3/DvcVe1ps+lFTzhauRTjKLbsz0zmcYngl1kYmCTQwQUptrCDx8JLtrR5+ekA2UyO88t4AtoLgPniiOaAiuO8qtMTnlwUY7xOf+aibQSMiH5c/qtRu726h8F1yXZX/wP4kvfhxIR62BecvH0EMkXZkGPVQWLS6STQBdkI6pb7DffuOFv3kK/fVmndk1anxkMijjU7yUbbUXyGsXq0bJIFBWVatOtnKMk/QDD6uabyQNoy6N0voz+p63ag33rOFVdXVY6a43bHg==
    nigro: AgAmo7xC32QsaibvF4oPervQ5zpNiCYHliA556oMQ5XUK4jeZpf0FP3rDTCzPHe72NH5agVT9esGwTt4jyoh/vpo8ZZgZww5XgCrb2b/TYREoC9MHtdIH8ShYZ235704flqYq1TWpCrj9kjSVuLC48z77t0XGKu9l1oAsGkq5lKE/WefJ9sbPylsz8ccS0AIPaOAECKmUingE1y5XA7yMeJhnqKdlHysmvXedGnhD6X180Gk/4rJdQZ46Xz24Ebd7zyUkGlnQOxh39A5M9Ob9iXWb2e49rWBq7fMf8hFCNaX+X9j20QPRjP/9QWTCToAcgKuTEHJ62sZIEbNcYHQiDmvrdFs90c7oc6X4qJ2GymW2nGRkzyfoTkrU0Wn5XMRjCzW2A2RX6zsiflsCJZdMDRmdBpyuu8VX3Pz3qIr3UEAxYm/79gDPsBPiPIw6DFu38U7EwBpTt9kkFIN+IUNOCakoNnCQYL+HAh+5xCKS0jMS34WXp2LITSCGnGrQt0y0OjC+1+nGw5USf0+71pN7YP5qrTgKRTOW3bt4LPxITUHU3T7YatATlIOEZSK1NFHY4lxp42q72VKwSQEM1bg/Tt29tIK9zhhSZlyp8JDm6yPzT2iLq9VziidwY1lyL8SZJ7RDzAYzcMkqXip766/EKYuOqFbU1+UP5pNZ00jQ2W1RJ/GwXNrCQ+lyZnYDsvG3nEeqGVK9w==
  template:
    data: null
    metadata:
      creationTimestamp: null
      name: petruxio
      namespace: garrido
```
   This secret can be pushed to a git repository, even a public one, because the values it contains are encrypted, the keys are still in clear text though.

* Create the secret in the Openshift cluster.  The secret needs to be created in the same namespace cotained in the definition.
```
$ oc new-project garrido
$ oc create -f secret1-sealed.yaml
```
Now the secret exists in the namespace garrido as a normal kubernetes secret:

