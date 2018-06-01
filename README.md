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

- Create new spring-boot-http component (= create a new application, BuildConfig, DeploymentConfig, Service)

  ```bash
  odo create openjdk18 --git https://github.com/snowdrop/ocp-odo-build-install.git
  ```
  
  **REMARK** : Deployment of the pod will fail as a missing ENV var is not defined to specify the uberjar file to be used !!
  
- Cleanup
  ```bash
  oc delete all --all
  oc delete pvc/openjdk18-s2idata
  ```  
  
**Steps**
 
- During the execution of the `odo create openjdk18 --git` command, the following resources will be created:
  - buildconfig, is, deploymentConfig, service, route
- A s2i build will take place using as input source the GIT repo passed as parameter
- When the build is finished, then a docker image of the Spring Boot application is deployed as ImageStream
- The DeploymentConfig is then triggered and a pod of the application will be created

