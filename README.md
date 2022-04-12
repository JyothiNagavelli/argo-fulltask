Argo Workflows And Argo Events
Scenario:-
In this article we will talk about Argo Workflow and Events. So here Whenever we made changes in GitHub it automatically trigger the Argo workflow by using Argo Events.
Prerequisites:-
1.Cluster
2.Docker Hub
3.GitHub
Argo Workflows:- Argo Workflows one of the more popular open-source tools for orchestrating parallel jobs in Kubernetes. Just look here
Argo Events:- Argo Events is an event-driven workflow automation for Kubernetes which helps you trigger K8s objects, Argo Workflows etc. You can check it here

Get started:
Here, I have one nodejs application on my GitHub. So whenever i made changes in GitHub i need to automatically build an image and push into Docker Hub with new changes. For building the image i am using kaniko in Argo Workflows. kaniko is a tool to build container images from a Dockerfile. make sure docker installed in your system.
Now, First create one kubernetes cluster and install Argo Workflows and Argo Events inside the cluster.
Argo Workflows installation
$ kubectl create ns argo
$ kubectl apply -n argo -f  https://raw.githubusercontent.com/argoproj/argo-workflows/master/manifests/quick-start-postgres.yaml
After installation do port-forwarding so you can access Argo Workflow UI
$ kubectl -n argo port-forward deployment/argo-server 2746:2746
(or)
And we have another way for access Argo Workflows UI. default service for argo-server is ClusterIP so we are changing into NodePort or LoadBalencer(LB)
$ kubectl edit svc <service name> -n argo
On browser hit https://localhost:<NodePort> or LB URL.
We get UI like this

Also install Argo CLI to communicate with cluster, using this commands
$ curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.2.8/argo-linux-amd64.gz
$ unzip argo-linux-amd64.gz
$ chmod +x argo-linux-amd64
$ mv ./argo-linux-amd64 /usr/local/bin/argo
$ argo version
$ argo --help
Here we need to create secret for kaniko with Docker Hub (or any image registry) credentials for pushing images into Docker Hub. For that we need to export credentials
export REGISTRY_SERVER=https://index.docker.io/v1/
export REGISTRY_USER=<dockerhub username>
export REGISTRY_PASS=<password>
export REGISTRY_EMAIL=<name@gmail.com>
Create secret in Argo namespace with this command
kubectl create secret docker-registry regcred --docker-server=$REGISTRY_SERVER --docker-username=$REGISTRY_USER --docker-password=$REGISTRY_PASS --docker-email=$REGISTRY_EMAIL -n argo
Now we create template for kaniko. In this template we give few parameters for workflow with this kaniko-tempalte.yaml file
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: container-image
spec:
  templates:
  - name: build-kaniko-git
    inputs:
      parameters:
      - name: app_repo
      - name: container_image
      - name: container_tag
    container:
      image: gcr.io/kaniko-project/executor:debug
      command: [/kaniko/executor]
      args:
      - --context={{inputs.parameters.app_repo}}
      - --destination={{inputs.parameters.container_image}}:{{inputs.parameters.container_tag}}
      - --force
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker/
Create template with argo CLI command
argo cluster-template create <file name> -n argo

template created like this in UI
For manual purpose use this workflow yaml file for building image and push into Docker Hub.
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: build-container-image-
  labels:
    workflows.argoproj.io/archive-strategy: "false"
spec:
  entrypoint: build
  serviceAccountName: argo
  volumes:
  - name: kaniko-secret
    secret:
      secretName: regcred
      items:
        - key: .dockerconfigjson
          path: config.json
  templates:
  - name: build
    dag:
      tasks:
      - name: build
        templateRef:
          name: container-image
          template: build-kaniko-git
          clusterScope: true
        arguments:
          parameters:
          - name: app_repo
            value: git://<git repo url>
          - name: container_image
            value: <dockerhub/imagename>
          - name: container_tag
            value: <tag>
Now Submit this file
argo submit <file name> -n argo
In UI workflow created like this and kaniko pushed our docker image into Docker Hub


