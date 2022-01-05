# Organizing Your Application

Throughout this book we have described various components of an application built on top of Kubernetes. We have described how to wrap programs up as containers, place those containers in Pods, replicate those Pods with ReplicaSets, and roll out software each week with deployments. We have even described how to deploy stateful and real-world applications that put together a collection of these objects into a single distributed system. But we have not covered how to actually work with such an appli‐ cation in a practical way. How can you lay out, share, manage, and update the various configurations that make up your application? That is the topic for this chapter.

## Principles to Guide Us

Before digging into the concrete details of how to structure your application, it’s worth considering the goals that drive this structure. Obviously, reliability and agility are the general goals of developing a cloud-native application in Kubernetes, but moving to the next level of detail, how does this actually relate to how you design the maintenance and deployment of your application? The following sections describe the various principles that we can use as a guide to design a structure that best suits these goals. The principles are:

    • Filesystems as the source of truth
    • Code review to ensure the quality of changes
    • Feature flags for staged roll forward and roll back

### Filesystems as the Source of Truth

When you first begin to explore Kubernetes, as we did in the beginning of this book, you generally interact with it imperatively. You run commands like kubectl run or kubectl edit to create and modify Pods or other objects running in your cluster. Even when we started exploring how to write and use YAML or JSON files, this was presented in an ad-hoc manner, as if the file itself is just a way station on the way to modifying the state of the cluster. In reality, in a true productionized application the opposite should be true.

Rather than viewing the state of the cluster—the data in etcd—as the source of truth, it is optimal to view the filesystem of YAML objects as the source of truth for your application. The API objects deployed into your Kubernetes cluster(s) are then a reflection of the truth stored in the filesystem.

### The Role of Code Review

it is also obvious that code review of these configurations is critical to the reliable deployment of services. In our experience, most service outages are self-inflicted via unexpected consequences, typos, or other simple mistakes. Ensuring that at least two people look at any configuration change significantly decreases the probability of such errors.

### Feature Gates and Guards

Once your application source code and your deployment configuration files are in source control, one of the most common questions that occurs is how these reposito‐ ries relate to one another. Should you use the same repository for application source code as well as configuration? This can work for small projects, but in larger projects it often makes sense to separate the source code from the configuration to provide for a separation of concerns. Even if the same people are responsible for both building and deploying the application, the perspectives of the builder versus the deployer are different enough that this separation of concerns makes sense.

If that is the case, then how do you bridge the development of new features in source control with the deployment of those features into a production environment? This is where feature gates and guards play an important role.

There are a variety of benefits to this approach. First, it enables the committing of code to the production branch long before the feature is ready to ship. This enables feature development to stay much more closely aligned with the HEAD of a repository, and thus you avoid the horrendous merge conflicts of a long-lived branch.

The use of feature flags thus both simplifies debugging problems in production and ensures that disabling a feature doesn’t require a binary rollback to an older version of the code that would remove all of the bug fixes and other improvements made by the newer version of the code.

## Managing Your Application in Source Control

Now that we have determined that the filesystem should represent the source of truth for your cluster, the next important question is how to actually lay out the files in the filesystem. Obviously, filesystems contain hierarchical directories, and a source- control system adds concepts like tags and branches, so this section describes how to put these together to represent and manage your application.

### Filesystem Layout

The first cardinality on which you want to organize your application is the semantic component or layer (e.g., frontend, batch work queue, etc.). Though early on this might seem like overkill, since a single team manages all of these components, it sets the stage for team scaling—eventually, a different team (or subteam) may be responsi‐ ble for each of these components.
Thus, for an application with a frontend that uses two services, the filesystem might look like:
    
    frontend/
    service-1/
    service-2/

Within each of these directories, the configurations for each application are stored. These are the YAML files that directly represent the current state of the cluster. It’s generally useful to include both the service name and the object type within the same file.

Thus, extending our previous example, the filesystem might look like:
    
    frontend/
       frontend-deployment.yaml
       frontend-service.yaml
       frontend-ingress.yaml
    service-1/
       service-1-deployment.yaml
       service-1-service.yaml
       service-1-configmap.yaml
    ...

### Managing Periodic Versions

It is very use‐ ful to be able to look back historically and see what your application deployment pre‐ viously looked like. Similarly, it is very useful to be able to iterate a configuration forward while still being able to deploy a stable release configuration.

Consequently, it’s handy to be able to simultaneously store and maintain multiple dif‐ ferent revisions of your configuration. Given the file and version control approach, there are two different approaches that you can use. The first is to use tags, branches, and source-control features. This is convenient because it maps to the same way that people manage revisions in source control, and it leads to a more simplified directory structure. The other option is to clone the configuration within the filesystem and use directories for different revisions. This approach is convenient because it makes simultaneous viewing of the configurations very straightforward.

In reality, the approaches are more or less identical, and it is ultimately an aesthetic choice between the two. Thus, we will discuss both approaches and let you or your team decide which you prefer.

