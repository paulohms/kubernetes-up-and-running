# Deployments

The Deployment object exists to manage the release of new versions. Deployments represent deployed applications in a way that transcends any particular version.
Additionally, deployments enable you to easily move from one version of your code to the next. This “rollout” process is specifiable and careful. It waits for a user configurable amount of time between upgrading individual Pods. It also uses health checks to ensure that the new version of the application is operating correctly, and
stops the deployment if too many failures occur.

The actual mechanics of the software rollout performed by a deployment is controlled by a deployment controller that runs in the Kubernetes cluster itself. This makes it easy to integrate deployments with numerous continuous delivery tools and services. Further, running server-side makes it safe to perform a rollout from places with poor or intermittent internet connectivity. Imagine rolling out a new version of your software from your phone while riding on the subway. Deployments make this possible and safe!
