# k8s-pks-jenkins

## Overall Guidance

* The Jenkins image and its deployment is fully tested with KOPS and DOCKERHUB
* It is also tested with PKS and HARBOR
* It is available as a ready image on my Dockerhub thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest
* Just ensure that the Jenkins cluster and YAML files are created accordingly
* Also the App Cluster and App YAMLs are deployed accordingly
* Also the credentials and pipelines are created accordingly
* This image does not install typical plugins of Kubernetes as it has pks and kubectl cli installed
* Cloudfoundry plugin is installed as it is required
* All kubernetes pipeline actions are managed with kubectl cli with credentials binding plugin (usual username/pwd) combos

## Prepare the Jenkins Docker Image

* Launch an EC2 with ubuntu 18.04
* Install docker on it

```
sudo su
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
apt-get update
apt-get install -y docker-ce
sudo usermod -aG docker ubuntu
apt-get install git -y
mkdir jenkins-cfcli-k8s-docker
git clone https://github.com/thecloudgarage/docker-jenkins.git
mv docker-jenkins/* jenkins-cfcli-k8s-docker/
vi Dockerfile
```
### Dockerfile contents (NOTE PLEASE CHANGE THE PIVOTAL TOKEN FOR YOUR ACCOUNT)

```
FROM jenkins/jenkins:lts

ENV JAVA_OPTS="-Djenkins.install.runSetupWizard=false"

ARG GIT_COMMIT=unspecified
LABEL git_commit=$GIT_COMMIT
# Run this command to find git commit:-
#docker inspect quay.io/shazchaudhry/docker-jenkins | jq '.[].ContainerConfig.Labels'

# Configure Jenkins
COPY config/*.xml $JENKINS_HOME/
COPY config/*.groovy /usr/share/jenkins/ref/init.groovy.d/

# Once jenkins is running and configured, run the following command to find the list of plugins installed:
##  curl -s -k "http://admin:admin@localhost:8080/pluginManager/api/json?depth=1" | jq -r '.plugins[].shortName' | tee plugins.txt
RUN /usr/local/bin/install-plugins.sh \
  ace-editor \
  ant \
  antisamy-markup-formatter \
  authentication-tokens \
  blueocean \
  blueocean-autofavorite \
  blueocean-commons \
  blueocean-config \
  blueocean-dashboard \
  blueocean-display-url \
  blueocean-events \
  blueocean-github-pipeline \
  blueocean-git-pipeline \
  blueocean-i18n \
  blueocean-jwt \
  blueocean-personalization \
  blueocean-pipeline-api-impl \
  blueocean-pipeline-editor \
  blueocean-pipeline-scm-api \
  blueocean-rest \
  blueocean-rest-impl \
  blueocean-web \
  bouncycastle-api \
  branch-api \
  build-timeout \
  cloudfoundry \
  cloudbees-folder \
  credentials \
  credentials-binding \
  display-url-api \
  docker-commons \
  docker-workflow \
  durable-task \
  email-ext \
  external-monitor-job \
  favorite \
  git \
  git-client \
  github \
  github-api \
  github-branch-source \
  gitlab-plugin \
  git-server \
  global-build-stats \
  gradle \
  handlebars \
  icon-shim \
  jackson2-api \
  jquery-detached \
  junit \
  keycloak \
  ldap \
  mailer \
  mapdb-api \
  matrix-auth \
  matrix-project \
  metrics \
  momentjs \
  pam-auth \
  pipeline-build-step \
  pipeline-github-lib \
  pipeline-graph-analysis \
  pipeline-input-step \
  pipeline-milestone-step \
  pipeline-model-api \
  pipeline-model-declarative-agent \
  pipeline-model-definition \
  pipeline-model-extensions \
  pipeline-rest-api \
  pipeline-stage-step \
  pipeline-stage-tags-metadata \
  pipeline-stage-view \
  plain-credentials \
  pubsub-light \
  purge-job-history \
  resource-disposer \
  role-strategy \
  scm-api \
  script-security \
  sse-gateway \
  ssh-credentials \
  ssh-slaves \
  structs \
  subversion \
  timestamper \
  token-macro \
  variant \
  windows-slaves \
  workflow-aggregator \
  workflow-api \
  workflow-basic-steps \
  workflow-cps \
  workflow-cps-global-lib \
  workflow-durable-task-step \
  workflow-job \
  workflow-multibranch \
  workflow-scm-step \
  workflow-step-api \
  workflow-support \
  ws-cleanup

USER root
# Install Docker from official repo
RUN apt-get update -qq && \
    apt-get install -qqy apt-transport-https ca-certificates curl gnupg2 software-properties-common && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add - && \
    apt-key fingerprint 0EBFCD88 && \
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable" && \
    apt-get update -qq && \
    apt-get install -qqy docker-ce && \
    usermod -aG docker jenkins && \
    chown -R jenkins:jenkins $JENKINS_HOME/

# install kubectl
RUN curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/lin
ux/amd64/kubectl
RUN chmod +x ./kubectl
RUN mv ./kubectl /usr/local/bin/kubectl

# install pks client
RUN wget --post-data="" --header="Authorization: Token PUTTOKENHERE" https://network.pivotal.io/api/v2/products/pivotal-container-service/releases/501833/product_files/528557/download -O "pks-linux-amd64-1.6.0-build.225"
RUN mv pks-linux-amd64-1.6.0-build.225 pks
RUN chmod +x ./pks
RUN mv ./pks /usr/local/bin/pks

# install cf cli
RUN wget https://anhalwaysfortest.s3.amazonaws.com/cf
RUN chmod +x ./cf
RUN mv ./cf /usr/local/bin/cf

USER jenkins

VOLUME [$JENKINS_HOME, "/var/run/docker.sock"]
```

