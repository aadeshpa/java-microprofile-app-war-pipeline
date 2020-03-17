# How to build and deploy a Java application using Maven and Tekton Pipelines to an external Virtual machine 

 This repository has a Java Microprofile microservice application with `maven` dependency. The application is configured using pom files so it can be build with Maven. This repository also has the `Tekton pipelines` files which will show how you can build this application using CI/CD technology of [Tekton Pipelines](https://github.com/tektoncd/pipeline). You will be able to deploy this application using the tekton pipelines to an external virtual machine which has `OpenLiberty` server already installed.
 
 ## Intended audience
 
 This guide is useful for someone who is interested in using `Tekton Pipelines` to automatically build their application using `Maven` tool into a `war` file , copy the `war` file to the external virtual machine, and then deploy it periodically on that external virtual machine in the `OpenLiberty` application server in a CI/CD format.
 
 
## Prerequisites

- [Openshift](https://www.openshift.com/products/container-platform) or [Minishift](https://www.redhat.com/sysadmin/learn-openshift-minishift)
- [OC cli](https://docs.openshift.com/container-platform/4.2/cli_reference/openshift_cli/getting-started-cli.html#cli-installing-cli_cli-developer-commands)
- [Tekton pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/install.md#installing-tekton-pipelines-on-openshift)
- External Virtual machine with [OpenLiberty](https://openliberty.io/downloads/) installed

## Steps to follow for pipelines setup
NOTE : Here assumption is you have setup your OCP cluster or Minishift cluster and you are ready to login to the cluster

1. You should have a external virtual machine with [OpenLiberty](https://openliberty.io/downloads/) installed in it

Note down the virtual machine `host ipaddress ` and `host password` which will be needed to be updated in the tekton task files explained below.

2. Create a `project` in the openshift cluster

```
  oc new-project <project-name>
  ex:
  oc new-project deployment-learning
```
Move to that new project namespace
```
  oc project deployment learning
```

3. Create a persistent volume (pv)
Pipelines require a configured volume that is used by the framework to share data across tasks. The pipeline run creates a Persistent Volume Claim (PVC) with a requirement for five GB of persistent volume.

Login to your cluster.  For example ,

```
oc login <master node IP>:8443
```

Clone this [repository](https://github.ibm.com/aadeshpa/java-microprofile-app-war-pipeline.git) and apply `pv.yaml` from it.

```
git clone https://github.ibm.com/aadeshpa/java-microprofile-app-war-pipeline.git
cd .pipelineruns/
oc apply -f pv.yaml -n deployment-learning
```

verify the pv `manual-pipeline-run-pvc` is created
```
oc get pv
```

4. Create the pipelineresource for Git source.

The pipeline provided in this repository refers to a `git-source` tekton pipeline resource. To understand more about tekton pipelineresource please go [here](https://github.com/tektoncd/pipeline/blob/master/docs/resources.md#git-resource)

We have provided with the pipelineresource file `java-mp-war-pipeline-resources.yaml` where you need to point to the github source and apply it as below

```
cd .pipelineruns/
oc apply -f java-mp-war-pipeline-resources.yaml
```
Verify the `git-source` is created
```
oc get source git-source -o yaml
```

5. Applying the tekton task file

Tekton pipelines has smallest unit as [steps](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md#steps) and these steps are used under the CRD(Custom resource definition) called [Tasks](https://github.com/tektoncd/pipeline/blob/master/docs/tasks.md#tasks)

We have provided the below task file for the pipeline for build the java microprofile app into war file and deploy it to external vm openliberty server. You need to activate them before you run pipelinerun.

Before you activate the task `java-mp-war-build-deploy-task.yaml` you need to edit it as below
```
cd ./pipelines
vi java-mp-war-build-deploy-task.yaml
```
In the `build` and `deploy` step under the task you will find the `env` variables `vmipaddess`  and `vmpassword` which you need to update from step 1 where you noted the virtual machine ipaddress and password.

Now you can activate the task
```
cd ./pipelines
oc apply -f java-mp-war-build-deploy-task.yaml -n deployment-learning
```
Verify that the task is created
```
oc get task javamp-war-build-deploy-task -n deployment-learning -o yaml
```

6. Activating the pipelines file

Tekton pipelines has the CRD called [pipelines](https://github.com/tektoncd/pipeline/blob/master/docs/pipelines.md#pipelines) which consists of tekton tasks.

Activate the pipeline file given in this repository that points to the task you activated above.

```
cd ./pipelines
oc apply -f java-mp-war-build-deploy-task-pl.yaml -n deployment-learning

```
Verify that the pipeline was created
```
oc get pipelines javamp-build-deploy-pl
```

## Steps to follow for pipelinerun 

### Activate the pipelinerun file to start the pipelinerun

As per above steps, now you are ready with the `pv`, `pipelineresource`, `task` and `pipeline` files activated. Now you can activate the pipelinerun file. To read more on tekton pipelineruns file please go [here](https://github.com/tektoncd/pipeline/blob/master/docs/pipelineruns.md#pipelineruns)

Before activating the pipelinerun file please verify in the cluster if you have [serviceaccount](https://docs.openshift.com/container-platform/3.6/dev_guide/service_accounts.html#overview) named `default`, since we have used it in the `javamp-war-b-d-manual-pipeline-run-template.yaml` file.

```
oc get sa default -o yaml
```
If you do not have one, create a [serviceaccount](https://docs.openshift.com/container-platform/3.6/dev_guide/service_accounts.html#overview) and add it to the pipelinerun file `javamp-war-b-d-manual-pipeline-run-template.yaml` before activating it.

Activate the pipelinerun file

```
cd .pipelineruns/
oc apply -f javamp-war-b-d-manual-pipeline-run-template.yaml

```

### Access the deployed application

Once above pipelinerun is successfull, you can check the logs to find the application was deployed in the virtual machine and you can access the application via the url `{your-vm-ipaddress}:9080` on the browser.
 

### Troubleshooting logs

Once you applied the pipelinerun file from above step, you can check the status and logs of the pod as below

```
oc get pods
NAME                                                              READY   STATUS      RESTARTS   AGE
javamp-war-b-d-manual-pipeline-run-build-deploy-task-d6dz-tmjcx   0/4     Completed   0          21m
```

1. Describe pod to see all the containers

```
oc describe pod <pod-name>
ex:
oc describe pod javamp-war-b-d-manual-pipeline-run-build-deploy-task-d6dz-tmjcx

                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From                                      Message
  ----    ------     ----       ----                                      -------
  Normal  Scheduled  <unknown>  default-scheduler                         Successfully assigned deployment-learning/javamp-war-b-d-manual-pipeline-run-build-deploy-task-d6dz-tmjcx to worker0.eaglets.os.fyre.ibm.com
  Normal  Pulled     29m        kubelet, worker0.eaglets.os.fyre.ibm.com  Container image "quay.io/openshift-pipeline/tektoncd-pipeline-creds-init:v0.10.1" already present on machine
  Normal  Created    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Created container credential-initializer
  Normal  Started    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Started container credential-initializer
  Normal  Pulled     28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Container image "quay.io/openshift-pipeline/tektoncd-pipeline-entrypoint:v0.10.1" already present on machine
  Normal  Created    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Created container place-tools
  Normal  Started    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Started container place-tools
  Normal  Pulled     28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Container image "quay.io/openshift-pipeline/tektoncd-pipeline-git-init:v0.10.1" already present on machine
  Normal  Created    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Created container step-git-source-git-source-bxkbg
  Normal  Started    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Started container step-git-source-git-source-bxkbg
  Normal  Pulling    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Pulling image "kabanero/ubi8-maven:latest"
  Normal  Pulled     28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Successfully pulled image "kabanero/ubi8-maven:latest"
  Normal  Created    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Created container step-build
  Normal  Started    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Started container step-build
  Normal  Pulled     28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Container image "aadeshpa/ubi8-maven-openssh-clients:2.0" already present on machine
  Normal  Created    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Created container step-copy-files-to-vm
  Normal  Started    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Started container step-copy-files-to-vm
  Normal  Pulled     28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Container image "aadeshpa/ubi8-maven-openssh-clients:2.0" already present on machine
  Normal  Created    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Created container step-deploy
  Normal  Started    28m        kubelet, worker0.eaglets.os.fyre.ibm.com  Started container step-deploy
```
2. Check logs as below


- All containers logs in your pod
```
oc logs -f <pod-name> --all-containers
```
or

- Single container log in your pod
```
oc logs --max-log-requests=20 <pod-name> -c <container-name>
```

- examples
```
oc logs --max-log-containers=20 javamp-war-b-d-manual-pipeline-run-build-deploy-task-d6dz-tmjcx --all-containers
```
or
```
oc logs -f javamp-war-b-d-manual-pipeline-run-build-deploy-task-d6dz-tmjcx -c step-build

```

3. Delete the pipelineruns

- You can list all the pipelineruns with below command
```
oc get pipelineruns -n deployment-learning

```
- To delete the pipelineruns you can use below command

```
oc delete pipelineurns --all
or
oc delete pipelineruns <pipelinerun-name>
```

