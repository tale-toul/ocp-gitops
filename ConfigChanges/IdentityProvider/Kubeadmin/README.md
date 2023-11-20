# REMOVING THE KUBEADMIN USER

To remove the kubeadmin user the [official instructions](https://docs.openshift.com/container-platform/4.7/authentication/remove-kubeadmin.html) say that the kubeadmin secret must be deleted.  ArgoCD does not have an easy way to delete a resource that is not previously under its controll, however what the official documentation does not say is that you don't really need to delete the secret, it is enough to overwrite the kubeadmin key.

The secret in this repo overwrites the original password hash with the word "VOID", this disables the kubeadmin user, and renders it non recoverable.

The secret needs not be encrypted because the key contents are of no use.