### Versioning with branches and tags

When you use branches and tags to manage configuration revisions, the directory structure is unchanged from the example in the previous section. When you are ready for a release, you place a source-control tag (e.g., git tag v1.0) in the configuration source-control system. The tag represents the configuration used for that ver‐ sion, and the HEAD of source control continues to iterate forward.

The world becomes somewhat more complicated when you need to update the release configuration, but the approach models what you would do in source control. First, you commit the change to the HEAD of the repository. Then you create a new branch named v1 at the v1.0 tag. You then cherry-pick the desired change onto the release branch (git cherry-pick <edit>), and finally, you tag this branch with the v1.1 tag to indicate a new point release.

### Versioning with directories

An alternative to using source-control features is to use filesystem features. In this approach, each versioned deployment exists within its own directory. For example, the filesystem for your application might look like this:

    frontend/
      v1/
        frontend-deployment.yaml
        frontend-service.yaml
      current/
        frontend-deployment.yaml
        frontend-service.yaml
    service-1/
      v1/
         service-1-deployment.yaml
         service-1-service.yaml
      v2/
         service-1-deployment.yaml
         service-1-service.yaml
      current/
         service-1-deployment.yaml
         service-1-service.yaml
    ...

Thus, each revision exists in a parallel directory structure within a directory associ‐ ated with the release. All deployments occur from HEAD instead of from specific revi‐ sions or tags. When adding a new configuration, it is done to the files in the current directory.
When creating a new release, the current directory is copied to create a new directory associated with the new release.

When performing a bugfix change to a release, the pull request must modify the YAML file in all the relevant release directories.

## Structuring Your Application for Development, Testing, and Deployment

In addition to structuring your application for a periodic release cadence, you also want to structure your application to enable agile development, quality testing, and safe deployment. This enables developers to rapidly make and test changes to the dis‐ tributed application, and to safely roll those changes out to customers.


### Progression of a Release

    HEAD
    The bleeding edge of the configuration; the latest changes.
    Development
    Largely stable, but not ready for deployment. Suitable for developers to use for building features.
    Staging
    The beginnings of testing, unlikely to change unless problems are found.
    Canary
    The first real release to users, used to test for problems with real-world traffic and likewise give users a chance to test what is coming next.
    Release
    The current production release.

### Introducing a development tag

To introduce a development stage, a new development tag is added to the source- control system and an automated process is used to move this tag forward. On a peri‐ odic cadence, HEAD is tested via automated integration testing. If these tests pass, the development tag is moved forward to HEAD. Thus, developers can track reasonably close to the latest changes when deploying their own environments, but they also can be assured that the deployed configurations have at least passed a limited smoke test.

### Mapping stages to revisions

It might be tempting to introduce a new set of configurations for each of these stages, but in reality, the Cartesian product of versions and stages would create a mess that is very difficult to reason about. Instead, the right practice is to introduce a mapping between revisions and stages.
Regardless of whether you are using the filesystem or source-control revisions to rep‐ resent different configuration versions, it is easy to implement a map from stage to revision. In the filesystem case you can use symbolic links to map a stage name to a revision:

    frontend/
       canary/ -> v2/
       release/ -> v1/
       v1/
         frontend-deployment.yaml
    ...

In the case of version control, it is simply an additional tag at the same revision as the appropriate version.
In either case, the versioning of releases proceeds using the processes described previ‐ ously, and separately the stages are moved forward to new versions as appropriate. Effectively this means that there are two simultaneous processes, the first for cutting new release versions and the second for qualifying a release version for a particular stage in the application lifecycle.

## Parameterizing Your Application with Templates

Once you have a Cartesian product of environments and stages, it becomes clear that it is impractical or impossible to keep them all entirely identical. And yet, it is impor‐ tant to strive for the environments to be as identical as possible. Variance and drift between different environments produces snowflakes and systems that are hard to reason about. If your staging environment is different than your release environment, can you really trust the load tests that you ran in the staging environment to qualify a release? To ensure that your environments stay as similar as possible, it is useful to use parameterized environments. Parameterized environments use templates for the bulk of their configuration, but they mix in a limited set of parameters to produce the final configuration. In this way most of the configuration is contained within a shared template, while the parameterization is limited in scope and maintained in a small parameters file for easy visualization of differences between environments.

## Parameterizing with Helm and Templates

There are a variety of different languages for creating parameterized configurations. In general they all divide the files into a template file, which contains the bulk of the configuration, and a parameters file, which can be combined with the template to produce a complete configuration. In addition to parameters, most templating lan‐ guages allow parameters to have default values if no value is specified.
The following gives examples of how to parameterize configurations using Helm, a package manager for Kubernetes. Despite what devotees of various languages may say, all parameterization languages are largely equivalent, and as with programming langauges, which one you prefer is largely a matter of personal or team style. Thus, the same patterns described here for Helm apply regardless of the templating lan‐ guage you choose.

The Helm template language uses the “mustache” syntax, so for example:

    metadata:
      name: {{ .Release.Name }}-deployment