### NOTE: OPTIONAL (AND NOT REQUIRED FOR OUR PIPELINES). BEST IF YOU SKIP

* Retag the image: docker tag thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest jenkins-revised-build:latest
* docker run -dit -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock jenkins-revised-build:latest
* Login with admin/admin
* Add the additional plugins
* Manage Jenkins > Manage Plugins > Avaialble > search for kubernetes and add kubernetes, kubernetes cli, kubernetes devops steps, kubernetes credentials, kubernetes credentials provider
* Click on Install without restart
* Now commit a new docker image from this running container

```
docker ps
docker commit <container-id> thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest
docker login
docker push thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest
docker kill the previous container
docker run -dit -p 8080:8080 -v /var/run/docker.sock:/var/run/docker.sock thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest
```
* Login into jenkins admin/admin
* Add credentials for dockerhub
* Left hand menu credentials > Stores scoped to Jenkins > Jenkins > Global credentials (unrestricted) > Add credentials > Kind username/password
* Fill in your docker hub account details and keep the ID and description as dockerhub
* Run a sample pipeline to download from github, build a docker image and push to docker hub
* Go to home > new item > pipeline > click pipeline tab and add the below pipeline script
** Eventually, the built image will be pushed to DockerHub registry thecloudgarage/k8s-jenkins-test-docker-push:latest

```
pipeline {
  environment {
    registry = "thecloudgarage/k8s-jenkins-test-docker-push"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/thecloudgarage/k8s-jenkins-test-docker-push.git'
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
  }
}
```

* In the cloned repo: 
* Delete the index.html file and its corresponding Dockerfile command COPY index.html /var/www/html. 
* And then build the pipeline.
* Once done, place back a sample index.html and the Dockerfile command COPY index.html /var/www/html
* Rerun the pipeline to view a new build

# Build from a sub-directory
* The github repo used also has a sub-directory
* Use the below pipeline with an extra step to change into a subdirectory of the repo and build the image
```
pipeline {
  environment {
    registry = "thecloudgarage/k8s-jenkins-test-docker-push"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/thecloudgarage/k8s-jenkins-test-docker-push.git'
      }
    }
    stage('change into subdirectory') {
      steps{
        sh "cd testing"
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
  }
}
```

# Jenkins running as an image on KOPS k8s cluster with DockerHub as registry. 

* This exercise just builds the image and pushes it to DockerHub. 
* It does not do any kubernetes deployments
* Launch a k8s cluster with KOPS
* Create the following files

### jenkins-storage-class.yml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2
```

### jenkins-pvc.yml

```
apiVersion: v1
metadata:
  name: jenkins-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "standard"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

### jenkins-deployment.yml 
* Please note how you run the container as level 0, and also how you mount the /var/run which is required for docker.sock)
* Also note how we use pvc to persist jenkins home directory

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  labels:
    app: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      securityContext:
        runAsUser: 0
        fsGroup: 0
      containers:
      - name: jenkins
        image: thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest
        imagePullPolicy: "Always"
        ports:
        - containerPort: 8080
        volumeMounts:
          - name: jenkins-home
            mountPath: "/var/jenkins_home"
          - name: docker-run
            mountPath: /var/run
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-pvc
        - name: docker-run
          hostPath:
            path: /var/run