Argo workflows in UI
Now we will automate this process. For that we need Argo Events
Argo Events installation

This image belongs to Argo Events
$ kubectl create namespace argo-events
$ kubectl apply -f 
https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/
install.yaml
$ kubectl apply -f 
https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/
install-validating-webhook.yaml
$ kubectl apply -n argo-events -f 
https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/
eventbus/native.yaml
Now create a secret for git-hub token, as shown below and apply it
apiVersion: v1
kind: Secret
metadata:
 name: github-access
type: Opaque
data:
token: <git token that is encoded with base64>
We need to give permission for role as cluster-admin. Use this file and apply it.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2022-03-29T11:08:50Z"
  name: cluster-admin-0
  resourceVersion: "150613"
  uid: 24dcb71a-d32f-417b-a8ab-9449d633becc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
  name: operate-workflow-sa
  namespace: argo-events
Now create a service for event source, and apply in argo-events namespace
apiVersion: v1
kind: Service
metadata:
  name: webhook-eventsource
spec:
  ports:
   - port: 12000
     protocol: TCP
     targetPort: 12000
  selector:
    eventsource-name: github
  type: NodePort
Install ngrok in your system here
Now with the help of ngrok, port forward the event source service port, as follow
ngrok tcp <port number>

ngrok page
This ngrok URL we need to mention in Event-source yaml file
Event-source:-
Now create a event-source for creating Webhook on GitHub. Event-source programmatically configures webhooks for projects on GitHub and helps sensor trigger the workloads on events.
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github
spec:
  selector:
    eventsource-name: github          # event source name
  github:
    nodejs-welcome:                   # event name
      repositories:
        - owner: <git hub user name>
          names:
            - <name of the repo 1>
            - <name of the repo 2>
      webhook:
        endpoint: /nodejs-welcome
        port: "12000"
        method: POST
        url: <url that is generated by ngrok>
      events:
        - "*"
      apiToken:
        name: github-access
        key: token
      insecure: true
      active: true
      contentType: json
Apply this file using following command
kubectl apply -f  <file>.yaml  -n argo-events


Webhooks on GitHub
Whenever redeliver it comes healthy.
Sensor:-
Now create a sensor and apply in argo-event namespace. Sensor defines a set of event dependencies and triggers . It listens to events on the eventbus and acts as an event dependency manager to resolve and execute the triggers.
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: github
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: test-dep
      eventSourceName: github       # event source name 
      eventName: nodejs-welcome     # event name 
  triggers:
    - template:
        name: argo-workflow-trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
		  metadata:
		    generateName: nodejs-nithya
		    namespace: argo
  		    labels:
			workflows.argoproj.io/archive-strategy: "false"
  		  spec:
                entrypoint: hello
		    serviceAccountName: argo
		    volumes:
			- name: kaniko-secret
			  secret:
			    secretName: regcred
			    items:
				- key: .dockerconfigjson
				  path: config.json
  		    templates:
		      - name: helloArgo Workflows And Argo Events
Scenario:-
In this article we will talk about Argo Workflow and Events. So here Whenever we made changes in GitHub it automatically trigger the Argo workflow by using Argo Events.
Prerequisites:-
1.Cluster
2.Docker Hub
3.GitHub
Argo Workflows:- Argo Workflows one of the more popular open-source tools for orchestrating parallel jobs in Kubernetes. Just look here
Argo Events:- Argo Events is an event-driven workflow automation for Kubernetes which helps you trigger K8s objects, Argo Workflows etc. You can check it here

Get started:
Here, I have one nodejs application on my GitHub. So whenever i made changes in GitHub i need to automatically build an image and push into Docker Hub with new changes. For building the image i am using kaniko in Argo Workflows. kaniko is a tool to build container images from a Dockerfile. make sure docker installed in your system.
Now, First create one kubernetes cluster and install Argo Workflows and Argo Events inside the cluster.
Argo Workflows installation
$ kubectl create ns argo
$ kubectl apply -n argo -f  https://raw.githubusercontent.com/argoproj/argo-workflows/master/manifests/quick-start-postgres.yaml
After installation do port-forwarding so you can access Argo Workflow UI
$ kubectl -n argo port-forward deployment/argo-server 2746:2746
(or)
And we have another way for access Argo Workflows UI. default service for argo-server is ClusterIP so we are changing into NodePort or LoadBalencer(LB)
$ kubectl edit svc <service name> -n argo
On browser hit https://localhost:<NodePort> or LB URL.
We get UI like this

