# Out of the Box Supply Chain with testing on Jenkins


For the most part this guide follows the official docs [here]https://docs.vmware.com/en/VMware-Tanzu-Application-Platform/1.5/tap/scc-ootb-supply-chain-testing-with-jenkins.html)

## 1. Prepare a Jenkins setup

The (obvious) first step is to make sure you have a working Jenkins setup, and while many different setups would work well, here's a list of things to make sure you have propperly installed and configured to ensure success with the rest of this guide:
- Install the following plug-ins and dependencies: Pipeline, Maven, Pipeline Maven, Groovy and Groovy Pipeline
- Configure Maven: Go to `Manage Jenkins > Global Tool Configuration > Maven` and configure Maven installation, easier to install from Apache, name it (e.g: `maven-3.9.2`)
- If you are using a Java17 app (like we do in this lab) you need to also configure JDK 17. Get a binary (e.g: tar.gz) URL and name it (e.g: `java-17`)
- Create New Item type Pipeline and name it (e.g: `tap-java-test`). Make sure its configuration includes:
  - Click on `This project is parametrized` and:
    - Add String Parameter: GIT_URL
    - Add String Parameter: SOURCE_REVISION
  - Add Pipeline Script > Copy contents from the `./config/jenkins-job-script.py` file in this repo. Make sure the `tools` reference the right maven and jdk names defined earlier


## 2. Test App with Jenkins integration

```bash
# Create Jenkins Secret with tthe user/password of your Jenkins Instance that you used to create the Jenkins Job in the previous step.
# Edit with your Jenkins user credentials before applying
kubectl apply -f ./config/jenkins-secret.yaml -n myapps

# Create Jenkins Pipeline
kubectl apply -f ./config/tekton-pipeline-jenkins.yaml

# Create Workload
tanzu apps workload create tanzu-java-jenkins \
  -f ./config/tjwa-jenkins-workload.yaml \
  -n myapps \
  --yes
```