```

### Lastly, create jenkins-service.yml

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30000
  selector:
    app: jenkins
```

## Jenkins pipeline 

* Go to home > new item > pipeline > click pipeline tab and add the below pipeline script
* Access the service at ec2-ip:30000 and login (30000 is listed as the nodePort value in the service yaml)
* Add docker credentials of the kind username password
* Note the registry path and credentials in the below pipeline

```
pipeline {
  environment {
    registry = "thecloudgarage/k8s-jenkins-test-docker-push:latest"
    registryCredential = 'dockerhub'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/thecloudgarage/jenkins-k8s-test.git'
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + ":$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( '', registryCredential ) {
            dockerImage.push()
          }
        }
      }
    }
    stage('Remove Unused docker image') {
      steps{
        sh "docker rmi $registry:$BUILD_NUMBER"
      }
    }
  }
}
```
* Now this pipeline and credentials will persist

# JENKINS ON PKS WITH HARBOR REGISTRY. 

* This exercise will pull the code from github, build a docker image, push it to harbor registry
* It will also update the kubernetes deployment (running in a different app cluster) 
* Before creating the cluster, configure the habor registries for tg-ad-app & nginx proxy deployment.
* Please match the registry names as specified in the YAMLs and also ensure that they match the Jenkins pipeline definitions
* Launch a PKS cluster for the application ambar-jboss-eap-7.2
* The YAML files for the ambar-jboss-eap-7.2 cluster are located here:
* The application is a jboss-eap-7.2 application with jboss-healthcheck war deployed
* The pipeline testing can be done by first removing the jboss-ashelloworld.war from github

> And also delete the ADD command from the Dockerfile
> Once the pipeline is run and verified
> You can upload the jboss-ashelloworld.war and insert the ADD command in the Dockerfile

* Deploy the files accordingly to create jboss app and its service
* Verify if the app is working

> Note the deployment YAML is configured with rolling update strategy, readiness and liveness probes to assist the jenkins automated pipeline deploys

> NOTE: Generally while logging in from a docker client into a Harbor registry we need to create a sub-directory under /etc/docker/certs.d with the harbor hostname e.g. abc.thecloudgarage.com and place the harbor root certificate ca.crt within it

> In the case of Jenkins, somehow I didn't need to do that. Initially, I felt that I will need to manually create a registry sub-directory in the certs.d of PKS node (which could be cumbersome)

> Somehow Jenkins access to harbor via username/password works without ca.crt

* Next create another PKS cluster for the Jenkins instance
* We are intentionally simulating running Jenkins on a separate cluster)

>> NOTE
>> In the case of Jenkins deployment on PKS
>> The docker.sock file is located under /var/vcap/data/sys/run/docker/docker.sock
>> Hence the Jenkins deployment YAML is different than the one used for KOPS cluster
>> Most would struggle to solve this important caveat. Luckily researched for a right way to access docker.sock

* Create the following files. Note we are running everything in default namespace (if required you can alter this)

### jenkins-storage-class.yml

```
kind: StorageClass
apiVersion: storage.k8s.io/v1beta1
metadata:
  name: standard
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp2
```

### jenkins-pvc.yml

```
apiVersion: v1
metadata:
  name: jenkins-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "standard"
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 8Gi
```

### jenkins-pks-deployment.yml

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: jenkins
  labels:
    app: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      securityContext:
        runAsUser: 0
        fsGroup: 0
      containers:
      - name: jenkins
        image: thecloudgarage/ambar-jenkins-cfcli-k8s-docker:latest
        imagePullPolicy: "Always"
        ports:
        - containerPort: 8080
        volumeMounts:
          - name: jenkins-home
            mountPath: "/var/jenkins_home"
          - name: dockersocket
            mountPath: /var/run/docker.sock
          - name: procdir
            mountPath: /host/proc
            readOnly: true
          - name: cgroups
            mountPath: /host/sys/fs/cgroup
            readOnly: true
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins-pvc
        - hostPath:
            path: /var/vcap/data/sys/run/docker/docker.sock
          name: dockersocket
        - hostPath:
            path: /proc
          name: procdir
        - hostPath:
            path: /sys/fs/cgroup
          name: cgroups
```

### Lastly, create jenkins-service.yml

```
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  type: NodePort
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30000
  selector:
    app: jenkins