Also install Argo CLI to communicate with cluster, using this commands
$ curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.2.8/argo-linux-amd64.gz
$ unzip argo-linux-amd64.gz
$ chmod +x argo-linux-amd64
$ mv ./argo-linux-amd64 /usr/local/bin/argo
$ argo version
$ argo --help
Here we need to create secret for kaniko with Docker Hub (or any image registry) credentials for pushing images into Docker Hub. For that we need to export credentials
export REGISTRY_SERVER=https://index.docker.io/v1/
export REGISTRY_USER=<dockerhub username>
export REGISTRY_PASS=<password>
export REGISTRY_EMAIL=<name@gmail.com>
Create secret in Argo namespace with this command
kubectl create secret docker-registry regcred --docker-server=$REGISTRY_SERVER --docker-username=$REGISTRY_USER --docker-password=$REGISTRY_PASS --docker-email=$REGISTRY_EMAIL -n argo
Now we create template for kaniko. In this template we give few parameters for workflow with this kaniko-tempalte.yaml file
apiVersion: argoproj.io/v1alpha1
kind: ClusterWorkflowTemplate
metadata:
  name: container-image
spec:
  templates:
  - name: build-kaniko-git
    inputs:
      parameters:
      - name: app_repo
      - name: container_image
      - name: container_tag
    container:
      image: gcr.io/kaniko-project/executor:debug
      command: [/kaniko/executor]
      args:
      - --context={{inputs.parameters.app_repo}}
      - --destination={{inputs.parameters.container_image}}:{{inputs.parameters.container_tag}}
      - --force
      volumeMounts:
        - name: kaniko-secret
          mountPath: /kaniko/.docker/
Create template with argo CLI command
argo cluster-template create <file name> -n argo

template created like this in UI
For manual purpose use this workflow yaml file for building image and push into Docker Hub.
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: build-container-image-
  labels:
    workflows.argoproj.io/archive-strategy: "false"
spec:
  entrypoint: build
  serviceAccountName: argo
  volumes:
  - name: kaniko-secret
    secret:
      secretName: regcred
      items:
        - key: .dockerconfigjson
          path: config.json
  templates:
  - name: build
    dag:
      tasks:
      - name: build
        templateRef:
          name: container-image
          template: build-kaniko-git
          clusterScope: true
        arguments:
          parameters:
          - name: app_repo
            value: git://<git repo url>
          - name: container_image
            value: <dockerhub/imagename>
          - name: container_tag
            value: <tag>
Now Submit this file
argo submit <file name> -n argo
In UI workflow created like this and kaniko pushed our docker image into Docker Hub


Argo workflows in UI
Now we will automate this process. For that we need Argo Events
Argo Events installation

This image belongs to Argo Events
$ kubectl create namespace argo-events
$ kubectl apply -f 
https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/
install.yaml
$ kubectl apply -f 
https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/
install-validating-webhook.yaml
$ kubectl apply -n argo-events -f 
https://raw.githubusercontent.com/argoproj/argo-events/stable/examples/
eventbus/native.yaml
Now create a secret for git-hub token, as shown below and apply it
apiVersion: v1
kind: Secret
metadata:
 name: github-access
type: Opaque
data:
token: <git token that is encoded with base64>
We need to give permission for role as cluster-admin. Use this file and apply it.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  creationTimestamp: "2022-03-29T11:08:50Z"
  name: cluster-admin-0
  resourceVersion: "150613"
  uid: 24dcb71a-d32f-417b-a8ab-9449d633becc
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
  name: operate-workflow-sa
  namespace: argo-events
