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

- Create new spring-boot-http component (= create a new application + BuildConfig)

  ```bash
  odo create openjdk18
  ```

- Push the source code as binary stream

  ```bash
  odo push
  ```
  
**Steps**
 
- During the execution of the `odo create openjdk18` command, the following resources will be created:
  - buildconfig, is, deploymentConfig, service, route
- A s2i build will take place using as source the following GIT repo `https://github.com/kadel/bootstrap-supervisored-s2i` 
- Next, the code source is pushed as binary stream and a build is started to generate the SpringBoot docker image
  

