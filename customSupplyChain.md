# Custom Supply Chain with a final step calling Jenkins

In this lab we will provide a step by step guide on how to create a Custom Supply Chain starting from the OOTB `source-test-scan-to-url` Supply Chain adding a final step after the Config Writer to call a Jenkins Job passing over some important conffiguration details collected through the building process.

### 1. Create new Template for the final step

We need a ClusterTemplate that stamps a resource that can execute a Tekton ClusterTask which is already equipped to make calls to a Jenkins API. To do this in a simpler way we will stamp a PipelineRun (inmutable) resource using tekton lifecycle. This PipelineRun resource will use a Tekton Pipeline which will be the one executing the ClusterTask. 

We have prepared this in the `./config/jenkins-final-template.yaml` in this repo.
- We made sure the PipelineRun has a `generateName`. Reason: every object templated with lifecycle: tekton should specify a generateName rather than a name; because when our inputs change, we’re not going to update the templated object, instead we’re going to create an entirely new object, and of course that new object can’t have the same hardcoded name. 
- We also have to set the ClusterTemplate `singleConditionType: Succeeded`. Reason: We need a healthRule that matches with a successful execution and status of the stamped object

We have also prepared both the Pipeline resource and Cluster Task including all parameters that we want to pass to the Jenkins job at this stage of the Supply Chain: All gitops config parameters used by the Config Writer: `./config/tekton-pipeline-jenkins-final.yaml` and `./config/jenkins-final-task.yaml` respectively.
> Note: To pass all the gitops parameters all the way through the PipelineRun > TaskRun and into the call to the Jenkins Job we ended up needing to add all of them as a json array in the `job-param` that is passed as an argument when calling the jenkins job. That ensures they are reaching the job and can be read from there. Previous attempts to use other params that are passed in the `env` like the `source-url` are not working, as it seems that only special `env` params like `source-url` and `source-revision` are properly processed by the Tekton ClusterTask to reach the Jenkins job.

Deploy all in your Build/Full Cluster
```bash
kubectl apply -f ./config/jenkins-final-task.yaml
kubectl apply -f ./config/tekton-pipeline-jenkins-final.yaml
kubectl apply -f ./config/jenkins-final-template.yaml
```

### 2. Use a ClusterConfigTemplate config-writer

In our new Custom Supply Chain the last step before the Delivery is going to be our new step where we call a Jenkins Job. We want this step to be executed after the config-writer and we want to actually use some of the configuration that the config-writer uses. To let Cartographer know about this the best thing would be to make sure that the config-writer is a `ClusterConfigTemplate` that writes an output we can read/check, instead of a `ClusterTemplate` which is what we get in the OOTB Supply Chains.

We have prepared this as a new template because we want to use it this way only in our Custom Supply Chain: `./config/config-writer-config-template.yaml`.
- We need a new name to deifferentiate it from the other `config-writer`: `config-writer-config-template`
- We need blocking behavior to prevent propagation of values down the supply chain until the config-writer object fulfills its health rules, so we change the `lifecycle` to `immutable`
- We want to pass along the input as output so we set the `configPath` to `.spec.inputs.params`, so that our final step can get that config.

Apply the new `config-writer-config-template`:
```bash
kubectl apply -f ./config/config-writer-config-template.yaml
```

Alternatively you could apply this changes to the existing `config-writer` via overlay, considering this will affect all OOTB Supply Chains already using that `config-writer`. We prepared this overlay in the `./config/ootb-templates-overlay.yaml` in this repo. To use it you need to apply it and then configure it in the `package_overlays` for `ootb-templates` in your `tap-values.yaml`:
```bash
package_overlays:
  - name: "ootb-templates"
    secrets: 
    - name: "ootb-templates-overlay"
```

### 3. Create Custom Supply Chain

Due to the very small changes we need to make we started from the `source-test-scan-to-url`. Then made a few changes:
- Change the `selectorMatchExpresion` to list new workload types to apply this SupplyChain to.
- Change the `config-writer` resource to reference a `ClusterConfigTemplate` with the name we defined: `config-writer-config-template`
- Add a new `jenkins-final` resource after the `config-writer` one referencing our new `jenkins-final-template` of kind `ClusterTemplate`, using the `config-writer` config.
All this is prepared in the `./config/source-test-scan-jenkins-to-url.yaml` file in this repo.

Apply it:
```bash
kubectl apply -f ./config/source-test-scan-jenkins-to-url.yaml

# confirm the supply chain is ready
tanzu apps cluster-supply-chain get source-test-scan-jenkins-to-url 
# should return something like this:                                                                                  ✔
# ---
# # source-test-scan-jenkins-to-url: Ready
# ---
# Supply Chain Selectors
#    TYPE          KEY                                   OPERATOR   VALUE
#    labels        apps.tanzu.vmware.com/has-tests                  true
#    expressions   apps.tanzu.vmware.com/workload-type   In         web-j
#    expressions   apps.tanzu.vmware.com/workload-type   In         server-j
#    expressions   apps.tanzu.vmware.com/workload-type   In         worker-j
```

### 4. Prepare a Jenkins Job in your Jenkins instance

You need a Jenkins (Pipeline) Job to be able to run when invoked from our new Supply Chain step.

The (obvious) first step is to make sure you have a working Jenkins setup, and while many different setups would work well, here's a list of things to make sure you have propperly installed and configured to ensure success with the rest of this guide:
- Install the following plug-ins and dependencies: Pipeline, Maven, Pipeline Maven, Groovy and Groovy Pipeline
- Create New Item type Pipeline and name it (e.g: `tap-finalizer`). Make sure its configuration includes:
  - Click on `This project is parametrized` and:
    - Add String Parameter: git_repository
    - Add String Parameter: git_branch
    - Add String Parameter: sub_path
  - Add Pipeline Script > Copy contents from the `./config/jenkins-final-job-script.py` file in this repo.

### 5. Deploy Workloads that use this supply chain

We have prepared 2 Workloads in this repo with the right `workload-type` and params to be able to be processed by the new Supply Chain:
```bash
tanzu apps workload create csharp-rest-service-x2 \
  -f ./config/crwa-x2-workload.yaml \
  -n myapps \
  --yes

# tanzu apps workload create tanzu-java-jenkins-x2 \
#   -f /config/tjwa-jenkins-x2-workload.yaml \
#   -n myapps \
#   --yes
```

Your application should be processed by your Custom Supply Chain running the new extra step at the end, and TAP GUI should be able to display it as well:

<img src="./images/supply-chain-end.png" width="800"><br>

In your Jenkins instance you should be able to see the last job execution and in the Output all the gitops information we passed to the Jenkins job:

<img src="./images/jenkins-final-job.png" width="800"><br>