Now create a service for event source, and apply in argo-events namespace
apiVersion: v1
kind: Service
metadata:
  name: webhook-eventsource
spec:
  ports:
   - port: 12000
     protocol: TCP
     targetPort: 12000
  selector:
    eventsource-name: github
  type: NodePort
Install ngrok in your system here
Now with the help of ngrok, port forward the event source service port, as follow
ngrok tcp <port number>

ngrok page
This ngrok URL we need to mention in Event-source yaml file
Event-source:-
Now create a event-source for creating Webhook on GitHub. Event-source programmatically configures webhooks for projects on GitHub and helps sensor trigger the workloads on events.
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: github
spec:
  selector:
    eventsource-name: github          # event source name
  github:
    nodejs-welcome:                   # event name
      repositories:
        - owner: <git hub user name>
          names:
            - <name of the repo 1>
            - <name of the repo 2>
      webhook:
        endpoint: /nodejs-welcome
        port: "12000"
        method: POST
        url: <url that is generated by ngrok>
      events:
        - "*"
      apiToken:
        name: github-access
        key: token
      insecure: true
      active: true
      contentType: json
Apply this file using following command
kubectl apply -f  <file>.yaml  -n argo-events


Webhooks on GitHub
Whenever redeliver it comes healthy.
Sensor:-
Now create a sensor and apply in argo-event namespace. Sensor defines a set of event dependencies and triggers . It listens to events on the eventbus and acts as an event dependency manager to resolve and execute the triggers.
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: github
spec:
  template:
    serviceAccountName: operate-workflow-sa
  dependencies:
    - name: test-dep
      eventSourceName: github       # event source name 
      eventName: nodejs-welcome     # event name 
  triggers:
    - template:
        name: argo-workflow-trigger
        argoWorkflow:
          operation: submit
          source:
            resource:
              apiVersion: argoproj.io/v1alpha1
              kind: Workflow
		  metadata:
		    generateName: nodejs-nithya
		    namespace: argo
  		    labels:
			workflows.argoproj.io/archive-strategy: "false"
  		  spec:
                entrypoint: hello
		    serviceAccountName: argo
		    volumes:
			- name: kaniko-secret
			  secret:
			    secretName: regcred
			    items:
				- key: .dockerconfigjson
				  path: config.json
  		    templates:
		      - name: hello
			  dag:
			    tasks:
				- name: build-container-image
				  templateRef:
				    name: container-image
				    template: build-kaniko-git
				    clusterScope: true
				  arguments:
				    parameters:
					- name: app_repo
					  value: git://<git repo url>
					- name: container_image
					  value: <name of the doker image>
					- name: container_tag
	       			  value: "tag"
Now made a change in the git, that will reflect in the workflow. Then check it in the UI, or check it in CLI.

workflow in Argo UI
So finally our image pushed into Docker Hub. Now check in Docker Hub Repository.

Docker images in Docker Hub
SUMMARY:-
Thus, we can trigger Argo Workflows with Argo Events to create Docker images and push into Docker Hub per every git commit or push. We can use different event source and sensor in Argo Events to trigger the workflows or deployments.
Reference:-
Argo Workflows - here
Argo Events - here
ngrok -here





			  dag:
			    tasks:
				- name: build-container-image
				  templateRef:
				    name: container-image
				    template: build-kaniko-git
				    clusterScope: true
				  arguments:
				    parameters:
					- name: app_repo
					  value: git://<git repo url>
					- name: container_image
					  value: <name of the doker image>
					- name: container_tag
	       			  value: "tag"
Now made a change in the git, that will reflect in the workflow. Then check it in the UI, or check it in CLI.

workflow in Argo UI
So finally our image pushed into Docker Hub. Now check in Docker Hub Repository.

Docker images in Docker Hub
SUMMARY:-
Thus, we can trigger Argo Workflows with Argo Events to create Docker images and push into Docker Hub per every git commit or push. We can use different event source and sensor in Argo Events to trigger the workflows or deployments.
Reference:-
Argo Workflows - here
Argo Events - here
ngrok -here




