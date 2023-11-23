# GITOPS CONFIGURATION FOR OPENSHIFT 4 CLUSTERS

## Setting up the demo

This instructions have been tested from a RHEL 8.7 host against an OCP 4.14 cluter.

Access as cluster admin to an Openshift 4 cluster is required.

Ansible is required.  The following command install ansible and the required associated packages:
```
$ sudo dnf install ansible
```
In RHEL 8.7 system that has been used, ansible uses python 3.9 but most of the python packages are for version 3.6
```
$ ansible --version
ansible [core 2.13.3]
...
  python version = 3.9.13 (main, Nov  9 2022, 13:16:24) [GCC 8.5.0 20210514 (Red Hat 8.5.0-15)]

$ ls -F /usr/lib/python3.6/site-packages/
authselect/                      file_magic-0.3.0-py3.6.egg-info/  jsonschema/                        pycparser-2.14-py3.6.egg-info/             sepolicy/
babel/                           idna/                             jsonschema-2.6.0-py3.6.egg-info/   pyinotify-0.9.6-py3.6.egg-info/            sepolicy-1.1-py3.6.egg-info
Babel-2.5.1-py3.6.egg-info/      idna-2.5-py3.6.egg-info/          jwt/                               pyinotify.py                               serial/
...
```

Install the kubernetes python library. This is needed by the k8s ansible module family:
```
$ sudo dnf install python3-kubernetes
```

The latest version of the code is in the git branch **arcentric**:
```
$ git clone https://github.com/tale-toul/ocp-gitops.git 
$ git checkout arcentric
$ git pull origin arcentric
```

The playbook needs to know the API entrypoint of the Openshift cluter.  Assign the value to the ansible variable **api_entrypoint**.  The value can be obtained running the following command from an already logged in session in the OCP cluster: 
```
$ oc whoami --show-server
https://api.cluster-lh48t.lh48t.sandbox180.opentlc.com:6443
```

In order to stablish secure connections with the API, the k8s modules need to have the CA certificates used by the cluster.  Obtain the CA bundle running the following command:
```
$ oc rsh -n openshift-authentication <oauth-openshift-pod> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > api-ca.crt
```
   Then define the variable **api_ca_cert** with the absolute or relative path to the file.

The playbook requires cluster admin credentials to make changes to the cluster.  The credentials are expected to be found in the file **Ansible/group_vars/user_credentials.vault**.  

The credentials must be in the form of a authentication token, like in the following example
```
token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImVVMFJ6Mjd6c3BHcmZ4a29sRURGekdQZ2dUSDNiRkItZjhLRE1vZzFFcHcifQ.eyJpc3M....
```
The above file should be encrypted with ansible-vault:
```
$ cd Ansible
$ echo "token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImVVMFJ6Mjd6c....." > group_vars/user_credentials.vault
$ ansible-vault encrypt --vault-id vault-id group_vars/user_credentials.vault
```
    The vault password or vault id must be passed to the playbook.

Review the file **Ansible/group_vars/all** and enable/disable the changes you want applied to the cluster.  

If you enable the deployment of the bgd application, go to the file **Applications/bgd/overlays/cluster-5cvx2/ingress-patch.yaml** and adapt the two values in that patch to reflect the DNS domain of your cluster.  It is recommended to create a new overlay directory for the cluster:
```
$ mkdir Applications/bgd/overlays/cluster-lh48t
$ cd Applications/bgd/overlays/cluster-lh48t                                                                                                                    
$ cp ../cluster-5cvx2/* .
$ oc whoami --show-console                                                                                                                                   
https://console-openshift-console.apps.cluster-lh48t.lh48t.sandbox180.opentlc.com

- op: replace
  path: /spec/rules/0/host
  value: bgd.apps.cluster-lh48t.lh48t.sandbox180.opentlc.com
- op: replace
  path: /spec/tls/0/hosts/0
  value: bgd.apps.cluster-lh48t.lh48t.sandbox180.opentlc.com
```
If you don't change those values, the application will be deployed nontheless.  

If you change the values, you need to modify the directory reference in the file **Ansible/roles/bgd_app/files/App-bgd_app.yaml** 
```
apiVersion: argoproj.io/v1alpha1
kind: Application
...
    path: Applications/bgd/overlays/cluster-lh48t/
    repoURL: https://github.com/tale-toul/ocp-gitops/
    targetRevision: arcentric
```
Push the changes to the git repository, for that you will have to fork the repository:
```
$ git add Ansible/roles/bgd_app/files/App-bgd_app.yaml
$ git add Applications/bgd/overlays/cluster-lh48t/
$ git commit -sv
$ git push origin arcentric
```

