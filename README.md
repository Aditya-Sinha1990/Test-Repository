FCM Deployment Docs &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; &emsp; ![logo](images/1_1.png =80x)
<br />

![FCM LOGIN](images/fcm_images/1_1.png =950x450)
<br /><br />
---
## Contents
- [FCM HELM CHART DETAILS](##1.-fcm-helm-chart-details)
- [Pre-requisites](#2.-pre-requisites)
- [FCM Deployment Pipelines](#3.-fcm-deployment-pipelines)
- [FCM Image and Container Registries](#4.-fcm-image-and-container-registries)
- [FCM Testing with T24)](#5.-environment-active-gate-deployment-saas-dynatrace)
- [Contributors](#7.-contributors)
- [LICENSE](#8.-license)

<br /><br /><br />

---
## 1. FCM Helm Chart Details
```textmate
FCM is divied based on the Release version and Image Version 

1. FCM 202009-SNAPSHOT
2. FCM 202101
```

<br /><br /><br />

---
## 2. Pre-requisites
- [x] FCM Images to be available in ctsdevelopimages CR
- [x] Making sure correct version and tags are present
- [x] Variables are configured in variables dir as per the pipeline
    - 202009 -> FLOWE
    - 202101 -> FRONT2BACK
- [ ] [Read About FCM?](https://temenosgroup.sharepoint.com/:p:/s/FCM/EQP3fIjnvq1PklZfgoe_TlQBi3UdqIJrCkN8kZsbtTyTMg?e=6UBxNSs)

<br /><br /><br />

---
## 3. Fcm Deployment Pipelines
- _**Example**_
    1. **flowe** `First Environment which host FCM 202009 images`
    1. **flowedev** `Additional Dev Environment which host FCM 202009 images`
    1. **Front2Back* `First Environment which host FCM 202101 images`
    1. **Front2BackDev* `Additional Dev Environment which host FCM 202101 images`
  

<br /><br /><br />
  
---  
## 4. FCM Image and Container Registries
- **About:**
    - Azure Container Registry is a managed, private Docker registry service based on the open-source Docker Registry 2.0. Create and maintain Azure container registries to store and manage your private Docker container images and related artifacts.
    - Use _*Azure container registries*_ with your existing container development and deployment pipelines, or use Azure Container Registry Tasks to build container images in Azure
    - ![Container Registries with FCM repos](images/fcm_images/1_2.png =700x300)

<br />

- **Steps:**
    1. This is being taken care by our _*./templates/dynatrace-deployment.yml*_ present in _*pipelines/fcm-deployment*_.
    1. [`Dynatrace DockerHub Repo`](https://hub.docker.com/u/dynatrace). As this deployment requires two images in build process.
       <br />
        - `dynatrace-oneagent-operator` ![Docker Image Version (latest semver)](https://img.shields.io/docker/v/dynatrace/dynatrace-oneagent-operator/v0.8.2?style=plastic)
          <br />
        - `oneagent` ![Docker Image Version (latest semver)](https://img.shields.io/docker/v/dynatrace/oneagent/latest?style=plastic)
          <br />

        -  ```bash
           docker pull dynatrace/dynatrace-oneagent-operator:v0.8.2
           docker pull dynatrace/oneagent
           ```           
    1. One can pull these images locally, tag them locally and push to Azure Container Registry`fcmtr` or can be a part of docker build images pipeline.
    1. Below code is the main working code behind `dynatrace-helm-chart` located in `helm` directory.
    1. ```bash
        kubectl create namespace dynatrace && \
        helm install dynatrace-oneagent-operator $(Build.Repository.LocalPath)/helm/dynatrace-oneagent-operator \
        -n dynatrace --values $(Build.Repository.LocalPath)/helm/dynatrace-oneagent-operator/values.yaml \
        --set platform="kubernetes",\
        oneagent.apiUrl="https://prb67049.live.dynatrace.com/api",\
        operator.image="fcmtcr.azurecr.io/dynatrace/dynatrace-oneagent-operator:v0.8.2",\
        oneagent.image="fcmtcr.azurecr.io/dynatrace/oneagent:latest"
       ```
    1. Working OneAgent-operator and One-Agent Pods can be checked through Buildvm `kubectl get pods --namespace=dynatrace`

        - ![Dynatrace Pods Running](images/1_6.png =800x200)

    1. The *values.yaml* file in _helm/dynatrace-oneagent-operator_ contains the necessary variables that is overidden using above bash.
    1. We can pass the necessary arguments and properties tag in: _*helm/dynatrace-oneagent-operator/values.yaml*_
        - ![Arguments and Tags](images/1_7.png =400x300)  &emsp; ![Arguments and Tags Passed](images/1_8.png =400x300)


<br /><br /><br />

---
## 5. Environment Active Gate Deployment SaaS Dynatrace
- **About:**
    - ActiveGate knows about the runtime structure of our Dynatrace environment and routes messages from OneAgents to the correct server endpoints.
    - To connect our Kubernetes clusters to Dynatrace server to take advantage of the dedicated Kubernetes/OpenShift overview page, we need to run an `Environemnt ActiveGate` in our environment (version 1.163+) when working with `one-agent-operator`.
    - In case OneAgents don't have access to the internet, you should install an ActiveGate to serve as a single access point, rather than opening the firewall for multiple hosts running OneAgents.
    - This approach greatly reduces the effort of managing and maintaining firewall and/or proxy configuration settings.
    - ActiveGate authenticates OneAgent requests (SSL handshake and environment ID authentication).
    - Currently it gets deployed on to the `buildvm`

<br />

- **Steps:**
    1. This is being taken care by our _*./templates/dynatrace-deployment.yml*_ present in _*pipelines/fcm-deployment*_.
    1. Activate gate downloads a shell script which needs to be placed on `Exceutable mode` and `run as a root`. Below is the main working code.
    1. ```bash
        wget -O Dynatrace-ActiveGate-Linux-x86-1.201.92.sh \
        "https://prb67049.live.dynatrace.com/api/v1/deployment/installer/gateway/unix/latest?arch=x86&flavor=default" \
        --header="Authorization: Api-Token $(dpasstoken)"
        chmod u+x Dynatrace-ActiveGate-Linux-x86-1.201.92.sh
        sudo ./Dynatrace-ActiveGate-Linux-x86-1.201.92.sh
        ```
    1. Active gates `helps to compress` heaps of data being passed from `oneagent` and send it to dynatrace server. Where wkith help of AI and ML, it get displayed on dashboards/Dynatrace UI.
    1. ![Dynatrace ActiveGate](images/1_4.png =300x450)
    1. ActiveGates can only send data to higher hierarchy levels. It is impossible to send data to the same or lower level of the hierarchy.

<br /><br /><br />

---
## 6. Dynatrace Kubernetes Credential Connection
- **About:**
    - This is required to poll the api server of AKS.
    - It's very similar to any other kubernetes dashboard which allows to show the current resources of Kubernetes.
    - This is being used with help of OneAgent-Monitoring Service Account Credentials.
    - Require to be able to poll`Kubernetese API-Server-Endpoint`, need to use other options for `Private AKS Cluster`.

<br />

- **Steps:**
    1. This is configured with help of script _*./scripts/DynatraceScripts/createKubeConnection.py*_.
    1. Details of requirements are in docstring:
    1. ```pydocstring
           """
           This script makes rest calls to dynatrace server to create a new kubernetese credential and get all existing ones
       
           Script Requires 4 Environment Variables for successful run:
                - Need to set in azure vault (preprovision.yaml)
                   - APITOKEN
       
                - Script Run Time variables (Automated)
                    - API_URL_ENDPOINT
                    - API_BEARER_TOKEN
                    - API_CLUSTER_NAME
           """
        ```
    1. Once the Connection is Done details will be showing in here:
        - ![Dynatrace Kubernetese Credential](images/1_5.png =650x300)


<br /><br />
---
## 7. Contributors
- [Aditya Sinha](mailto:msinha@temenos.com)
- [Sabitha K R](mailto:sabithakr@temenos.com)
- [Praveen](mailto:dpraveenkumar@temenos.com)
  <br /><br />

---
## 8. LICENSE
- [License To Internal Use](images/fcm_images/LICENSE)
