# Basic Lab

## Prerequsites

1. Operating Environment
    1. Red Hat OpenShift v4.10 or v4.12 (recommended).
    1. Installed with storages classes for File (RWX) and Block (RWO).

1. Install Tools 
    1. [git cli](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
    1. [oc cli](https://docs.openshift.com/container-platform/4.12/cli_reference/openshift_cli/getting-started-cli.html)
    1. [tkn cli](https://tekton.dev/docs/cli/)
    1. [jq](https://jqlang.github.io/jq/download/)

## Required base capabilities/compeonents

The Basic Lab is based on Cloud Pak for Integration 2023.2.1. The versions of each components used are:

1. Operators
    1. IBM Integration Platform Navigator v7.1
    1. IBM App Connect v9.1

1. Instances
    1. Plaform Navigator instance v2023.2.1
    1. App Connect Dashboard instance v12.0.9.0-r1
    1. App Connect Integration Server instance v12.0.9.0-r1 (deployed via Tekton pipeline)

## Before you start

1. Setup your Git repository
    1. Fork this repository to your Git organisation. Use your browser and go to this [link](https://github.com/khongks/app-connect-tekton-pipeline/fork).
    1. Clone the repository using the following command.
        ```sh
        git clone https://github.com/<org>/app-connect-tekton-pipeline
        ```
    1. Go to the folder `app-connect-tekton-pipeline`. (important)

1. Get your Git Personal Access Token (PAT)
    1. Login to your Git.
    1. Click on the Account icon (top-right) and from the drop down, click on `Settings`.
    1. Then, click on `Developer settings` from the navigator pane (scroll to the bottom).
    1. Click on `Personal access tokens > Fine-grained` tokens.
    1. Click on the `Generate new token` button.
    1. Give an meaningful `Name` for your token, adjust the `Expiration` to 90 days, ensure to select `Public Respositories` and click `Generate token` button (at the bottom).
    1. Copy/save the value of the token.
    1. Generate the `github-credentials.yaml` file
        ```sh
        export GIT_USER=<YOUR_GIT_USER_NAME>
        export GIT_TOKEN=<YOUR_GIT_PERSONAL_ACCESS_TOKEN>
        envsubst < github-credentials.yaml.tmpl > github-credentials.yaml
        ```
    1. You should have generated a `github-credentials.yaml` with the content here
        ```yaml
        apiVersion: v1
        kind: Secret
        metadata:
          name: github-credentials
          annotations:
            tekton.dev/git-0: https://github.com
        type: kubernetes.io/basic-auth
        stringData:
          username: <YOUR_GIT_USER_NAME>
          password: <YOUR_GIT_PERSONAL_ACCESS_TOKEN>
        ```

1. Setup IBM entitlement key
    1. Obtain IBM entitlement key
        1. Go to the [Container software library](https://myibm.ibm.com/products-services/containerlibrary).
        1. Copy the key - this will be used to create a pull secret later.
    1. From the Terminal, export the IBM entitlement key.
        ```sh
        export IBM_ENTITLEMENT_KEY=<ibm-entitlment-key-you-copied>
        ```

1. Setup storage classes
    1. Check what storageclasses are available. If there is none, you cannot proceed.
        ```sh
        oc get sc
        ```
        Results:
        ```sh
        NAME                          PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
        localblock                    kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  109m
        ocs-storagecluster-ceph-rbd   openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   103m
        ocs-storagecluster-ceph-rgw   openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  106m
        ocs-storagecluster-cephfs     openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   103m
        openshift-storage.noobaa.io   openshift-storage.noobaa.io/obc         Delete          Immediate              false                  101m
        ```
    1. From the terminal, export the storage classes.
        ```sh        
        export FILE_STORAGECLASS=ocs-storagecluster-cephfs
        export BLOCK_STORAGECLASS=ocs-storagecluster-ceph-rbd
        ```
## Install Cloud Pak for Integration operators

1. Go to the folder `demo-pre-reqs`.
1. Add IBM software to Operator Hub.
    ```sh
    oc apply -f ibm-catalog-source.yaml
    ```
    Results:
    ```sh
    catalogsource.operators.coreos.com/ibm-operator-catalog created
    ```
1. Install operators needed for the demo.
    ```sh
    oc apply -f operators
    ```
    Results:
    ```sh
    subscription.operators.coreos.com/ibm-appconnect created
    subscription.operators.coreos.com/ibm-eventstreams created
    subscription.operators.coreos.com/ibm-integration-platform-navigator created
    subscription.operators.coreos.com/postgresql created
    subscription.operators.coreos.com/openshift-pipelines-operator-rh created
    ```

## Deploy Platform UI

1. Go to the folder `demo-pre-reqs`, if you have not done so.
1. Create `integration` project.
    ```sh
    oc new-project integration
    ```
    Results:
    ```sh
    Now using project "integration" on server "https://<clusterID>.<domainName>:6443".
    ```
1. Setup entitlement key.
    ```sh
    oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=${IBM_ENTITLEMENT_KEY} --docker-server=cp.icr.io -n integration
    ```
    Results:
    ```sh
    secret/ibm-entitlement-key created
    ```
1. Setup environment variables.
    ```sh
    export FILE_STORAGECLASS=ocs-storagecluster-cephfs
    export CP4I_LICENSE=L-YBXJ-ADJNSM
    export CP4I_VERSION=2023.2.1
    envsubst < cp4i/platform-navigator.yaml.tmpl > cp4i/platform-navigator.yaml
    ```
1. Install Platform UI.
    ```sh
    oc apply -f ./cp4i
    ```
    Results:
    ```sh
    platformnavigator.integration.ibm.com/navigator created
    ```
1. Wait for Platform UI to be ready. This takes 30-45 mins because it also installs Cloud Pak for Integration (foundational services).
    ```sh
    oc get platformnavigator navigator -n integration
    ```
    Results:
    ```sh
    NAME        REPLICAS   VERSION   STATUS    READY   LASTUPDATE   AGE     MESSAGE
    navigator   1                    Pending           2m16s        4m15s   Waiting for Zen to install. This can take up to 30 minutes, please wait. The ZenService object in the [integration] namespace is being fulfilled. The ZenService status is [Running reconciliation].
    ```
1. When installation is completed, you should see the following.
    ```sh
    oc get platformnavigator navigator -n integration
    ```
    Results:
    ```sh
    NAME        REPLICAS   VERSION      STATUS   READY   LASTUPDATE   AGE   MESSAGE
    navigator   1          2023.2.1-1   Ready    True    51m          85m   Platform UI has been provisioned.
    ```
1. Access Platform UI to verify installation
    1. Get URL.
        ```sh
        oc get route -n integration navigator-pn -ojson | jq -r .spec.host
        ```
        Results:
        ```sh
        navigator-pn-integration.apps.<clusterID>.<domainName>
        ```
    1. Get credentials.
        ```sh
        oc get secret -n ibm-common-services platform-auth-idp-credentials -ojson | jq -r .data.admin_password | base64 -d
        ```
    1. Go to private window of a browser (incognito mode) and login to Platform UI.
        ![Platform UI](../demo-pre-reqs/images/platform-ui.png)

## Deploy ACE Dashboard UI

1. Go to the folder `demo-pre-reqs`, if you have not done so.
1. Create `ace-demo` project.
    ```sh
    oc new-project ace-demo
    ```
    Results:
    ```sh
    Now using project "ace-demo" on server "https://api.<clusterID>.<domainName>:6443".
    ```
1. Setup entitlement key.
    ```sh
    oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=${IBM_ENTITLEMENT_KEY} --docker-server=cp.icr.io -n ace-demo
    ```
    Results:
    ```sh
    secret/ibm-entitlement-key created
    ```
1. Setup environment variables.
    ```sh
    export FILE_STORAGECLASS=ocs-storagecluster-cephfs
    export ACE_LICENSE=L-LFMR-BTD75V
    export ACE_VERSION=12.0.9.0-r1
    envsubst < appconnect/dashboard.yaml.tmpl > appconnect/dashboard.yaml
    ```
1. Install ACE Dashboard UI.
    ```sh
    oc apply -f ./appconnect
    ```
    Results:
    ```sh
    dashboard.appconnect.ibm.com/ace-dashboard created
    ```
1. Wait for ACE Dashboard UI to be ready. This takes 5-7 mins.
    ```sh
    oc get dashboard ace-dashboard -n ace-demo
    ```
    Results: 
    ```sh
    NAME            RESOLVEDVERSION   REPLICAS   CUSTOMIMAGES   STATUS    URL   AGE
    ace-dashboard   12.0.9.0-r1       1          false          Pending         67s
    ```
1. When installation is completed, you should see the following.
    ```sh
    oc get dashboard ace-dashboard -n ace-demo 
    ```
    Results:
    ```sh
    NAME            RESOLVEDVERSION   REPLICAS   CUSTOMIMAGES   STATUS   URL                                                                                                          AGE
    ace-dashboard   12.0.9.0-r1       1          false          Ready    https://cpd-integration.apps.<clusterID>.<domainName>/integration/run/integrations/ace-demo/ace-dashboard/   5m47s
    ```
1. Access ACE Dashboard UI to verify installation.
    1. Get URL.
        ```sh
        oc get route -n ace-demo ace-dashboard-ui -ojson | jq -r .spec.host
        ```
        Results:
        ```sh
        ace-dashboard-ui-ace-demo.apps.<clusterID>.<domainName>
        ```
    1. Get credentials.
        ```sh
        oc get secret -n ibm-common-services platform-auth-idp-credentials -ojson | jq -r .data.admin_password | base64 -d
        ```
    1. Go to private window of a browser (incognito mode) and login to ACE Dashboard UI.
        ![ACE dashboard UI](../demo-pre-reqs/images/ace-dashboard-ui.png)

## Setup Tekton Pipeline

1. Go to the folder `app-connect-tekton-pipeline`.
1. Make sure you have setup environment variable IBM_ENTITLEMENT_KEY.
1. Run the script `0-setup.sh`. This script with setup the namespace for the pipeline, setup the credentials for Git and IBM registry pull secrets, service account and permission, pipeline and its tasks.
    ```sh
    ./0-setup.sh 
    ```
    Results:
    ```sh
    > ---------------------------------------------------------------
    > checking for tekton CLI
    > ---------------------------------------------------------------
    > ---------------------------------------------------------------
    > checking for oc CLI
    > ---------------------------------------------------------------
    > ---------------------------------------------------------------
    > checking for IBM entitlement key env var
    > ---------------------------------------------------------------
    > ---------------------------------------------------------------
    > creating namespace to run pipelines in
    > ---------------------------------------------------------------
    namespace/pipeline-ace configured
    > ---------------------------------------------------------------
    > checking for github credentials for cloning the repo from a pipeline
    > ---------------------------------------------------------------
    secret/github-credentials configured
    > ---------------------------------------------------------------
    > storing docker credentials for pulling image for BAR file builder and tester
    > ---------------------------------------------------------------
    secret/ibm-entitlement-key configured
    > ---------------------------------------------------------------
    > creating service account to run the pipelines as
    > ---------------------------------------------------------------
    serviceaccount/pipeline-deployer-serviceaccount configured
    > ---------------------------------------------------------------
    > setting up permissions for the deploy pipeline
    > ---------------------------------------------------------------
    clusterrole.rbac.authorization.k8s.io/pipeline-deployer-aceflows-role unchanged
    clusterrolebinding.rbac.authorization.k8s.io/pipeline-deployer-aceflows-rolebinding unchanged
    serviceaccount/pipeline-deployer-serviceaccount configured
    > ---------------------------------------------------------------
    > adding image builder permissions
    > ---------------------------------------------------------------
    clusterrole.rbac.authorization.k8s.io/system:openshift:scc:privileged added: "pipeline-deployer-serviceaccount"
    > ---------------------------------------------------------------
    > creating tasks for the deployment pipeline
    > ---------------------------------------------------------------
    task.tekton.dev/convert-p12-to-jks configured
    task.tekton.dev/ace-create-bar-file configured
    task.tekton.dev/ace-create-configuration configured
    task.tekton.dev/ace-create-datasource configured
    task.tekton.dev/ace-create-integration-server configured
    task.tekton.dev/ace-create-policy-project configured
    task.tekton.dev/git-clone-home configured
    task.tekton.dev/read-secret configured
    task.tekton.dev/ace-run-tests configured
    task.tekton.dev/ace-update-templates-from-secrets configured
    > ---------------------------------------------------------------
    > creating deployment pipeline
    > ---------------------------------------------------------------
    pipeline.tekton.dev/pipeline-ace-integration-server configured
    ```
1. Verify the setup, go to the OpenShift console, navigate to `Pipelines > Pipelines`. You will see this pipeline configured.
    ![deploy-pipeline](../screenshots/deploy-pipeline.png)

## Deploy "simple-demo" integration server using Tekton

1. Go to the folder `app-connect-tekton-pipeline`.
1. Setup environment variables.
    ```sh
    export FILE_STORAGECLASS=ocs-storagecluster-cephfs
    export ACE_LICENSE=L-LFMR-BTD75V
    export ACE_VERSION=12.0.9.0-r1
    export ACE_IMAGE_URL=cp.icr.io/cp/appc/ace-server-prod@sha256:246828d9f89c4ed3a6719cd3e4b71b1dec382f848c9bf9c28156f78fa05bc4e7
    envsubst < simple-pipelinerun.yaml.tmpl > simple-pipelinerun.yaml
    ```
    Refer to the documentation on the values needed [Licensing reference for App Connect](https://www.ibm.com/docs/en/app-connect/containers_cd?topic=resources-licensing-reference-app-connect-operator) and [App Connect server image](https://www.ibm.com/docs/en/app-connect/containers_cd?topic=obtaining-app-connect-enterprise-server-image-from-cloud-container-registry).
1. Run the pipeline.
    ```sh
    ./1-deploy-simple-integration-server.sh
    ```
    Results:
    ```sh
    > ---------------------------------------------------------------
    > running the pipeline
    > ---------------------------------------------------------------
    pipelinerun.tekton.dev/ace-deploy-lcd99
    > ---------------------------------------------------------------
    > tailing pipeline logs
    > ---------------------------------------------------------------
    ....
    ....
    ....
    > ---------------------------------------------------------------
    > pipeline complete
    > ---------------------------------------------------------------
    ```
1. View Pipelinerun in OpenShift Console.
    ![Simple-pipelinerun](../screenshots/example-pipelinerun-simple.png)

## Test `simple-demo` integration server

1. Submit an HTTP request to the `simple-demo` flow
    ```sh
    curl "http://$(oc get route -nace-demo hello-world-http -o jsonpath='{.spec.host}')/hello"
    ```
    Results:
    ```sh
    {"hello":"world"}% 
    ```