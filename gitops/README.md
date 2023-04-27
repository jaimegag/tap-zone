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
export AWS_ACCESS_KEY_ID="" # AWS Access Key ID for Acme DNS challenge
export AWS_SECRET_ACCESS_KEY="" # AWS Secret Access Key for Acme DNS challenge
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
export IMGPKG_REGISTRY_HOSTNAME_0=registry.tanzu.vmware.com
export IMGPKG_REGISTRY_USERNAME_0=$TANZU_NET_USERNAME
export IMGPKG_REGISTRY_PASSWORD_0=$TANZU_NET_PASSWORD
export IMGPKG_REGISTRY_HOSTNAME_1="harbor.rito.tkg-vsp-lab.hyrulelab.com" # replace with your registry
export IMGPKG_REGISTRY_USERNAME_1=$REGISTRY_USERNAME
export IMGPKG_REGISTRY_PASSWORD_1=$REGISTRY_PASSWORD
export INSTALL_REGISTRY_USERNAME=$REGISTRY_USERNAME
export INSTALL_REGISTRY_PASSWORD=$REGISTRY_PASSWORD
export INSTALL_REGISTRY_HOSTNAME="harbor.rito.tkg-vsp-lab.hyrulelab.com" # replace with your registry
export TAP_VERSION="1.5.0"
export INSTALL_REPO=tap

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
./setup-repo.sh zora sops
git add . && git commit -m "Add full-tap-cluster"
git push
```

## 6. Configure Tanzu Application Platform

### 6.1 Preparing sensitive Tanzu Application Platform values

```bash
mkdir -p $HOME/Code/tap-zone/local-config/tmp-enc
chmod 700 $HOME/Code/tap-zone/local-config/tmp-enc
cd $HOME/Code/tap-zone/local-config/tmp-enc
age-keygen -o key.txt
export SOPS_AGE_RECIPIENTS=$(cat key.txt | grep "# public key: " | sed 's/# public key: //')
cd $HOME/Code/tap-zone/
sops --encrypt ./gitops/tap-sensitive-values-baseline.yaml > ./local-config/tmp-enc/tap-sensitive-values.sops.yaml
cd $HOME/Code/tap-zone/local-config/tmp-enc
mv tap-sensitive-values.sops.yaml $HOME/Code/tap-gitops/clusters/zora/cluster-config/values/
cp key.txt ../ 
export SOPS_AGE_KEY_FILE=$HOME/Code/tap-zone/local-config/key.txt
cd $HOME/Code/tap-zone/
rm -rf $HOME/Code/tap-zone/local-config/tmp-enc
```

### 6.2 Preparing non-sensitive Tanzu Application Platform values

```bash

```