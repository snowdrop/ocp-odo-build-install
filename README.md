# Scenario to use odo and a Spring Boot application

## PREREQUISITES 

- Minishift version `>= 1.22`
- `admin-user` addon installed.

## Instructions

- [Install odo](https://github.com/redhat-developer/odo#installation) on Macos. Version tested is `0.0.18`

  ```bash
  sudo curl -L https://github.com/redhat-developer/odo/releases/download/v0.0.18/odo-darwin-amd64 -o /usr/local/bin/odo && chmod +x /usr/local/bin/odo
  ```
  
  or using brew tool
  ```bash
  brew tap kadel/odo
  brew install kadel/odo/odo
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
 
- Install the official Red Hat OpenJDK-1.8 S2I Build Image using this command : 
  ```bash
  oc import-image openjdk18 --from=registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift --confirm -n openshift
  ```
  
  **REMARK** : Next patch it to add the missing `builder` annotation
  ```bash
  oc annotate istag/openjdk18:latest tags=builder -n openshift
  ```
  
  OR alternatively, install the image using an imagestream within the openshift namespace. This imagestream contains the builder annotation that odo is looking for
  to populate its catalog
  ```bash
  oc apply -f is-java-s2i.yml -n openshift
  ```
  
- Compile and package the project locally
  ```bash
  mvn clean package
  ```
  
- Create an application which represents the microservices or components that we will install
  ```bash
  odo app create springbootapp
  Creating application: springbootapp in project: demo
  Switched to application: springbootapp in project: demo
  ```
  
  during this step, odo is creating a config file
  ```bash
  cat ~/.kube/odo             
  activeApplications:
  - active: true
    activeComponent: ""
    name: springbootapp
    project: demo
  settings: {}
  ```
  
- Check if our Java Builder image is well installed on OpenShift
  ```bash
  odo catalog list components
  NAME                           PROJECT       TAGS
  dotnet                         openshift     2.0,latest
  httpd                          openshift     2.4,latest
  java                           openshift     8,latest
  nginx                          openshift     1.10,1.12,1.8,latest
  nodejs                         openshift     0.10,10,4,6,8,8-RHOAR,latest
  perl                           openshift     5.16,5.20,5.24,5.26,latest
  php                            openshift     5.5,5.6,7.0,7.1,latest
  python                         openshift     2.7,3.3,3.4,3.5,3.6,latest
  redhat-openjdk18-openshift     openshift     1.0,1.1,1.2,1.3,1.4
  ruby                           openshift     2.0,2.2,2.3,2.4,2.5,latest
  wildfly                        openshift     10.0,10.1,11.0,12.0,13.0,8.1,9.0,latest
  ```
  
- Create a new SpringBoot's odo `component` using this the Github project.

  ```bash
  odo create redhat-openjdk18-openshift:1.4 sb1 --git https://github.com/snowdrop/ocp-odo-build-install.git
  ✓   Checking component
  ✓   Checking component version
  ✓   Creating component sb1
  ✓   Triggering build from git
  OK  Component 'sb1' was created and ports 8080/TCP,8443/TCP,8778/TCP were opened
  OK  Component 'sb1' is now set as active component
  ```
  
  **Remark** : This command is creating a BuildConfig's file and will not at all use the `inner` loop but instead the `outerloop`

  **WARNING** : The deployment of the pod will fail as an [ENV var](https://github.com/redhat-developer/odo/issues/501) is not declared to specify the `uberjar` file to be used.
  Then apply the following env var on the `BuildConfig` resource and restart the build:

  ```
  oc set env bc/sb1-springbootapp ARTIFACT_COPY_ARGS=*-exec.jar 
  oc start-build sb1-springbootapp
  ```

- Let's delete the component
  ```bash
  odo delete sb1
  Are you sure you want to delete sb1 from springbootapp? [y/N]: y
   ✓   Deleting component sb1
   OK  Component sb1 from application springbootapp has been deleted
  ```   
  
- Create a new component where we will upload the code from the local directory instead of using git binary build
  ```bash
  odo create redhat-openjdk18-openshift:1.3 sb2 --local ./
  ✓   Checking component
  ✓   Checking component version
  ✓   Creating component sb2
  OK  Component 'sb2' was created and ports 8778/TCP,8080/TCP,8443/TCP were opened
  OK  Component 'sb2' is now set as active component
  To push source code to the component run 'odo push'

  OR
  
  odo create openjdk18:latest sb1 --binary ./target/ocp-odo-build-install-1.0-exec.jar
  ✓   Checking component
  ✓   Checking component version
  ✓   Creating component sb1
  ✓   Creating component sb1
  OK  Component 'sb1' was created and ports 8080/TCP,8443/TCP,8778/TCP were opened
  OK  Component 'sb1' is now set as active component
  ```  
  
- Now push the code developed locally
  ```bash
  odo push
  Pushing changes to component: sb1
   ✓   Waiting for pod to start
   ✓   Copying files to pod
   ✓   Building component
   OK  Changes successfully pushed to component: sb1
  ```
  
- Access the service/endpoint 
  ```bash
  odo url create --port 8080 sb1
  Adding URL to component: sb1
   OK  URL created for component: sb1
  
  sb1 - http://sb1-springbootapp-demo.192.168.99.50.nip.io
  ```  
  
- Cleanup
  ```bash
  oc delete all --all
  ```    
  
**Steps**
 
- During the execution of the `odo create openjdk18 --git` command, the following resources will be created: imageStream, buildConfig, deploymentConfig and service
- Next, a pod will be created, a supervisord added to be able to call the service requesting to compile and next to restart the microservice (= java -jar)
- The code is git cloned (or pushed), then the s2i script responsible to do the mvn compilation will take place.
- When the compilation is finished, then the script `assemble-and-restart` is called by the supervisord
- The microservices is (re)started
