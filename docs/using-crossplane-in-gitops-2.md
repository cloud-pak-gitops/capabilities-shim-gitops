<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Using Crossplane in GitOps](#using-crossplane-in-gitops)
  - [Store Manifests in Git Repository](#store-manifests-in-git-repository)
    - [Managed Resources](#managed-resources)
      - [Using Kustomize](#using-kustomize)
    - [Managed Resources As Template](#managed-resources-as-template)
    - [Composition and CompositeResourceDefinition](#composition-and-compositeresourcedefinition)
  - [Crossplane vs. Helm](#crossplane-vs-helm)
    - [Crossplane Composition vs. Helm Templates](#crossplane-composition-vs-helm-templates)
    - [XRD, XRC, XR vs. values.yaml](#xrd-xrc-xr-vs-valuesyaml)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

# Using Crossplane in GitOps, Part II 

In this article, I will share my recent study on Crossplane use in GitOps. I will use Argo CD as the GitOps tool to demonstrate how Crossplane can work with it to provision applications from git to target cluster. During the time, I will also explore some common considerations and practices that you might think of as well when use Crossplane in GitOps.

## Store Manifests in Git Repository

Now that you have launched the environment with all the prerequisites ready including Crossplane, you can start to push application manifests in git to trigger the application provisioning driven by GitOps. There are several ways that we can consider.

### Managed Resources

In Crossplane, managed resource is Kubernetes custom resource defined and handled by provider. You can think of Crossplane with its provider equivalent to Kubernetes controller or operator. As such, when you push managed resource as manifest in git, it will be synchronized by Argo CD from git repository to target cluster, then detected by the provider and drive the actual application provisioning.

![](images/managed-resource.png)

This approach is very straightforward, but may not scale very much because it does not allow you to customize the configuration.

For example, if you want to deploy the application to another namespace instead of the default one. You need a way to override the default configuration.

Another example is the per-environment deployment. When you deploy application to multiple clusters where some clusters may have specific configuration than others, you may need per-environment configuration.

Of course, if you have one folder for each environment in git, you can copy and paste all the manifests to each folder that maps to the specific environment and do the environment specific modifications there. This will lead to many duplications which is hard to maintain when the repository grows.

#### Using Kustomize

Per environment configuration can be done by [Kustomize](https://kustomize.io/). By using Kustomize, you can have the manifests with their default configuration at `base` layer, then specify the custom settings at `overlays` layer to override the base one.

However, it should not be overused too much. The reason is that it obfuscates the understanding of what is actually deployed. If there are many kustomize-based versions of the same application manifests for different clusters, you have to assemble the final YAML in your head to understand what is actually deployed in each cluster. In such a case, a templated framework like Helm would help.

### Managed Resources As Template

Helm dynamically generates the configuration based on functions and parameters. It results in more reusable manifests stored in git. By using Helm, you can extract customizable configuration out of the managed resources, and put them into `values.yaml` with default values provided. With that, the managed resources stored in git will be templated manifests.

![](images/managed-resource-template.png)

The good thing is that Argo CD supports Helm quite well. You can override the configuration defined in `values.yaml` when you define Argo `Application` resource for your application to be deployed. As an example, in the below Argo `Application`, we customized the name of the capabilities, and the namespace to be deployed via `spec.source.helm.parameters`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: capabilities-logging-app
  namespace: argocd
spec:
  destination:
    namespace: dev
    server: 'https://kubernetes.default.svc'
  source:
    path: config/capabilities/crossplane-helm/logging
    repoURL: 'https://github.com/morningspace/capabilities-shim-gitops'
    targetRevision: HEAD
    helm:
      parameters:
        - name: metadata.namespace
          value: dev
        - name: metadata.name
          value: dev-env-logging-stack
  project: default
```

You can even override the configuration via Argo CD UI when you create or update the Argo `Application`.

![](images/argo-ui-helm.png)

### Composition and CompositeResourceDefinition

Crossplane has a powerful composition engine. By defining `Composition` resource, it can compose multiple resources from providers driven by different vendors at backend at different level from infrastructure to application.

It also supports `CompositeResourceDefinition` (XRD) resource, which is extracted from the resources to be composed, and exposed as configurable settings with well-defined type and schema.

Both `Composition` and `CompositeResourceDefinition` resources can be checked into git, so they can be synchronized by GitOps tool from git to target cluster. Based on that, you can define `CompositeResource` (XR) or `CompositeResourceClaim` (XRC), which is usually environment specific, and check it in git as well. After it is synchronized from git to target cluster, this will trigger Crossplane to generate the corresponding managed resources which will be detected by the providers and drive the actual application provisioning.

![](images/composition-and-xrd.png)

In our case, since we have already synchronized and installed the Crossplane configuration package which includes all `Composition` and `CompositeResourceDefinition` resources to the target cluster by defining and checking the `configurations.pkg.crossplane.io` resource in git, you do not have to check these packaged resources in git any more. The only thing you need to check in git is the `CompositeResourceClaim` resource such as below:

```yaml
apiVersion: capabilities.morningspace.io/v1alpha1
kind: LoggingClaim
metadata:
  annotations:
    capabilities.morningspace.io/provider-config: kubernetes-provider
  name: my-logging-stack
spec:
  parameters:
    esVersion: 7.13.3
    kibanaVersion: 7.13.3
  compositionSelector:
    matchLabels:
      capability: logging
      provider: olm
```

The `CompositeResourceClaim` resource is usually environment specific. That means you can put it into environment specific folder in git. However, if you want the resource to be reusable and only override it partially per environment, you can also use Kustomize. Below is a sample folder structure:

```
└── environments
    ├── base
    │   ├── logging-claim.yaml
    │   └── kustomization.yaml
    └── overlays
        └── dev
            ├── logging-claim.yaml
            └── kustomization.yaml
```

There is a `logging-claim.yaml` in `base` folder, and a customized version in `overlays/dev` folder to override the base for environment dev.

```yaml
apiVersion: capabilities.morningspace.io/v1alpha1
kind: LoggingClaim
metadata:
  name: my-logging-stack
spec:
  parameters:
    esVersion: 7.15.0
    kibanaVersion: 7.15.0
```

Here we changed the version of Elasticsearch and Kibana to 7.15.0 as opposed to the default value in base, 7.13.3.

## Crossplane vs. Helm

When use Composition, you may notice that it is very similar to Helm templates since essentially they both compose a set of Kubernetes resources. From application deployment point of view, Crossplane, as a deployment tool, provides some building blocks that are very similar to what Helm does, but they also have differences. In this section, I will explore these building blocks and make one-to-one comparison between Crossplane and Helm.

Before that, there is one thing you may need to know: Crossplane and Helm are not mutual exclusive. Instead, they can be combined together. For example, you have already seen that Crossplane managed resources can be made as template using Helm. Especially, when you only use Crossplane providers and do not use its comopsition feature, the Crossplane runtime with the provider is very similar to a Kubernetes controller or an operator. In such a case, using Helm to render Kubernetes resources managed by the controller or operator is a very common practice.

### Crossplane Composition vs. Helm Templates

A Crossplane `Composition` resource defines way of composing a set of Kubernetes resources. It is equivalent to Helm templates which include a set of template files and each file maps to one or more Kubernetes resources. The difference is that Composition organizes resources in a monolithic way where all resources are defined in the same file. But for Helm templates, they are separate files under the same folder.

Instead of templating, Crossplane renders the `Composition` resource by extracting values from `CompositeResource` (XR) or `CompositeResourceClaim` (XRC) resource and patching it to specific fields on managed resources. This is very similar to Kustomize.

Also, a Crossplane Configuration package typically includes a set of Compositions, which map to multiple Helm charts or sub-charts.

At a much higher level, we usually see Crossplane Composition is used to composing modules from infrastructure, service, to application in more coarse grained way. On the other hand, Helm usually focuses on "composing" modules at application level in more fine grained way.

![](images/crossplane-vs-helm.png)

### XRD, XRC, XR vs. values.yaml

The Crossplane `CompositeResource` (XR) or `CompositeResourceClaim` (XRC) resource is essentially equivalent to the `values.yaml` file in Helm. Just like `values.yaml`, XR/XRC is aimed to extract the configurable settings out of the original resources for people to consume.

As an example, in our demo project, there is also a folder including all manifests used to provision the demo application using Helm. If you look at the `values.yaml` inside the folder, you will see it is very similar to the `CompositeResourceClaim` resource that we defined as above:

```yaml
metadata:
  name: my-logging-stack
  namespace: default
spec:
  parameters:
    esVersion: 7.13.3
    kibanaVersion: 7.13.3
```

The major difference between the two representations is that, Crossplane uses a more well-defined data structure to organize these configurable settings. That is `CompositeResourceDefinition` (XRD). By defining XRD, you can control user input with these settings in a more controlled manner. For example, each field has a type and can be required or optional. All user input verification happens at server side. This is very different than Helm. Also, each field can have a description so that can be well documented.

Another difference is that, GitOps tool such as Argo CD has integrated with Helm very well.
You can specify custom settings in Argo `Application` resource even from UI without touching `values.yaml` directly. Crossplane on the other side has no such level of integration at the moment.

Here is a table that summarizes all above differences that we explored.

| Crossplane                                | Helm        | Description
|:------------------------------------------|:------------|:-----
| Composition                               | Templates   | Both to compose a set of Kubernetes resources, but Composition uses patch to override while Helm uses template.
| CompositeResource, CompositeResourceClaim | values.yaml | Both to allow user input as configurable settings. Argo CD has better support on Helm, e.g: to specify values in Argo `Application` resource or from UI.
| CompositeResourceDefinition               | n/a         | CompositeResourceDefinition as a schema has better user input control.

*(To be continued)*