# Install TAP 1.5 through GitOps on TKGm

Official docs to [Install Tanzu Application Platform through Gitops with Secrets OPerationS (SOPS)
](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install-gitops-sops.html)

Choosing ESO vs SOPS, check this [docs page](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/install-gitops-reference.html#choosing-sops-or-eso). I this guide we will use SOPS.

## 0. Repo instructions

You'll need a `creds.sh` file located in a `local-config` at the root of this repo containing these three environment variables:
```bash
export TANZU_NET_USERNAME="" # Your TanzuNet username
export TANZU_NET_PASSWORD="" # Your TanzuNet password
export REGISTRY_USERNAME="tap-admin" # A Harbor Registry username
export REGISTRY_PASSWORD="" # A Harbor Registry password
export GITHUB_ACCESS_TOKEN="" # A GitHub Token
export AWS_ACCESS_KEY_ID="" # AWS Access Key ID for Acme DNS challenge if using Route53
export AWS_SECRET_ACCESS_KEY="" # AWS Secret Access Key for Acme DNS challenge if using Route53
```
Then run:
```bash
source ./local-config/creds.sh
```

## 1. Prerequisites

### 1.1 Install SOPS CLI
SOPS CLI to view and edit SOPS encrypted files. To install the SOPS CLI, see SOPS documentation in [GitHub](https://github.com/mozilla/sops/releases).
```bash
# For Mac ARM
curl -JOL https://github.com/mozilla/sops/releases/download/v3.7.3/sops-v3.7.3.darwin.arm64
sudo chmod +x sops-v3.7.3.darwin.arm64 && sudo mv sops-v3.7.3.darwin.arm64 /usr/local/bin/sops
```

### 1.2 Install Age CLI
Age CLI to create an ecryption key used to encrypt and decrypt sensitive data. To install the Age CLI, see age documentation in [GitHub](https://github.com/FiloSottile/age#installation).
```bash
# For Mac
brew install age
```

### 1.3 Prepare k8s cluster
In this guide we will deploy on Tanzu Kubernetes Grid multicloud on vSphere, which is one of the supported scenarios.

It's assumed some infra/k8s components are deployed and prepared beforehand:
- Existing TKGm 2.1.1 workload cluster deployed with minimum 16 CPUs across all available nodes (2 nodes w/ 8 CPUs per node better than 3 nodes w/ 5 CPUs per node due to DaemonSets), and with 8 GB memory and 100GB disk per node.
- No TKGm user-managed Packages pre installed in the workload clusters we are targeting for TAP deployment.
- AVI L4 is enabled in workload cluster (or other L4 solution like MetalLB or Native Cloud LB), but no AVI L7 enabled.
- This guide will not go over how to provision the above. [Oficial docs](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/using-tkg-21/workload-index.html) are available for readers to follow; and alternatively the [TKG-Lab](https://github.com/Tanzu-Solutions-Engineering/tkg-lab/) can also be used to achieve a similar outcome.

### 1.4 Get Harbor Ready

You need a registry, and you can use a Harbor installed as a Package in TKGm shared services cluster or standalone Harbor.
- Ensure that Harbor has a sufficiently large PV for the registry, more than 30GB, tested successfully with 50GB.
- You need Harbor to be deployed with certificate signed with a trusted CA or you will have to deploy the self-signed cert or untrusted CA in the worker nodes of the TAP cluster to allow TBS and any other app pulling images to trust your Harbor Registry.
  - As an example, the [TKG-Lab](https://github.com/Tanzu-Solutions-Engineering/tkg-lab/blob/main/scripts/generate-and-apply-harbor-yaml.sh#L34) deploys Harbor Package with Let's Encrypt based certs to have a CA that is trusted by containerd
  - Otherwise make sure you use an overlay with your Harbor CA or self-signed cert when you create the TKG cluster where TAP will run. Docs [here](https://docs.vmware.com/en/VMware-Tanzu-Kubernetes-Grid/2.1/using-tkg-21/workload-clusters-secret.html).
- Create a `tap-admin` user. Use the same password from the `REGISTRY_PASSWORD` env variable in your `.local-config/creds.sh` file
- Create a `tap` project.
- Assign `tap-admin` user to `tap` project as `Project Admin`.

### 1.5 Install Tanzu CLI plugins

Download the tanzu-framework bundle from TanzuNet and run these steps from the folder where the .tar file is to add plugins to your existing Tanzu CLI configuration:
```bash
mkdir -p tanzu-cli-tap
tar xvf tanzu-framework-darwin-amd64-v0.28.1.1.tar -C ./tanzu-cli-tap
tanzu plugin install --local ./tanzu-cli-tap/cli apps
tanzu plugin install --local ./tanzu-cli-tap/cli accelerator
tanzu plugin install --local ./tanzu-cli-tap/cli services
tanzu plugin install --local ./tanzu-cli-tap/cli insight
```

### 1.6 Other assumptions 
- DNS configuration can be set up to point wildcard fqdn (e.g: `*.tap.zora.tkg-vsp-lab.hyrulelab.com`) to contour-external envoy service VIP. This mapping will be done later in this guide.
- You have `imgpkg` CLI installed

## 2. Relocate Images to Harbor

```bash
# Set env vars
export REGISTRY_HOSTNAME="harbor.rito.tkg-vsp-lab.hyrulelab.com" # Replace with your Harbor Registry hostname 
export IMGPKG_REGISTRY_HOSTNAME_0=registry.tanzu.vmware.com
export IMGPKG_REGISTRY_USERNAME_0=$TANZU_NET_USERNAME
export IMGPKG_REGISTRY_PASSWORD_0=$TANZU_NET_PASSWORD
export IMGPKG_REGISTRY_HOSTNAME_1=$REGISTRY_HOSTNAME
export IMGPKG_REGISTRY_USERNAME_1=$REGISTRY_USERNAME
export IMGPKG_REGISTRY_PASSWORD_1=$REGISTRY_PASSWORD
export INSTALL_REGISTRY_USERNAME=$REGISTRY_USERNAME
export INSTALL_REGISTRY_PASSWORD=$REGISTRY_PASSWORD
export INSTALL_REGISTRY_HOSTNAME=$REGISTRY_HOSTNAME
export TAP_VERSION="1.5.0"
export INSTALL_REPO=tap
export TAP_CLUSTER_NAME="zora"

# login to registries
docker login $INSTALL_REGISTRY_HOSTNAME
docker login $IMGPKG_REGISTRY_HOSTNAME_0

# To query for the available versions of Tanzu Application Platform on VMWare Tanzu Network Registry, run:
imgpkg tag list -i ${IMGPKG_REGISTRY_HOSTNAME_0}/tanzu-application-platform/tap-packages | sort -V

# Relocate the images with the imgpkg CLI by running:
imgpkg copy -b ${IMGPKG_REGISTRY_HOSTNAME_0}/tanzu-application-platform/tap-packages:${TAP_VERSION} --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/tap-packages --include-non-distributable-layers
```

## 3. Prepare GitOps repo and Deploy Key

```bash
# Go to your loca-config folder in this repo
cd ./local-config
# Generate ssh key with Ed25519 algorithm
ssh-keygen -t ed25519 -C "jgonzalezagu@vmware.com"
# chose a new name like "tap_id_ed25519" for buth public and private keys to be generated in the folder
```

Now Create a new `Deploy Key` in the repository that you will use for this GitOps approach using the public key you just generated. Follow instructions [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys#deploy-keys)

## 4. Download and unpack Tanzu GitOps Reference Implementation (RI)

```bash
# Extract RI
tar xvf $HOME/Code/tap/1.5/tanzu-gitops-ri-*.tgz -C $HOME/Code/tap-gitops/
# Commit the initial state:
cd $HOME/Code/tap-gitops
git add . && git commit -m "Initialize Tanzu GitOps RI"
git push -u origin
```

## 5. Create full-cluster configuration

```bash
cd $HOME/Code/tap-gitops
./setup-repo.sh ${TAP_CLUSTER_NAME} sops
git add . && git commit -m "Add full-tap-cluster"
git push
```

## 6. Configure Tanzu Application Platform

### 6.1 Preparing sensitive Tanzu Application Platform values

```bash
# Go to the /local-config folder of this repo
cd $HOME/Code/tap-zone/
# Generate age key
age-keygen -o ./local-config/key.txt
export SOPS_AGE_RECIPIENTS=$(cat ./local-config/key.txt | grep "# public key: " | sed 's/# public key: //')
# Copy the sample sensitive values file to your private local-config
cp ./gitops/tap-sensitive-values.yaml ./local-config/tap-sensitive-values.yaml
# Inject sensitive values in sensitive values file (we use the same registry for tap images and app images, but different repos)
yq e -i '.tap_install.sensitive_values.shared.image_registry.username = strenv(INSTALL_REGISTRY_USERNAME)' ./local-config/tap-sensitive-values.yaml
yq e -i '.tap_install.sensitive_values.shared.image_registry.password = strenv(INSTALL_REGISTRY_PASSWORD)' ./local-config/tap-sensitive-values.yaml
# Encrypt sensitive values file
sops --encrypt ./local-config/tap-sensitive-values.yaml > ./local-config/tap-sensitive-values.sops.yaml
# Copy encrypted file to gitos repository
mv ./local-config/tap-sensitive-values.sops.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/values/
export SOPS_AGE_KEY_FILE=$HOME/Code/tap-zone/local-config/key.txt
```

### 6.2 Prepare Let's Encrypt Cluster Issuer

GoogleCloudDNS steps below. Replace your email and zone/project details in the env var exports. This also needs `gcloud-dns-credentials.json` in your `./local-config` folder.
```bash
cd $HOME/Code/tap-zone/
export ACME_EMAIL="jaguilar@pivotal.io"
export GCLOUD_PROJECT="fe-jaguilar"
export GCLOUD_CREDENTIAL_BASE64=$(base64 -i ./local-config/gcloud-dns-credentials.json)
# Prepare Cluster Issuer
cp ./gitops/gclouddns-lets-encrypt-cluster-issuer.yaml ./local-config/lets-encrypt-cluster-issuer.yaml
yq e -i '.spec.acme.email = env(ACME_EMAIL)' ./local-config/lets-encrypt-cluster-issuer.yaml
yq e -i '.spec.acme.solvers[0].dns01.cloudDNS.project = env(GCLOUD_PROJECT)' ./local-config/lets-encrypt-cluster-issuer.yaml
# Prepare Gcloud Credentials Secret
cp ./gitops/gclouddns-credential-secret.yaml ./local-config/gclouddns-credential-secret.yaml
yq e -i '.data."credentials.json" = env(GCLOUD_CREDENTIAL_BASE64)' ./local-config/gclouddns-credential-secret.yaml
# Encrypt sensitive data
sops --encrypt ./local-config/gclouddns-credential-secret.yaml > ./local-config/gclouddns-credential-secret.sops.yaml
# Copy files to gitos repository to the new dependant-resources folder to be processed after tap packages
mkdir -p $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/dependant-resources/
mv ./local-config/lets-encrypt-cluster-issuer.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/dependant-resources/
mv ./local-config/gclouddns-credential-secret.sops.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/dependant-resources/

# Create dependent-resources app in the repo to a general config folder that gets processed with tap packages
# This will create the App after TAP packages are deployed and sync the dependant-resources
mkdir -p $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/config/general
cp ./gitops/dependant-resources-app.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/config/general/
export DEPENDANT_RESOURCES_GIT_SUBPATH="clusters/${TAP_CLUSTER_NAME}/cluster-config/dependant-resources"
yq e -i '.spec.fetch[0].git.subPath = env(DEPENDANT_RESOURCES_GIT_SUBPATH)' $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/config/general/dependant-resources-app.yaml
```

### 6.2 Preparing non-sensitive Tanzu Application Platform values

Update the `./gitops/tap-values.yaml` file in this repo with the corect values for registries, repositories and other environment specific and non-sensitive parameters.

```bash
# copy tap-values file to gitops repo
cd $HOME/Code/tap-zone/
cp ./gitops/tap-values.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/values/tap-non-sensitive-values.yaml
# Change all properties for registries, repositories and other environment specific and non-sensitive parameters with the right values

# copy CNR overlay to gitops repo to a general config folder that gets processed with tap packages
mkdir -p $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/config/general
cp ./gitops/cnrs-network-config-overlay.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/config/general/cnrs-network-config-overlay.yaml

# Create namespace config for namespace provisioner to sync from gitops repo
mkdir -p $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/namespace-provisioner/namespaces
cp ./gitops/desired-namespace.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/namespace-provisioner/namespaces/desired-namespace.yaml
cp ./gitops/namespaces.yaml $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/cluster-config/namespace-provisioner/namespaces/namespaces.yaml
```

### 6.3 Generate Tanzu Application Platform installation and Tanzu Sync configuration

```bash
# Set env variables
export INSTALL_REGISTRY_HOSTNAME="harbor.rito.tkg-vsp-lab.hyrulelab.com" # replace with your registry hostname
export INSTALL_REGISTRY_USERNAME=$REGISTRY_USERNAME # you set these earlier via ./local-config/creds.sh
export INSTALL_REGISTRY_PASSWORD=$REGISTRY_PASSWORD # you set these earlier via ./local-config/creds.sh
export GIT_SSH_PRIVATE_KEY=$(cat $HOME/Code/tap-zone/local-config/tap_id_ed25519)
export GIT_KNOWN_HOSTS=$(ssh-keyscan github.com)
export SOPS_AGE_KEY=$(cat $HOME/Code/tap-zone/local-config/key.txt)
export TAP_PKGR_REPO=harbor.rito.tkg-vsp-lab.hyrulelab.com/tap/tap-packages # replace with your registry hostname

# Generate the Tanzu Application Platform install and the Tanzu Sync configuration files by using the provided script
cd $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/ # replace with the right cluster name (last folder)
./tanzu-sync/scripts/configure.sh
git add cluster-config/ tanzu-sync/
git commit -m "Configure install of TAP 1.5.0"
git push
```

### 6.4 Deploy Tanzu Sync

```bash
# change context to your cluster where you plan to deploy TAP
kubectl config use ${TAP_CLUSTER_NAME}-admin@${TAP_CLUSTER_NAME}
# Run deploy script from the gitops repo, cluster folder
cd $HOME/Code/tap-gitops/clusters/${TAP_CLUSTER_NAME}/
./tanzu-sync/scripts/deploy.sh
```

At this point TAP is installed in your cluster

## 7. Configure TAP post initial install

The following is a combination of addtional config steps that are not included in the gitops approach because:
- They are used more independently through demonstrations, or
- I'm still working on incorporating them to the gitops approach

### 7.1 Set up GitOps repository secret for Supply Chain

[Official docs to set up the GIt secret for GitOps](https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-git-auth.html)

Create token based secret
```bash
cd $HOME/Code/tap-zone/
cp ./config/git-secret.yaml ./local-config/git-secret.yaml
yq e -i '.stringData.username = strenv(GITHUB_USERNAME)' ./local-config/git-secret.yaml
yq e -i '.stringData.password = strenv(GITHUB_ACCESS_TOKEN)' ./local-config/git-secret.yaml
kubectl apply -f ./local-config/git-secret.yaml -n myapps
kubectl patch serviceaccount default -n myapps -p '{"secrets": [{"name": "git-secret"}]}'
```

### 7.2 Deploy Tekkton Pipline and Scan Policy`
```bash
# relocate gradle image
imgpkg copy -i gradle:latest --to-repo ${INSTALL_REGISTRY_HOSTNAME}/${INSTALL_REPO}/gradle --include-non-distributable-layers
# edit ./config/tekton-pipeline.yaml to use your relocated image instead
kubectl apply -f ./config/tekton-pipeline.yaml -n myapps
kubectl apply -f ./config/scan-policy.yaml -n myapps
```


## 8. Deploy Sample Apps

### 8.1 Deploy Tanzu Java Demo Application

Get tanzu-java-web-app from Accelerator. Replace the default value dev.local in the prefix for container image registry field with the registry in the form of `SERVER-NAME/REPO-NAME`. The `SERVER-NAME/REPO-NAME` must match what was specified for registry as part of the installation values for `ootb_supply_chain_basic`. Example: "harbor.rito.tkg-vsp-lab.hyrulelab.com/tap". This value will be injected in the `Tiltfile` used for the Inner Loop.
Unzip and push to a public Git repository, remember that registry url to use it in the "git" configuration of the `workload.yaml` used below:

```bash
tanzu apps workload create tanzu-java-web-app \
  -f ./config/tjwa-workload.yaml \
  -n myapps \
  --yes
```

Import workload backstage catalog info into TAP GUI
1. Log into TAP GUI using the url defined in your tap-values.yaml file. Example: http://tap-gui.tap.zora.tkg-vsp-lab.hyrulelab.com/
2. On the software catalog page click on the Register button
3. Use https://github.com/jaimegag/tanzu-java-web-app/blob/main/catalog/catalog-info.yaml for the url in the form
4. Follow the rest of the instructions in the wizard

### 8.2. Deploy .NET Core Steeltoe Application

Get weatherforecast-steeltoe from Accelerator.
Unzip and push to a public Git repository, remember that registry url to use it in the "git" configuration of the `workload.yaml` used below:

```bash
tanzu apps workload create weatherforecast \
  -f ./config/wfs-workload.yaml \
  -n myapps \
  --yes
```

Import workload backstage catalog info into TAP GUI
1. Log into TAP GUI using the url defined in your tap-values.yaml file. Example: http://tap-gui.tap.zora.tkg-vsp-lab.hyrulelab.com/
2. On the software catalog page click on the Register button
3. Use https://github.com/jaimegag/weatherforecast-steeltoe/blob/main/catalog/catalog-info.yaml for the url in the form
4. Follow the rest of the instructions in the wizard




Things to tweak/confirm/complete:
- git secret
  - Create via gitops 
- techdocs
  - Incldue in both values and sensitive values
- parametrize the non-sensitive values to make this reusable
  - Include PARAMS_YAML