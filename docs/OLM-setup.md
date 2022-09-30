# Understanding Operator Lifecycle Manager(OLM)

This document helps us understand how OLM helps us with setting up the automated deployment of operators on operator version updates.

This extend the way to deploy the operator to cluster without the usage of operator-sdk which we discussed in previous chapter about Operator-Initialisation.

## What is OLM?

OLM is a project, a part of Operator Framework of which operator-sdk is also a part. It is a utility which extends Kubernetes and provides a way to install, manage, and upgrade operators and their dependencies in a cluster.

It removes an overhead of deploying a new operator whenever there's an update in any part of the operator.

OLM is an important component of Openshift Cluster Platform. It aids the kube admin to make the updates on Operator to be deployed to any cluster anywhere on globe automatically. Using Subscriptions and OperatorHub.io which we will discuss later.

### First let's break how usually deployment of an operator happens:-

In the last chapter we generated all the manifests in the `config` folder.

This `config` folder contains RBAC permissions, CRDs, and Controller Deployment Yamls.

If we run `make deploy`, it will deploy all these manifests one by one in order. Bringing up the controller pods as per the deployment kind yaml.

So no automation during deployment.

If there's a change in these manifests present in `config` folder after updates. One has to distribute these manifests as a form of docker image and one has to deploy it on their container accordingly maintaining versions and all other things.

To automate all these OLM comes to rescue.

## How OLM solves the issue of automating install, manage and update of operators

### Before start discussing the components of OLM, I will delve into types of packaging formats the manifests are bundled into([reference](https://docs.openshift.com/container-platform/4.7/operators/understanding/olm-packaging-format.html)):-

The packaging format difference starts from the point we trigger catalog creation, that is a new bundle folder is already created with all the CRDs, CSV, and roles and we update the catalog which already exists in the form of registry or index.

1. Package Manifests Format:-
    - The Package Manifest Format for Operators is the legacy packaging format introduced by the Operator Framework. While this format is deprecated in OpenShift Container Platform 4.5, it is still supported and Operators provided by Red Hat are currently shipped using this method.
    - It creates a catalog using the directory structure:-
        ```
        etcd
        ├── 0.6.1
        │   ├── etcdcluster.crd.yaml
        │   └── etcdoperator.clusterserviceversion.yaml
        ├── 0.9.0
        │   ├── etcdbackup.crd.yaml
        │   ├── etcdcluster.crd.yaml
        │   ├── etcdoperator.v0.9.0.clusterserviceversion.yaml
        │   └── etcdrestore.crd.yaml
        ├── 0.9.2
        │   ├── etcdbackup.crd.yaml
        │   ├── etcdcluster.crd.yaml
        │   ├── etcdoperator.v0.9.2.clusterserviceversion.yaml
        │   └── etcdrestore.crd.yaml
        └── etcd.package.yaml
        ```
        The requirements for the proper packaging are-

        - It should be nested with each bundle version(CSV) as a directory containing all the generated manifests and metadata for the corresponding version.
        - A package manifest file detailing channels and version graph such that OLM knows how to update the operator

2. Bundle Format - It is the new packaging format OLM is shifting to, the major change in the formating is that it uses indexing using DB Registry.
    - Each of the bundle version generated need to be created as a docker image first. This image will contain all the manifests and metadata for that bundle.
    - Now a version-upgrade command need to be executed on top of existing index image to create a graph of version sequence. This will generate a db index file such that each CSV pointing to a bundle image.
    - Push this index image to registry such that OLM can update it's registry for that catalog.

In our project we have used Bundle Format. But all the existing RH operators use Package Manifest Format.

### Now we cover the components of OLM:- 

- `manifests` - All the config files required for the operator deployment like RBAC permissions, ServiceAccounts, Operator Deployment, CRDs, ConfigMaps etc.
- `Bundle Image` - It is a collection of manifests & metadata for a particular operator version, determined by CSV(Cluster Service Version)
- `Cluster Service Version(CSV)` - It is a yaml generated as a part of bundle which assists OLM to run an operator successfully inside a cluster. It has information about all the requirements to run an oerator, like the RBAC rules, CRs to manage, dependent CRDs, Controller Image to execute, install strategies, name, version, logo image etc.
- `Catalog Registry`- It is a database registry, OLM uses to save and read various Catalog Sources present for the corresponding Operator installation metadata.
- `Catalog Source` - It is a repository of metadata about all the versions and channels available for the Operator registry or index. For bundle package format it is a indexed DB containing information about each bundle version's metadata and manifests. For package manifests format, it is a repository containing all the bundles with a package manifest yaml file with update graph.
- `Subscription` - It is a way to subscribe a watch on the package manifests via catalog source such that OLM keeps a watch as per subscription demand(channel, installPlanApproval) to keep looking for latest version present in Package Manifests and update the currentCSV field of the subscription.
Subscription also contains config for setting envs, PVC, config maps for the operator which need to be deployed.
- `Install Plan` - Updating of operator in cluster is done via a install strategy, the version update graph plays a role here, the operator in the cluster is updated in sequence similar to version graph present in the package manifest or the index DB. If the operator is getting installed first time, directly latest CSV is installed.
- `Operator Groups` - For all the operators in the same Namespace other than default, an operator group is needed for OLM to deploy an operator. OperatorGroup is kind/CRD owned by OLM's Catalog Operator for managing RBAC permissions in the target namespaces where operator manages its CRs.

