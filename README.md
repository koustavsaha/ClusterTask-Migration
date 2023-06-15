# Migration from ```ClusterTasks``` to Tekton ```Resolvers``` in Openshift Pipelines 

Publish date: ASAP, not tied to a launch date
#### Author: 
Koustav Saha,  
Vincent Demeester 

## What are ```ClusterTasks```: 
```ClusterTasks``` are CRDs that Openshift Pipeline provides that are the cluster-scoped equivalent of a Task. They share all of the same fields as a ```Task``` but can be referenced regardless of the namespace that a ```TaskRun``` is executing in. By contrast a ```Task``` can only be referred to by a ```TaskRun``` in the same namespace. 
Openshift Pipelines ships with various default cluster tasks like ```git clone```, ```buildah```, etc. Additionally users can create their own ClusterTasks by creating a ```Task``` object with the ```kind``` field set to ```ClusterTask```. 
See the below example of a ClusterTask and how it is getting used in pipelines. We will use this same task to show how you can migrate from ClusterTask using various resolvers. 

```YAML
apiVersion: tekton.dev/v1beta1
kind: ClusterTask
metadata:
  name: generate-random-number
spec:
  description: Generates a random number.
  inputs:
    parameters:
    - name: min
      type: string
    - name: max
      type: string
  steps:
  - name: generate-random-number
    image: busybox
    command: ["/bin/echo", "$((RANDOM % $(params.max - params.min) + params.min))"]
  outputs:
    result:
      type: string
```

You install the ```ClusterTask``` in your openshift cluster 

```bash
oc create -f path/to/my/generate-random-number.yaml
```

Once you have installed a cluster task, you can use it in a pipeline in any namespace by specifying the taskRef field in the tasks section of the pipeline definition. 

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: generate-random-number-pipeline
spec:
  tasks:
  - name: generate-random-number
    taskRef:
      name: generate-random-number
      kind: ClusterTask
  results:
  - name: random-number
    value: $(tasks.generate-random-number.outputs.result)
```

## Why is ```ClusterTask``` getting deprecated ? 

There are a few reasons why cluster tasks are not widely used. 
- First, cluster tasks can be difficult to manage. They are not namespaced, so they can be overwritten by other cluster tasks. Additionally there is no built in versioning, hence if someone updates a ClusterTask in a non-backward compatible way, it will break any usage of it that didn't take this change into account.
- Second, they are not necessary for most use cases. Tekton Pipelines are run in a single namespace, so there is no need for a cluster-scoped Task. 
- Third, cluster tasks are not as secure as namespaced tasks. They can be accessed by any user in the cluster, even if they do not have access to the namespace where the TaskRun is running.

## Ways to migrate from ```ClusterTask```

The ```resolver``` feature in Tekton serves as an alternative to ```ClusterTask``` in the context of Openshift Pipelines. It has been introduced as a Generally Available (GA) feature starting from Openshift Pipelines 1.11. The Tekton Resolver is designed as a robust and extensible tool that enables users to resolve resources from remote locations, encompassing Git repositories, cloud buckets, and other storage systems. Implementation-wise, the Resolver is realized as a Kubernetes Custom Resource Definition (CRD), which ensures its compatibility and deployability across diverse Kubernetes clusters.

The Tekton Resolver offers a range of built-in resolvers, which are ```Git Resolver```, ```Bundle Resolver```, ```Cluster Resolver```, and ```Hub Resolver```. Each resolver comes with its own distinct capabilities and limitations. In the subsequent section, we will delve into the details of these resolvers and explore how they can effectively replace ClusterTasks. To illustrate these concepts, we will continue utilizing the aforementioned example mentioned earlier.

### [Cluster Resolver](https://tekton.dev/docs/pipelines/cluster-resolver/): 

A cluster resolver is a Tekton Pipelines feature that allows you to reference tasks and pipelines from other namespaces. This can be a replacement for ClusterTasks, you can add the tasks that you want to be available across the cluster in one or multiple namespace. These namespaces will allow an admin to create and maintain tasks that you were previously using as ClusterTasks. Thus by consolidating the desired tasks in designated namespaces, which assumes an administrative role, organizations can ensure enhanced security measures. Specifically, only the administrator of this namespace possesses the necessary privileges to modify or delete the tasks that are made available throughout the cluster. Additionally, you can block some namespaces to resolve tasks/pipelines from. 

Following the previous example mentioned in the ClusterTasks section, you can create the task in let’s say example namespace as a regular task object. 

```YAML
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: generate-random-number
spec:
  description: Generates a random number.
  inputs:
    parameters:
    - name: min
      type: string
    - name: max
      type: string
  steps:
  - name: generate-random-number
    image: busybox
    command: ["/bin/echo", "$((RANDOM % $(params.max - params.min) + params.min))"]
  outputs:
    result:
      type: string