indicates that Release.Name should be substituted into the name of a deployment. To pass a parameter for this value you use a values.yaml file with contents like:

    Release:
      Name: my-release

Which after parameter substitution results in:

    metadata:
      name: my-release-deployment

## Filesystem Layout for Parameterization

Now that you understand how to parameterize your configurations, how do you apply that to the filesystem layouts we have described previously? To achieve this, instead of treating each deployment lifecycle stage as a pointer to a version, each deployment lifecycle is the combination of a parameters file and a pointer to a spe‐ cific version. For example, in a directory-based layout this might look like:

    frontend/
      staging/
        templates -> ../v2
        staging-parameters.yaml
      production/
        templates -> ../v1
        production-parameters.yaml
      v1/
        frontend-deployment.yaml
        frontend-service.yaml
      v2/
        frontend-deployment.yaml
        frontend-service.yaml
    ...

Doing this with version control looks similar, except that the parameters for each life‐ cycle stage are kept at the root of the configuration directory tree:
    
    frontend/
      staging-parameters.yaml
      templates/
        frontend-deployment.YAML
    ....


## Deploying Your Application Around the World

the final step in structuring your configurations is to deploy your application around the world. But don’t think that these approaches are only for large-scale applications. In reality, they can be used to scale from two different regions to tens or hundreds around the world. In the world of the cloud, where an entire region can fail, deploying to multiple regions (and managing that deployment) is the only way to achieve sufficient uptime for demanding users.

### Architectures for Worldwide Deployment

Generally speaking, each Kubernetes cluster is intended to live in a single region, and each Kubernetes cluster is expected to contain a single, complete deployment of your application. Consequently, a worldwide deployment of an application consists of mul‐ tiple different Kubernetes clusters, each with its own application configuration.

Describing how to actually build a worldwide application, especially with complex subjects like data replication, is beyond the scope of this chapter, but we will describe how to arrange the application configurations in the filesystem.
Ultimately, a particular region’s configuration is conceptually the same as a stage in the deployment lifecycle. Thus, adding multiple regions to your configuration is iden‐ tical to adding new lifecycle stages. For example, instead of:

    • Development • Staging
    • Canary
    • Production

You might have:

    • Development • Staging
    • Canary
    • EastUS
    • WestUS • Europe • Asia

Modeling this in the filesystem for configuration, this looks like:
    
    frontend/
      staging/
        templates -> ../v3/
        parameters.yaml
      eastus/
        templates -> ../v1/
        parameters.yaml
      westus/
        templates -> ../v2/
        parameters.yaml
      ...

If you instead are using version control and tags, the filesystem would look like:
    
    frontend/
      staging-parameters.yaml
      eastus-parameters.yaml
      westus-parameters.yaml
      templates/
        frontend-deployment.yaml
    ...    

### Implementing Worldwide Deployment

Now that you have configurations for each region around the world, the question becomes one of how to update those various regions. One of the primary goals of using multiple regions is to ensure very high reliability and uptime. While it would be tempting to assume that cloud and data center outages are the primary causes of downtime, the truth is that outages are generally caused by new versions of software rolling out. Because of this, the key to a highly available system is limiting the effect or “blast radius” of any change that you might make. Thus, as you roll out a version across a variety of regions, it makes sense to move carefully from region to region in order to validate and gain confidence in one region before moving on to the next.

Rolling out software across the world generally looks more like a workflow than a single declarative update: you begin by updating the version in staging to the latest version and then proceed through all regions until it is rolled out everywhere. But how should you structure the various regions, and how long should you wait to vali‐ date between regions?

To determine the length of time between rollouts to regions, you want to consider the “mean time to smoke” for your software. This is the time it takes on average after a new release is rolled out to a region for a problem (if it exists) to be discovered. Obvi‐ ously, each problem is unique and can take a varying amount of time to make itself known, and that is why you want to understand the average time. Managing software at scale is a business of probability, not certainty, so you want to wait for a time that makes the probability of an error low enough that you are comfortable moving on to the next region. Something like two to three times the mean time to smoke is proba‐ bly a reasonable place to start, but it is highly variable depending on your application.

To determine the order of regions, it is important to consider the characteristics of various regions. When you have successfully rolled out to both a low- and a high-traffic region, you may have confidence that your application can safely roll out everywhere. However, if there are regional variations, you may want to also test more slowly across a variety of geographies before pushing your release more broadly.

When you put your release schedule together, it is important to follow it completely for every release, no matter how big or how small. Many outages have been caused by people accelerating releases either to fix some other problem, or because they believed it to be “safe.”

### Dashboards and Monitoring for Worldwide Deployments

 it is essential to develop dashboards that can tell you at a glance what version is running in which region, as well as alerting that will fire when too many different ver‐ sions of your application are deployed. A best practice is to limit the number of active versions to no more than three: one testing, one rolling out, and one being replaced by the rollout. Any more active versions than this is asking for trouble.