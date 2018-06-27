# Instructions

- Install odo

  ```bash
  brew install odo
  ```

- Git clone project
  
  ```bash
  git clone https://github.com/snowdrop/ocp-odo-build-install.git && cd ocp-odo-build-install
  ```

- Log on to OpenShift and create project

  ```bash
  oc login $(minishift ip):8443 -u admin -p admin
  oc new-project ocp-odo-build-install
  ```
  
- Add the OpenJDK-1.8 S2I Build Image as it is not installed by default on minishift - ocp 3.9
  ```bash
  oc create -f is-openjdk18.yaml
  ``` 

- Create new spring-boot-http component (= create a new application, DeploymentConfig, Service)

  ```bash
  odo create openjdk18 --git https://github.com/snowdrop/ocp-odo-build-install.git
  ```
  **REMARK** : 
  - Deployment of the pod will fail as a [missing ENV var](https://github.com/redhat-developer/odo/issues/501) is not defined to specify the uberjar file to be used !!
  
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

