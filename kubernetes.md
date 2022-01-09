
* Deployment
    * An object that can represent an application running on the cluster.
    * Manages a replicated application, typically by running Pods with no local state.
    * You describe a desired state in a Deployment, and the Deployment Controller changes the actual state to the desired state at a controlled rate. 
* Pods
    * The smallest and simplest Kubernetes object.
    * A Pod represents a set of running containers on your cluster.
    * Pods are the most atomic unit of work in a Kubernetes cluster.
    * Pods are the runtime objects that live in specific nodes.
* ReplicaSet
    * A ReplicaSet's purpose is to maintain a stable set of replica Pods running at any given time.
* Service
    * An abstract way to expose an application running on a set of Pods as a network service.