```

## PKS Jenkins pipeline

> The PKS cluster name inserted in this pipeline is sampleone
> Please change it to the actual cluster name hosting tg-ad-app

* Login to jenkins by accessing the node IP and nodeport
* Add Harbor credentials named harborthecloudgarage in the form of username and password
* Then add PKS credentials named pks in the form of username and password
* Ensure the below pipeline variables (credentials, registry names, github urls) are configured appropriately
* Run the below pipeline
* The create the below pipeline

> NOTE: 
> The tg-ad-app is primarily a JAVA environment running on JBOSS EAP 7.2
> There are two deployments, first being a backend application tg-ad-app running on JBOSS EAP 7.2 and the second is a NGINX reverse proxy
> The deployment application contains a /jboss-healthcheck path running a helloworld war file
> The initial deployment is run from the default image which does not have /jboss-as-helloworld (basically jboss-as-helloworld.war) deployed
> The github repo contains this jboss-as-helloworld.war file and the Dockerfile also contains instructions to copy it to the image
> During the pipeline run, the Dockerfile will build the new image and we can observe /jboss-as-helloworld path

* NOTE: That this pipeline run is using a different github repo, registry and credentials.

```
pipeline {
  environment {
    registry = "harbor.thecloudgarage.com/tg-ad-app"
    registryCredential = 'harborthecloudgarage'
    dockerImage = ''
  }
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/thecloudgarage/k8s-tg-ad-app.git'
      }
    }
    stage('Building image') {
      steps{
        script {
          dockerImage = docker.build registry + "/tg-ad-app:$BUILD_NUMBER"
        }
      }
    }
    stage('Deploy Image') {
      steps{
        script {
          docker.withRegistry( 'https://harbor.thecloudgarage.com/tg-ad-app', registryCredential ) {
            dockerImage.push()
          }
        }
      }
	}
    stage('Deploy on cluster') {
      steps{
	   script {
	      withCredentials([usernamePassword(credentialsId: 'pks', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
            sh 'pks login -a api.pks.thecloudgarage.com --username $USERNAME --password $PASSWORD -k'
            sh 'pks get-credentials sampleone'
            sh 'kubectl config use-context sampleone'
            sh 'kubectl set image deployment tg-ad-app -n tg-ad-app tg-ad-app=harbor.thecloudgarage.com/tg-ad-admin/tg-ad-admin:$BUILD_NUMBER'
          }
		}
      }
    }
  }
}
```

> NOTE: To support the above pipeline, the app needs to be configured as below


### Example: tg-ad-app.yml (It's there in the github repo)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tg-ad-app
  namespace: tg-ad-app
  labels:
    app: tg-ad-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tg-ad-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 25%
  template:
    metadata:
      labels:
        app: tg-ad-app
    spec:
      containers:
      - name: tg-ad-app
        image: harbor.thecloudgarage.com/tg-ad-app/tg-ad-app:latest
        ports:
        - name: tg-ad-http
          containerPort: 8080
        - name: tg-ad-https
          containerPort: 8443
        - name: tg-ad-mgmt
          containerPort: 9990
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /jboss-healthcheck
            port: 8080
          initialDelaySeconds: 5
        readinessProbe:
          httpGet:
             path: /jboss-healthcheck
             port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
```

* Once the pipeline is run., you will see the old POD will terminate and new one will
* have the new image and build number as generated by the pipeline

# JENKINS CLOUDFOUNDRY (PWS) PIPELINE FOR CF PUSH

* Need to have a github project
* In my case I tested a static website (github repo is in the pipeline) 
* So along with the project files
* We will also need a Staticfile (empty)
* Plus we also need a manifest.yml (without manifest the app does not get pushed from jenkins)

### manifest.yml

```
---
applications:
- name: anhstaticwebsite
```

* Then create a regular username/password credential named 'pws' in jenkins
* Since this pipeline deploys over Pivotal Web Services I am using my pws userid/password
* Change the API endpoint based on the PCF PAS deployment accordingly

> Note that we already have a Cloudfoundry plugin installed that gives the ability to define the API, ORG, SPACE and CREDENTIALS using the credentials binding plugin of Jenkins

* Create a pipeline in jenkins... alter the variables accordingly....

```
pipeline {
  agent any
  stages {
    stage('Cloning Git') {
      steps {
        git 'https://github.com/thecloudgarage/staticwebsite.git'
      }
    }
    stage('Deploy on PWS') {
      steps{
	   script {
	      pushToCloudFoundry(
  target: 'api.run.pivotal.io',
  organization: 'letsplay',
  cloudSpace: 'Production',
  credentialsId: 'pws'
)
          }
		}
      }
    }
  }
```

AND RUN THE PIPELINE.... IT WORKS!!!!!

