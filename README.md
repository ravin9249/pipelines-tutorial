# OpenShift Pipelines Tutorial

You will use the [Spring PetClinic](https://github.com/spring-projects/spring-petclinic) sample application during this tutorial, which is a simple Spring Boot application.

Create the Kubernetes objects for deploying the PetClinic app on OpenShift. The deployment will not complete since there are no container images built for the PetClinic application yet. That you will do in the following sections through a CI/CD pipeline:

```bash
$ oc create -f https://raw.githubusercontent.com/ravin9249/pipelines-tutorial/release-v0.7/petclinic/manifests.yaml
```

You should be able to see the deployment in the OpenShift Web Console.

## Install Tasks

Install the `openshift-client` and `s2i-java` tasks from the catalog repository using `oc` or `kubectl`, which you will need for creating a pipeline in the next section:

```bash
$ oc create -f https://raw.githubusercontent.com/ravin9249/tektoncd-catalog/release-v0.7/openshift-client/openshift-client-task.yaml
$ oc create -f https://raw.githubusercontent.com/ravin9249/pipelines-catalog/release-v0.7/s2i-java-8/s2i-java-8-task.yaml

```

You can take a look at the tasks you created using the [Tekton CLI](https://github.com/tektoncd/cli/releases):

```
$ tkn task ls

NAME               AGE
openshift-client   58 seconds ago
s2i-java-8         1 minute ago
```

## Create Pipeline

A pipeline defines a number of tasks that should be executed and how they interact with each other via their inputs and outputs.

In this tutorial, you will create a pipeline that takes the source code of the PetClinic application from GitHub and then builds and deploys it on OpenShift using [Source-to-Image (S2I)](https://docs.openshift.com/container-platform/4.1/builds/understanding-image-builds.html#build-strategy-s2i_understanding-image-builds).

<p align="center"><img src="images/pipeline-diagram.svg" width="700" alt="Pipeline Diagram" /></div>

Here is the YAML file that represents the above pipeline:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: Pipeline
metadata:
  name: petclinic-deploy-pipeline
spec:
  resources:
  - name: app-git
    type: git
  - name: app-image
    type: image
  tasks:
  - name: build
    taskRef:
      name: s2i-java-8
    params:
      - name: TLSVERIFY
        value: "false"
    resources:
      inputs:
      - name: source
        resource: app-git
      outputs:
      - name: image
        resource: app-image
  - name: deploy
    taskRef:
      name: openshift-client
    runAfter:
      - build
    params:
    - name: ARGS
      value:
        - rollout
        - latest
        - spring-petclinic
```

This pipeline performs the following:
1. Clones the source code of the application from a git repository (`app-git`
   resource)
2. Builds the container image using the `s2i-java-8` task that generates a
   Dockerfile for the application and uses [Buildah](https://buildah.io/) to
   build the image
3. The application image is pushed to an image registry (`app-image` resource)
4. The new application image is deployed on OpenShift using the `openshift-cli`

You might have noticed that there are no references to the PetClinic git
repository or the image registry it will be pushed to. That's because pipeline in Tekton
are designed to be generic and re-usable across environments and stages through
the application's lifecycle. Pipelines abstract away the specifics of the git
source repository and image to be produced as `PipelineResources`. When triggering a
pipeline, you can provide different git repositories and image registries to be
used during pipeline execution. Be patient! You will do that in a little bit in
the next section.

The execution order of task is determined by dependencies that are defined between the tasks via inputs and outputs as well as explicit orders that are defined via `runAfter`.

Create the pipeline by running the following:

```bash
$ oc create -f https://raw.githubusercontent.com/ravin9249/pipelines-tutorial/release-v0.7/pipeline/01-build-deploy.yaml
```

Alternatively, in the OpenShift web console, you can click on the **+** at the top right of the screen while you are in the **pipelines-tutorial** project:

![OpenShift Console - Import Yaml 1](images/console-import-yaml-1.png)

Paste the YAML into the text editor and click on **Create**:

![OpenShift Console - Import Yaml 2](images/console-import-yaml-2.png)

Upon creating the pipeline via the web console, you will be taken to a **Pipeline Details** page that gives an overview of the pipeline you created:

![OpenShift Console - Pipeline Details](images/pipeline-details.png)

Check the list of pipelines you have created using the CLI:

```
$ tkn pipeline ls

NAME                       AGE              LAST RUN   STARTED   DURATION   STATUS
petclinic-deploy-pipeline  25 seconds ago   ---        ---       ---        ---
```

## Trigger Pipeline

Now that the pipeline is created, you can trigger it to execute the tasks
specified in the pipeline.

First, you should create a number of `PipelineResources` that contain the specifics of the git repository and image registry to be used in the pipeline during execution. Expectedly, these are also reusable across multiple pipelines.

The following `PipelineResource` defines the git repository for the PetClinic application:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-git
spec:
  type: git
  params:
  - name: url
    value: https://github.com/spring-projects/spring-petclinic
```

And the following defines the OpenShift internal image registry for the PetClinic image to be pushed to:

```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: petclinic-image
spec:
  type: image
  params:
  - name: url
    value: image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/spring-petclinic
```

Create the above pipeline resources via the OpenShift web console or by running the following:

```bash
$ oc create -f https://raw.githubusercontent.com/ravin9249/pipelines-tutorial/release-v0.7/pipeline/02-resources.yaml
```

You can see the list of resources created using `tkn`:

```bash
$ tkn resource ls

NAME              TYPE    DETAILS
petclinic-git     git     url: https://github.com/spring-projects/spring-petclinic
petclinic-image   image   url: image-registry.openshift-image-registry.svc:5000/pipelines-tutorial/spring-petclinic
```

A `PipelineRun` is how you can start a pipeline and tie it to the git and image resources that should be used for this specific invocation. You can start the pipeline using `tkn`:

```bash
$ tkn pipeline start petclinic-deploy-pipeline \
        -r app-git=petclinic-git \
        -r app-image=petclinic-image \
        -s pipeline
```

The `-r` flag specifies the pipeline resources that should be provided to the pipeline, and the `-s` flag specifies the service account to be used for running the pipeline.

> **Note**: OpenShift Pipelines 0.7 does not automatically use the `pipeline` service account for running pipelineruns. This has been fixed in the next release (OpenShift Pipelines 0.8), but if you want to use the OpenShift web console developer perspective to start the pipeline with OpenShift Pipelines 0.7, run the following commands to elevate the permissions of the `default` service account, which is currently used by default for running pipelineruns that are started by the OpenShift Console:  
>  ```
>  $ oc adm policy add-role-to-user edit -z default
>  ```

As soon as you start the `petclinic-deploy-pipeline` pipeline, a pipelinerun will be instantiated and pods will be created to execute the tasks that are defined in the pipeline.

```bash
$ tkn pipeline list
NAME                        AGE             LAST RUN                              STARTED         DURATION   STATUS
petclinic-deploy-pipeline   23 seconds ago   petclinic-deploy-pipeline-run-tsv92  23 seconds ago   ---        Running
```

Check out the logs of the pipelinerun as it runs using the `tkn pipeline logs` command which interactively allows you to pick the pipelinerun of your interest and inspect the logs:

```
$ tkn pipeline logs -f
? Select pipeline : petclinic-deploy-pipeline
? Select pipelinerun : petclinic-deploy-pipeline-run-tsv92 started 39 seconds ago
```

After a few minutes, the pipeline should finish successfully.

```bash
$ tkn pipeline list

NAME                        AGE             LAST RUN                              STARTED         DURATION    STATUS
petclinic-deploy-pipeline   7 minutes ago   petclinic-deploy-pipeline-run-tsv92   7 minutes ago   7 minutes   Succeeded
```



If you want to re-run the pipeline again, you can use the following short-hand command to rerun the last pipelinerun again that uses the same pipeline resources and service account used in the previous pipeline run:

```
tkn pipeline start petclinic-deploy-pipeline --last
```
