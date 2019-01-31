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
  
- Create an application which represents the microservices or components that we will install
  ```bash
  odo app create springbootapp
  ```
  
- Create a new SpringBoot's odo component with the Github project. Maven build will take place

  ```bash
  odo create openjdk18 sb1 --git https://github.com/snowdrop/ocp-odo-build-install.git
  ```

  **WARNING** : The deployment of the pod will fail as a [missing ENV var](https://github.com/redhat-developer/odo/issues/501) is not declared to specify the uberjar file to be used. Then apply the following env var oc command on the `BuildConfig` resource and restart the build:

  ```
  ctrl-c
  oc cancel-build sb1-springbootapp-1
  oc env bc/sb1-springbootapp ARTIFACT_COPY_ARGS=*-exec.jar 
  oc start-build sb1-springbootapp
  ```
  
  **IMPORTANT** : If you try to upload the source using this command `odo create openjdk18 sb1 --local ./src`, then the supervisord's sidecar will not work
  
  ```
  Cloning "https://github.com/kadel/bootstrap-supervisored-s2i " ...
	Commit:	ab5f0c21325c0bb7d7c5e187b7a9fc430c987f2d (Merge pull request #1 from mik-dass/test)
	Author:	Tomas Kral <tomas.kral@gmail.com>
	Date:	Mon Jun 4 15:22:37 2018 +0200
  + set -eo pipefail
  + PATH=/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/s2i
  + HOME=/opt/app-root
  + curl -s -o /tmp/get-pip.py https://bootstrap.pypa.io/get-pip.py 
  + /usr/bin/python /tmp/get-pip.py --user
  The directory '/opt/app-root/.cache/pip/http' or its parent directory is not owned by the current user and the cache has been   disabled. Please check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
  The directory '/opt/app-root/.cache/pip' or its parent directory is not owned by the current user and caching wheels has been   disabled. check the permissions and owner of that directory. If executing pip with sudo, you may want sudo's -H flag.
  Collecting pip
  Downloading https://files.pythonhosted.org/packages/5f/25/e52d3f31441505a5f3af41213346e5b6c221c9e086a166f3703d2ddaf940/pip-  18.0-py2.py3-none-any.whl  (1.3MB)
  Collecting setuptools
  Downloading      https://files.pythonhosted.org/packages/66/e8/570bb5ca88a8bcd2a1db9c6246bb66615750663ffaaeada95b04ffe74e12/setuptools-40.2.0-py2.py3-none-any.whl  (568kB)
  Collecting wheel
  Downloading https://files.pythonhosted.org/packages/81/30/e935244ca6165187ae8be876b6316ae201b71485538ffac1d718843025a9/wheel-0.31.1-py2.py3-none-any.whl  (41kB)
  Installing collected packages: pip, setuptools, wheel
  Could not install packages due to an EnvironmentError: [Errno 13] Permission denied: '/opt/app-root'
  Check the permissions.
  error: build error: non-zero (13) exit code from registry.access.redhat.com/redhat-openjdk-18/openjdk18- openshift@sha256:dc84fed0f6f40975a2277c126438c8aa15c70eeac75981dbaa4b6b853eff61a6
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