### Architecture of OLM:-

OLM functionality is managed by two operators:-
1. Catalog Operator
2. OLM Operator

They manage above components and own CRDs, here are CRDs owned by each operator:-
1. Catalog Operator:-
    - Subscription
    - Install Plan
    - Catalog Source

2. OLM Operator:-
    - Cluster Service Version
    - Operator Group

Below are the functionalities they provide:-

1. Catalog Operator:-
    - It connects to all the catalog sources present on the cluster. It has access to cache of CRDs and CSV, indexed by name.
    - It performs actions as per the status of install plans, it can be in state of resolved or unresolved:-
        - If install plan is unresolved:-
            - a CSV is resolved as per required by subscription.
            - Owned and required CRDs are resolved which operator will use.
            - Required CRDs are matched with their CSVs
            This brings the unresolved Install Plan to resolved status.
        - If the install plan is resolved:-
            - create all the required resources discovered for it manually if user permits or automatically.
    - Check for the subscription and updates in catalog source and create the install plans based on that.

    Note:- Not found any mention who deals with the version update graph, but mostly it must be a catalog operator that creates/replaces CSV in the update version order.

2. OLM Operator:-
    The task of an OLM Operator is to bring the balance between the desired state(CSV CR) and actual operator deployment.
    - This operator compares the CSV and installed operator version, if they differ it starts the update.
    - Operator performs a check if all the required dependencies present in CSV are met, the operator controller is deployed/updated using the install strategy present in CSV, it also tells about which version to replace.

### OLM Architecture Flowchart:-

![Flowchart]()

We are assuming here in the flowchart that required Operator Namespace, Catalog Source, Subscription and operator group are in place. 

1. All the manifests created by Operator-SDK is bundled up in a version bundle with a CSV created for that bundle containing related metadata.
2. Now depending on the catalogs packaging format we generate catalog registry which OLM uses for operator installation and version updates:-
    - Bundle Format - the bundles are containerized and push to the registry. Post that a db index is updated based on existing index image with the latest bundle created added to index. And this pushed image of index acts as the registry for OLM on clusters.
    - Package Manifests Format- the bundles are managed in a directory structure, package manifest is accordingly updated and this catalog directory is pushed as a containerized image which acts as a registry for OLM on clusters.
3. Since there's a subscription in place, it asks for resolving an install plan for the package/channel/start-version mentioned in subscription yaml
4. Since we know Catalog Operator watches the unresolved install plans and take action accordingly on it by performing:-
    - Resolving CSVs for installation.
    - Resolving owned and required CRDs.
    - Matching required CRDs with their CSVs.

    This brings the install plan in the resolved state. Thus the creation of Install Plan is done.

    1. This is a step which Catalog Sources keep performing for updating the Operator DB Registry which Catalog Operator uses for above resolution.
    2. Catalog Sources updates its own registry about all the channels and version history available in index image or registry image.

5. Catalog Operator checks for the installPlanApproval in subscription and asks user for confirmation if it is manual or starts running install plan on it's own.
6. Catalog operator create all the resources it discovered while resolving the install plan. And thus the CSV and dependent required resources are created/updated.

The below steps are working separately overall as it is handled by OLM operator, but to understand the OLM in sequence

7. OLM Operator watches all the CSVs as it owns the CRD, so it gets an event from Kube's Controller-Manager about a change. Perform a check on all the required resources for starting an Operator Controller.

If satisfies the check, it installs the operator using the install-startegy present in CSV. The CSV also contains information of replacing version. So OLM knows how to handle the change in Operator version.

8. OLM Operator deployment yaml is a managed object for OLM Operator and it creates that, which is then picked by Deployment Controller for running the deployment as pods.

This way a operator is installed on a cluster using OLM.

The above discussion is more on how install operator is triggered for first time that is by creating subscription CR.

9. But for modifying Operator it can be configured in below ways:-
    1. Editing Subscription spec- Setting starting CSV and installPlanApproval as manual will bring the Operator to desired version by following above steps starting from Step 3.
    2. Catalog Source updated by polling, new version added to Catalog Registry or Index. If Subscription's installPlanApproval is Automatic the operator update will be automatic starting from a unresolved Install Plan creation by Catalog Operator.


## References

- https://docs.openshift.com/container-platform/4.7/operators/understanding/olm/olm-understanding-olm.html
- https://olm.operatorframework.io/docs/concepts/olm-architecture/
- https://github.com/operator-framework/operator-lifecycle-manager/blob/master/doc/design/philosophy.md