```

You install the task manifest in openshift cluster in example namespace 

```bash
oc create -f path/to/my/generate-random-number.yaml -n example 
```

The you can use refer this task in a pipeline in any namespace like below - 

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: generate-random-number-pipeline
spec:
  tasks:
  - name: generate-random-number
    taskRef:
      resolver: cluster
      params:
        - name: kind
          value: task
        - name: name
          value: generate-random-number
        - name: namespace
          value: example 
  results:
  - name: random-number
    value: $(tasks.generate-random-number.outputs.result)
```

Additionally, you can also use the cluster resolver to reference pipelines. To do this, you need to set the ```kind``` parameter to ```pipeline```. 
Thus, the cluster resolver is a powerful feature that allows you to share tasks and pipelines across namespaces. This can make it easier to manage your pipelines and tasks, and it can also help you to improve the security and performance of your workloads. However, the shortcoming of this approach currently is that a user might not have the right to list the task and pipeline in the namespace that the resolvers looks into, making "discoverability" hard.

### [Git Resolver](https://tekton.dev/docs/pipelines/git-resolver/): 

A Git resolver is a Tekton Pipelines feature that allows you to reference tasks and pipelines from Git repositories. This can be useful for storing your tasks and pipelines in a central location, such as GitHub, and then referencing them from your pipelines. The git resolver has two modes: cloning a repository anonymously (public repositories only), or fetching individual files via an SCM provider’s API using an API token. The private repositories and API token can be set inside git-resolver-config from openshift pipelines operator. 

This can again be a replacement for ClusterTasks where the tasks can be maintained inside a central repository and reference the tasks by specifying the git repository URL, revision, path, org, etc. This allows you with a benefit of proper versioning and maintenance of the tasks. 

Continuing with the previous example, let us consider a scenario where the task YAML file is hosted within a designated location, such as the URL ```"https://github.com/tektoncd/catalog,"``` specifically under the path ```"task/generate-random-number/."``` Moreover, the task is maintained with different versions, for instance, the version ```"0.6/generate-random-number.yaml"``` Consequently, it becomes possible to reference a specific version of the task within the pipeline, thus ensuring precise task execution based on the desired version.

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: generate-random-number-pipeline
spec:
  tasks:
  - name: generate-random-number
    taskRef:
      resolver: git
      params:
        - name: url
          value: https://github.com/tektoncd/catalog.git
        - name: revision
          value: main
        - name: pathInRepo
          value: task/generate-random-number/0.6/generate-random-number.yaml
  results:
  - name: random-number
    value: $(tasks.generate-random-number.outputs.result)
```

In addition, the Git resolver can be utilized to reference pipelines, further expanding its functionality. By employing the Git resolver, users gain the flexibility to organize their desired tasks and pipelines in a manner that ensures availability throughout the cluster, while simultaneously minimizing replication efforts. Additionally, leveraging the security features offered by Git, such as Role-Based Access Control (RBAC), enhances the overall security posture of the pipelines.
At present, the Git resolver is compatible with the following platforms:
- github.com and GitHub Enterprise
- gitlab.com and self-hosted GitLab instances
- Gitea
- BitBucket Server
- BitBucket Cloud

### [Bundles Resolver](https://tekton.dev/docs/pipelines/bundle-resolver/): 

A bundles resolver is a feature in Tekton Pipelines that allows you to reference Tekton resources from a Tekton bundle image. A Tekton bundle is a collection of Tekton resources that are packaged together and can be deployed as a single unit. Bundles can be used to share Tekton resources with others, or to make it easier to deploy Tekton resources in a consistent way.
Moreover, Tekton bundles are built on top of the OCI image format. This means that bundles can be stored and distributed in any OCI registry, such as Docker Hub or Quay.io.

This can again be a replacement for ClusterTasks where you first create and push a tekton bundle containing the task in a registry and then refer the task in your pipeline using bundles resolver by passing image and other parameters in ```taskref```. As tekton bundles are built on top of OCI registry, you get the security benefits of OCI like image signing and content trust to make sure that the images are authentic and have not been tampered with, additionally you also reap the scalability and reliability benefits of OCI registries with Tekton Bundle. 

Continuing with the example, , 
First you need to create and push the bundle to a registry e.g. docker with proper tagging which will help in maintaining the task. 
```bash
​​tkn bundle push docker.io/myorg/mybundle:1.0 -f path/to/my/generate-random-number.yaml
```

Then you can use this image in a pipeline with bundles resolver. 

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: generate-random-number-pipeline
spec:
  tasks:
  - name: generate-random-number
    taskRef:
      resolver: bundles 
      params:
        - name: bundle
          value: docker.io/myorg/mybundle:1.0
        - name: name
          value: generate-random-number
        - name: kind
          value: task
  results:
  - name: random-number
    value: $(tasks.generate-random-number.outputs.result)
```
Currently there is a limitation of no more than 10 individual layers (Pipelines and/or Tasks) can be placed in a single image. However, we do recommend to use a single image per task/pipeline to reap the benefits of versioning, tagging and security. 


