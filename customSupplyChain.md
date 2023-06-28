# Custom Supply Chain with a final step calling Jenkins

In this lab we will provide a step by step guide on how to create a Custom Supply Chain starting from the OOTB `source-test-scan-to-url` Supply Chain adding a final step after the Config Writer to call a Jenkins Job passing over some important conffiguration details collected through the building process.

### 1. Create new Template for the final step

We need a ClusterTemplate that stamps a resource that can execute a Tekton ClusterTask which is already equipped to make calls to a Jenkins API. To do this in a simpler way we will stamp a PipelineRun (inmutable) resource using tekton lifecycle. This PipelineRun resource will use a Tekton Pipeline which will be the one executing the ClusterTask. 

We have prepared this in the `./config/jenkins-final-template.yaml` in this repo.
- We made sure the PipelineRun has a `generateName`. Reason: every object templated with lifecycle: tekton should specify a generateName rather than a name; because when our inputs change, we’re not going to update the templated object, instead we’re going to create an entirely new object, and of course that new object can’t have the same hardcoded name. 
- We also have to set the ClusterTemplate `singleConditionType: Succeeded`. Reason: We need a healthRule that matches with a successful execution and status of the stamped object

We have also prepared both the Pipeline resource and Cluster Task including all parameters that we want to pass to the Jenkins job at this stage of the Supply Chain: All gitops config parameters used by the Config Writer: `./config/tekton-pipeline-jenkins-final.yaml` and `./config/jenkins-final-task.yaml` respectively.

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

### 4. Deploy Workloads that use this supply chain

Both Workloads have been prepared in this repo:
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