Run the ansible playbook with a command like:
```
$ cd Ansible
$ ansible-playbook -vvv add-config.yaml -e api_entrypoint="https://api.cluster-lh48t.lh48t.sandbox180.opentlc.com:6443" -e api_ca_cert=api-ca.crt --vault-id vault-id
```

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

## Ansible automation

An ansible playbook exists in the Ansible directory mostly as a reference to see how to install the components required to have a fully functional Gitops service. 

The playbook uses the k8s modules (k8s; k8s_auth; k8s_info), these modules require the __python2-openshift__ package and ansible 2.9 to be installed in the system running the playbook.

The following ansible variables need to be defined in order to successfully run the playbook:

* The playbook needs to know the API entrypoint of the Openshift cluter.  Assing the value to the ansible variable **api_entrypoint**.  The value can be obtained running the following command from an already logged in session in the OCP cluster: 
```
$ oc whoami --show-server
https://master.ocpext.example.com:443
```
* In order to stablish secure connections with the API, the k8s modules need to have the CA certificates used by the cluster.  Obtain the CA bundle running the following command:
```
$ oc rsh -n openshift-authentication <oauth-openshift-pod> cat /run/secrets/kubernetes.io/serviceaccount/ca.crt > ingress-ca.crt
```
   Then define the variable **api_ca_cert** with the absolute or relative path to the file.

* The playbook requires the credentials of an admin user because it makes privileges changes to the cluster like installing operators.  The credentials are expected to be found in a file called **user_credentials.vault** in the same directory where the playbook is located.  The format of the file is:
```
ocp_user: kubeadmin
ocp_pass: <password>
```
This file should be encrypted with ansible-vault:
```
$ ansible-vault encrypt user_credentials.vault
```
    The vault password must be passed to the playbook.

The playbook imports any sealed keys found in the directory __Operators/SealedSecrets/sealedKeys__ all these keys should be encrypted with ansible-vault using the same password used for the credentials file above.

An example command to run the playbook is:
```
$ ansible-playbook  add-config.yaml -e api_entrypoint="https://api.magne.jjerezro.emeatam.support:6443" -e api_ca_cert=ingress-ca.crt --vault-id vault-id
```


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
The operator, controller, and secret containing the x509 certificate used to encrypt/unencrypt the secrets are created in the sealed-secrets namespace:
```
$ oc get pods
NAME                                                    READY   STATUS    RESTARTS   AGE
sealed-secrets-operator-helm-76bd999f7f-84vvw           1/1     Running   0          13m
sealedsecretcontroller-sealed-secrets-8b848ccc7-4b996   1/1     Running   0          3m30s
$ oc get secrets|grep tls
sealed-secrets-key7kvdp                                 kubernetes.io/tls                     2      3m30s
```

### Certificate management

When the SealedSecretController is created, it will generate a new TLS certificate (sealing key) that is used to encrypt/decrypt secrets.  The certificate is stored in a secret of type __kubernetes.io/tls__ and will be named like __sealed-secrets-keyxff2g__.  As with any other normal secret, the certificate is not encrypted.

The sealing key is automatically renewed every 30 days, the latest key is used to encrypt new Sealed Secrets, the older keys are kept so existing Sealed Secrets can still be decrypted.

Since every SealedSecretController creates its own sealing key, this means that a secret encrypted in one cluster cannot be used/decrypted on a different cluster.  This is a security meassure to avoid revealing a secret by simply deploying it on a different cluster.  However this makes it difficult to have generic configuration definitions in git that can be applyed to any new installed cluster.  

To avoid the above limitation it is possible to inject additional decrypting certificates to the SealedSecretController. So the certificate used to encrypt a secret in one cluster can be used to decrypt it on a different one. Here is how to do it:

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
$ oc delete pod -l app.kubernetes.io/instance=sealedsecretcontroller -n sealed-secrets
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
Sealed secrets are by default bound to a particular namespace and a sealing key to avoid the secret being deployed and therefore revealed into another namespace or cluster.

If no namespace is specified in the yaml definition the SealedSecretController will assign the current active namespace when the secret is encrypted, to avoid this it is better to explictly specify the namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
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
    carta: AgCCT9ThqudYoGn7c...==
    nigro: AgAmo7xC32QsaibvF...==
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