### [Hub Resolver](https://tekton.dev/docs/pipelines/hub-resolver/): 

A hub resolver in Tekton is a component that is used to resolve Tekton resources from the Tekton Hub or Artifact Hub. The Tekton Hub/Artifact Hub is a collection of Tekton resources, such as tasks and pipelines. The hub resolver can be used to pull in resources from the Tekton Hub and use them in your own Tekton pipelines.

By default, the hub resolver supports the public endpoint of Tekton Hub/Artifact Hub. However, you can configure the resolver to use your own hub instance by specifying the ```ARTIFACT_HUB_API``` or ```TEKTON_HUB_API``` environment variables.

You can use the hub resolver to replace cluster tasks. To do this, you need to host your tasks in your own Tekton Hub/Artifact Hub instance and then refer to those objects using the hub resolver. Hosting your tasks in a hub instance will make it easier for you to maintain and search for them using the hub UI, making them easier for other teams to consume.

Following the previous example: 
- Create a git repository and add your tasks to it.
- Deploy your own hub instance using the instructions [here](https://github.com/tektoncd/hub/blob/main/docs/DEPLOYMENT.md).
- Set the ```ARTIFACT_HUB_API``` or ```TEKTON_HUB_API environment``` variable to the URL of your hub instance.
- Use the hub resolver to refer to your tasks in your Tekton pipelines.

```YAML
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: generate-random-number-pipeline
spec:
  tasks:
  - name: generate-random-number
    taskRef:
     resolver: hub
     params:
      - name: kind
        value: task
      - name: name
        value: generate-random-number
  results:
  - name: random-number
    value: $(tasks.generate-random-number.outputs.result)
```

### Custom Resolvers

If none of the built-in resolvers “fit”, you can create your own, in any language. Tekton provides an easy way to create one in the form of a go framework/library.

The approach is similar to the way the Tekton controller is written, but in a more simple fashion. In a gist, a resolver watches the ResolutionRequest object, and picks up the one that belong to itself (through the name of the resolver, like “hub”, “git”, …). From there it will get a set of parameters and the contract is that it should return either a valid Task or Pipeline, in a relatively short period of time. This can be used to generate dynamic Task or Pipeline based of some template and parameters.

This is documented in more detail in upstream docs : [How to write a Resolver](https://tekton.dev/docs/pipelines/how-to-write-a-resolver/).


## Conclusion: 
All resolvers offer viable alternatives to ClusterTasks, with unique capabilities and limitations.

- Cluster resolver follows a K8s native approach that provides additional benefits compared to ClusterTasks, such as k8s RBAC and storage. However, it does not address task maintenance, version management, or task trust.
- Git resolver is valuable for task organization, versioning, and trust management. It reduces replication efforts and enhances security through git’s own RBAC.
- Bundle resolver offers best security benefits through OCI, such as image signing and content trust. It provides scalability and reliability advantages offered by OCI registries. 
- Hub resolver additionally reaps the benefits of easy searching and filtering of tasks using hub UI.
- Then you also have an option to create your own custom resolver.

The choice of resolver depends on the specific use case and deployment model of cluster tasks. Factors such as task organization, version management, trust, collaboration, and maintenance will help determine the most suitable resolver for your needs.



