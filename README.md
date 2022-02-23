# tanzu-ansible

# Ansible jobs for Tanzu

The playbooks, roles, and tasks attempt to follow the steps outlined in 
the Tanzu docs.


## Assumptions

- ansible installed
- git installed
- location to pull binaries via http
- docker already installed
- cluster configs are located in ~/.config/tanzu/clusters/$CLUSTER_NAME


## Install

    mkdir -p ~/.config/tanzu
    git clone --depth=1 gitea@ralph.simpson:mtritab/tanzu-ansible.git ~/.config/tanzu/ansible


## Binaries Setup

The cli-setup playbook install the binaries necessary to intall tanzu, packages, and tmc.
It will install the binaries in the USER's ~/.local/bin directory. If shared binaries are required
OS directory, change USER: root, and BIN_DIR: /usr/local/bin, and modify the playbook by adding sudo
to the install command.

- DOES NOT install docker
- binaries MUSt be staged where they can be downloaded via http

The following variables must be defined in group_vars/all/cli.yaml

    BIN_DIR: ~/.local/bin
    USER: myuserid

The following variables must be defined in group_vars/linx.yaml

    OS_1: linux
    TANZU_CLI_URL: http://example.com/binaries/cli/raw/branch/master/tanzu-cli-bundle-linux-amd64.tar
    KUBECTL_CLI_URL: http://example.com/binaries/cli/raw/branch/master/kubectl-linux-v1.21.2+vmware.1.gz
    YQ_CLI_URL: https://github.com/mikefarah/yq/releases/download/v4.18.1/yq_linux_amd6

To run playbook

    cd ~/.config/tanzu/ansible
    ansible-playbook cli-setup.yaml
or

    ansible-playbook ~/.config/tanzu/ansible/cli-setup.yaml

If a playbook fails, you can can start where it left off using the ansible --start-at-task flag.

    ansible-playbook cli-setup.yaml --start-at-task="carvel-install"

Individual steps in the playbook via ansible tags. Each task/role in the playbook is tagged with the same name.

    ansible-playbook cli-setup.yaml --tags "yq-install"


## Management Cluster Standup

The management-cluster-standup playbook is a collection of roles which create a management
cluster and registers it with TMC. If TMC prereqs haven't been set-up prior, remove the tmc role from the playbook.

**Currently, all tmc roles require the tmc login command to have been run manually before being run.**

- The playbook requires one variable on the cmd line to pass in the management cluster name.
- The management cluster config file must be located in ~/.config/tanzu/clusters/$CLUSTER_NAME/cluster-config.yaml
- An ansible config file for each cluster muste be located in ~/.config/tanzu/clusters/$CLUSTER_NAME/ansible-config.yaml

The playbook contains a role that exports the kubeconfig to ~/.config/tanzu/clusters/$CLUSTER_NAME/config. This KUBECONFIG
is exported and required for all subsequent tasks performed against the cluster.

To run playbook

    cd ~/.config/tanzu/ansible
    ansible-playbook management-cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-man"
or

    ansible-playbook ~/.config/tanzu/ansible/management-cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-man"

If a playbook fails, you can can start where it left off using the ansible --start-at-task flag.

    ansible-playbook management-cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-man" --start-at-task="export-management-kubeconfig"

Individual steps in the playbook via ansible tags. Each task/role in the playbook is tagged with the same name.

    ansible-playbook management-cluster-standup.yaml ---extra-vars "CLUSTER_NAME=tkgv1-man" --tags="tmc-mc-register"



## Cluster Standup

The cluster-standup playbook is a collection of roles which create a cluster and can install tanzu
user managed packages. If TMC prereqs haven't been set-up prior, remove the tmc role from the playook.

**Currently, all tmc roles require the tmc login command to have been run manually before being run.**
**The tmc-cluster-integration role requires pre-existing to.json and tsm.json manifests with the correct values.**

- The playbook requires one variable on the cmd line to pass in the cluster name.
- The cluster config file must be located in ~/.config/tanzu/clusters/$CLUSTER_NAME/cluster-config.yaml
- An ansible config file for each cluster must be located in ~/.config/tanzu/clusters/$CLUSTER_NAME/ansible-config.yaml

The playbook contains a role that exports the kubeconfig to ~/.config/tanzu/clusters/$CLUSTER_NAME/config. This KUBECONFIG
is exported and required for all subsequent tasks performed against the cluster.

To run playbook

    cd ~/.config/tanzu/ansible
    ansible-playbook cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-cluster2"
or

    ansible-playbook ~/.config/tanzu/ansible/cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-cluster2"

If the playbook fails, you can can start where it left off using the ansible --start-at-task flag.

    ansible-playbook cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-cluster2" --start-at-task="package-install-fluent-bit"

Individual steps in the playbook via ansible tags. Each task/role in the playbook is tagged with the same name.

    ansible-playbook cluster-standup.yaml --extra-vars "CLUSTER_NAME=tkgv1-cluster2" --tags="install-cluster-issuer"


## Package Install role

- The package-install role requires two ansible vars - PACKAGE (the short package name) and PACKAGE_NS. 
- The two vars can be defined in the playbook or reference a variable defined in ansible-config.yaml

If the package is already installed, the package-install role will exit.
If ~/.config/tanzu/clusters/$CLUSTER_NAME/$PACKAGE/$PACKAGE-data-values.yaml exists, the role will use the existing file.
If the values file does not exist, the role will download the latest package image, extract the default values file, and
copy it to ~/.config/tanzu/clusters/$CLUSTER_NAME/$PACKAGE/$PACKAGE-data-values.yaml. The role will then use the new file when installing the package.


## Cluster Issuer

Installs a self-signed CA.

- Requires ~/.config/tanzu/clusters/$CLUSTER_NAME/tanzu-issuer.yaml. It should contain two manifests, a cluster issuer
and tls secret both named tanzu-issuer


## ArgoCD Install

The install-argocd role deploys example argo manifests and is not production ready.

- Requires cert-manager, the tanzu-issuer, and contour to be installed.
- Requires cluster subdir argocd containing install-with-argo-cd.yaml, argocd-cert.yaml, and argocd-httpproxy.yaml
