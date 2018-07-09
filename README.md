# Instructions

**PREREQUISITE** : Minishift `3.9` using Centos ISO `1.9.0` as we can't install image from red hat registry using latest centos distro `(> 1.9.0)` and `admin-user` addon installed !

- [Install odo](https://github.com/redhat-developer/odo#installation) on Macos

  ```bash
  sudo sh -c 'curl -L https://github.com/redhat-developer/odo/releases/download/v0.0.6/odo-darwin-amd64.gz | gzip -d > /usr/local/bin/odo; chmod +x /usr/local/bin/odo'
  ```
  
  **Remark** : Alternative approach is to compile the project if `go` is installed on your machine
  ```bash
  cd $GOPATH/src
  go get -u github.com/redhat-developer/odo/...
  cd github.com/redhat-developer/odo
  make install
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
 
- Install the OpenJDK-1.8 S2I Build Image, as defined within this project, using this command as it is not installed with the `community` image on ocp : 
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

**Issue**

- OpenJDK-1.8 S2I image can be installed using the minishift's `xpaas addon` and if you want to install the latest image which is `1.4.4`,   then use this command : 
  ```bash
  oc apply -f https://raw.githubusercontent.com/openshift/openshift-ansible/release-3.9/roles/openshift_examples/files/examples/v3.9/xpaas-streams/openjdk18-image-stream.json -n openshift
  ```

