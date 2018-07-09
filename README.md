# Scenario to use odo and a Spring Boot application

## PREREQUISITES 

- Minishift `3.9` using Centos ISO `1.9.0` as we can't install image from red hat registry using latest centos distro `(> 1.9.0)` due to a missing Red Hat CA Cert not installed locally and available for docker to pull images from Red Hat Registry server 
- `admin-user` addon installed.

## Instructions

- [Install odo](https://github.com/redhat-developer/odo#installation) on Macos

  ```bash
  sudo sh -c 'curl -L https://github.com/redhat-developer/odo/releases/download/v0.0.6/odo-darwin-amd64.gz | gzip -d > /usr/local/bin/odo; chmod +x /usr/local/bin/odo'
  ```

- Next, git clone the following Spring Boot project
  
  ```bash
  git clone https://github.com/snowdrop/ocp-odo-build-install.git && cd ocp-odo-build-install
  ```

- Log on to OpenShift and create a new project

  ```bash
  oc login $(minishift ip):8443 -u admin -p admin
  oc new-project ocp-odo-build-install
  ```
 
- Install the official Red Hat OpenJDK-1.8 S2I Build Image using the following command : 
  ```bash
  oc create -f is-all-openjdk18.yaml
  ```
  
  **REMARK** : If you try to install the official image using the command `oc import-image` then the annotation tag's builder needed by odo is removed !
  
- Create a new SpringBoot's odo component which means, create a new application, buildConfig, DeploymentConfig & Service

  ```bash
  odo create openjdk18 sb1 --git https://github.com/snowdrop/ocp-odo-build-install.git
  ```

  **WARNING** : The deployment of the pod will fail as a [missing ENV var](https://github.com/redhat-developer/odo/issues/501) is not declared to specify the uberjar file to be used. Then apply the following env var oc command on the `BuildConfig` resource and restart the build:

  ```
  ctrl-c
  oc cancel-build sb1-1
  oc env bc/sb1 ARTIFACT_COPY_ARGS=*-exec.jar 
  oc start-build sb1
  ```

- Cleanup
  ```bash
  oc delete all --all
  oc delete pvc/openjdk18-s2idata
  ```  
  
**Steps**
 
- During the execution of the `odo create openjdk18 --git` command, the following resources will be created:
  - is, deploymentConfig, service, route
- Next A pod will be created, a supervisord added to be able to call the service requesting to compile and next to restart the microservice (= java -jar)
- The code is git cloned (or pushed), then the s2i script resposnible to do the mvn compilation will take place.
- When the compilation is finished, then the script `assemble-and-restart` is called by the supervisord
- The microservices is (re)started

**Issue**

- OpenJDK-1.8 S2I image can be installed using the minishift's `xpaas addon` and if you want to install the latest image which is `1.4.4`, then use this command : 
  ```bash
  oc apply -f https://raw.githubusercontent.com/openshift/openshift-ansible/release-3.9/roles/openshift_examples/files/examples/v3.9/xpaas-streams/openjdk18-image-stream.json -n openshift
  ```

