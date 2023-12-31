## StreamSets SDK Kubernetes Deployment Example with Ingress
This project provides an example of how to use the [StreamSets Platform SDK](https://docs.streamsets.com/platform-sdk/latest/index.html) to programmatically create a [Kubernetes Deployment](https://docs.streamsets.com/portal/platform-controlhub/controlhub/UserGuide/Deployments/Kubernetes.html#concept_ec3_cqg_hvb) of [Data Collector](https://streamsets.com/products/data-collector-engine/) (SDC) using an Advanced Kubernetes configuration (as described in Step 2 [here](https://docs.streamsets.com/portal/platform-controlhub/controlhub/UserGuide/Deployments/Kubernetes.html#task_xvp_g1n_jvb)) that includes a Kubernetes Service and Ingress, with HTTPS-based access to the deployed Data Collectors.  This approach may be necessary if [Direct REST APIs](https://docs.streamsets.com/portal/platform-controlhub/controlhub/UserGuide/Engines/Communication.html#concept_dt2_hq3_34b) must be used rather than [WebSocket Tunneling](https://docs.streamsets.com/portal/platform-controlhub/controlhub/UserGuide/Engines/Communication.html#concept_hbg_fq3_34b).

### Prerequisites

- A Python 3.6+ environment with the StreamSets Platform SDK v5.1+ module installed 

- The StreamSets Organization should have WebSocket communication disabled

- An existing Kubernetes cluster with the ability to deploy an Ingress Controller. For this example I will use [ingress-nginx](https://kubernetes.github.io/ingress-nginx/deploy/) on Azure Kubernetes Service (AKS)

- [API Credentials](https://docs.streamsets.com/portal/platform-controlhub/controlhub/UserGuide/OrganizationSecurity/APICredentials_title.html#concept_vpm_p32_qqb) for a user with permissions to create Deployments 

### Configuration Details
- This project deploys one or more SDCs, each within its own deployment, with a Service and Ingress per SDC.

- TLS is terminated at the ingress controller, with options to have the Ingress Controller use either <code>https</code> or <code>http</code> as the backend protocol to forward requests to SDCs.  If using <code>https</code> as the backend protocol, set a custom keystore for SDC as described below. Regardless of the backend protocol, SDC will have an <code>https</code> URL, so in either case, SDC will function as an Authoring Engine.

- Path-based routing will be configured for the Ingress

### Test Environment
- AKS using Kubernetes version v1.25.11 

- ingress-nginx version 1.6.4

- SDC version 5.6.0

### Create a Namespace for the StreamSets Deployments
Create a namespace for the StreamSets Deployment:

<code>$ kubectl create ns ns1</code>

### Deploy the Ingress Controller
I installed ingress-nginx v1.6.4 on AKS using the command:

<code>$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.6.4/deploy/static/provider/cloud/deploy.yaml</code>

### Get the External IP for the Ingress Controller's Load Balancer Service

<img src="images/external-ip.png" alt="external-ip" width="700"/>

### Map a DNS Name to the Load Balancer's IP
I added a record to my DNS to map the name <code>aks.onefoursix.com</code> to the Load Balancer's external IP, and confirm that using nslookup:

	$  nslookup aks.onefoursix.com
	Server:	8.8.8.8
	Address:	8.8.8.8#53

	Non-authoritative answer:
	Name:	aks.onefoursix.com
	Address: 20.69.83.54

### Store a TLS key and cert for the Load Balancer in a Secret
I'll use a wildcard cert and key for <code>*.onefoursix.com</code> in the files <code>tls.crt</code> and <code>tls.key</code> respectively. Store the TLS key and cert in a Kubernetes tls Secret:

	$ kubectl -n ns1 create secret tls streamsets-tls \
    	--key ~/certs/tls.key --cert ~/certs/tls.crt

This step is not necessary if the load balancer is a Layer-4 load balancer just doing TCP passthrough without terminating TLS. In this example, the load balancer is a Layer-7 load balancer and does terminate TLS, and can then, optionally, use TLS to talk to backend SDCs.



### If SDCs are configured to use https themselves, store a custom keystore and keystore password in a Secret
If the backend protocol is <code>https</code> we'll need to set a custom keystore for SDC. I'll package the TLS key and cert used for the Load Balancer into a keystore named <code>onefoursix.jks</code>, and save that keystore in a Kubernetes Secret named <code>sdc-keystore</code>:

	$ kubectl -n ns1 create secret generic sdc-keystore --from-file=onefoursix.jks
	
I'll also save the keystore password in a secret named <code>sdc-keystore-password</code>

	$ kubectl -n ns1 create secret generic sdc-keystore-password --from-file=keystore-password.txt


### Create a Kubernetes Environment
I'll create a new Kubernetes Environment named <code>aks-ns1</code> and set the namespace to <code>ns1</code>. Activate the Environment but do not play the generated shell command; instead, click the <code>View Kubernetes YAML</code> button. In the dialog that opens, click the <code>copy</code> button. Paste the text into a text document on your local machine named <code>agent.yaml</code> (the name is not critical).

#### Edit the generated YAML for the Agent
On or around line 21, replace this line:

    resources: ["pods"]

with this line:

	resources: ["pods", "services"]

And add this new section:

```
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "create", "patch", "delete"]
```

Those changes will allow the Kubernetes Agent to create Service and Ingress resources

The updated rules section in the Role resource should look like this:

```
rules:
  - apiGroups: [""]
    resources: ["pods", "services"]
    verbs: ["get", "list", "create", "patch", "delete"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "create", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "create", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "create", "patch", "delete"]
  - apiGroups: ["autoscaling"]
    resources: ["horizontalpodautoscalers"]
    verbs: ["get", "list", "create", "patch", "delete"]
```


Also, if you need to set a proxy for the Kubernetes Agent, set that on or around line 102 as the value for the environment variable <code>STREAMSETS_KUBERNETES_AGENT_JAVA_OPTS</code> (for details on this step, see the docs [here](https://docs.streamsets.com/portal/platform-controlhub/controlhub/UserGuide/Environments/Kubernetes.html#task_tgv_rrg_lwb)).


#### Deploy the Agent
Apply the <code>agent.yaml</code> script to deploy the Agent:

	$ kubectl apply -f agent.yaml

Make sure the Agent comes online for the Environment:

<img src="images/agent.png" alt="agent" width="700"/>

### Configure a Kubernetes Deployment
Clone this project to your local machine. 

#### Edit deployment.properties
Edit the file <code>deployment.properties</code> in the root of the project. 

See the comments in the [deployment.properties](deployment.properties) file for comments about each property.

Here is an example <code>deployment.properties</code> file for my environment that specifies <code>https</code> as the backend protocol, sets a custom keystore, and uses the template <code>yaml/sdc-service-ingress-keystore.yaml</code> :

```
[deployment]
SCH_URL=https://na01.hub.streamsets.com
ORG_ID=8030c2e9-xxxx-xxxx-xxxx-97c8d4369386
ENVIRONMENT_NAME=aks-ns1
LOAD_BALANCER_HOSTNAME=aks.onefoursix.com
BACKEND_PROTOCOL=https
SDC_KEYSTORE=onefoursix.jks

SDC_DEPLOYMENT_MANIFEST=yaml/sdc-service-ingress-keystore.yaml
SDC_VERSION=5.6.0

DEPLOYMENT_TAGS=k8s-sdc-5.6.0,california

USER_STAGE_LIBS=apache-kafka_3_4,aws,aws-secrets-manager-credentialstore,jdbc,jython_2_7,sdc-snowflake

ENGINE_LABELS=dev,california

SDC_MAX_CPU_LOAD=80.0
SDC_MAX_MEMORY_USED=70.0
SDC_MAX_PIPELINES_RUNNING=10

SDC_JAVA_MIN_HEAP_MB=2024
SDC_JAVA_MAX_HEAP_MB=2024
SDC_JAVA_OPTS=""

REQUESTS_MEMORY=3Gi
LIMITS_MEMORY=4Gi
REQUESTS_CPU=1000m
LIMITS_CPU=3000m
```

#### Edit additional config files  (optional) 

Edit any of the additional files in the project's <code>etc</code> directory as needed, including:

	credential-stores.properties
	proxy.properties
	sdc-log4j2.properties
	security.policy
	
 
Note that the included <code>sdc.properties</code> file is a template that contains tokens like <code>${SDC_BASE_HTTP_URL}</code> that will be replaced by the Python script.  Change any other properties as needed.

You can avoid storing sensitive values in the <code>credential-stores.properties</code> file by setting values in the file like this:

```
credentialStore.aws.config.access.key=${file("aws-access-key.txt")}
credentialStore.aws.config.secret.key=${file("aws-secret-key.txt")}
```
Store the sensitive values in Kubernetes Secrets, and then add Volume and VolumeMount sections to the manifest used in the project, like tthis:

```
    spec:
      containers:
        - env:
          ...
          volumeMounts:
            - mountPath: /etc/sdc/aws-access-key.txt
              name: aws-access-key
              subPath: aws-access-key.txt
            - mountPath: /etc/sdc/aws-secret-key.txt
              name: aws-secret-key
              subPath: aws-secret-key.txt
      ...
      volumes:
        - name: aws-access-key
          secret:
            secretName: aws-access-key
        - name: aws-secret-key
          secret:
            secretName: aws-secret-key

```
Of course, it's far better to use the <code>instanceProfile</code> security method rather than access keys, but if you have to use keys, at least store them in Secrets rather than in the properties files.

 
#### Edit the deployment manifest (optional) 

There are two manifest templates included in the project: 

- <code>yaml/sdc-service-ingress-keystore.yaml</code> for <code>https</code> backend protocol
- <code>yaml/sdc-service-ingress.yaml</code> for <code>http</code> backend protocol

Edit the manifest file template you need.  Note that many of the property values, like <code>${DEP_ID}</code> will be replaced by the Python script. 


### Set your API Credentials
Set your [API credentials] in the file <code>private/sdk-env.sh</code> as quoted strings with no spaces, like this:

	export CRED_ID="esdgew……193d2"
	export CRED_TOKEN="eyJ0…………J9."

### Deploy a single SDC
Make the script <code>create-k8s-deployment.sh</code> executable:

	$ chmod +x create-k8s-deployment.sh

Execute the script, passing it two args: the name of your StreamSets Kubernetes Environment and the Deployment suffix.  For example:

	./create-k8s-deployment.sh aks-ns1 sdc1

If all goes well you should see console output like this:
```
$ ./create-k8s-deployment.sh aks-ns1 sdc1
---------
Creating StreamSets Deployment
Environment Name aks-ns1
DEPLOYMENT_SUFFIX: sdc1
---------

2023-08-24 21:47:24 Connecting to Control Hub
2023-08-24 21:47:25 Getting the environment
2023-08-24 21:47:25 Found environment aks-ns1
2023-08-24 21:47:25 Using namespace ns1
2023-08-24 21:47:25 Creating a deployment builder
2023-08-24 21:47:25 Creating deployment aks-ns1-sdc1
2023-08-24 21:47:28 Adding Stage Libs: apache-kafka_3_4,aws,aws-secrets-manager-credentialstore,jdbc,jython_2_7,sdc-snowflake
2023-08-24 21:47:28 Loading sdc.properties
2023-08-24 21:47:28 Setting SDC URL to https://aks.onefoursix.com/sdc1/
2023-08-24 21:47:28 Setting SDC http.port to -1
2023-08-24 21:47:28 Setting SDC https.port to 18630
2023-08-24 21:47:28 Setting SDC keystore to onefoursix.jks
2023-08-24 21:47:28 Loading credential-stores.properties
2023-08-24 21:47:28 Loading security.policy
2023-08-24 21:47:28 Loading sdc-log4j2.properties
2023-08-24 21:47:28 Loading proxy.properties
2023-08-24 21:47:29 Using yaml template: yaml/sdc-service-ingress-keystore.yaml
2023-08-24 21:47:29 Setting DEP_ID to 422ba761-6ab1-4deb-9b3b-f8db43c32d61
2023-08-24 21:47:29 Setting NAMESPACE to ns1
2023-08-24 21:47:29 Setting SDC_VERSION to 5.6.0
2023-08-24 21:47:29 Setting ORG_ID to 8030c2e9-1a39-xxxx-xxxx-97c8d4369386
2023-08-24 21:47:29 Setting SCH_URL to https://na01.hub.streamsets.com
2023-08-24 21:47:29 Setting REQUESTS_MEMORY to 3Gi
2023-08-24 21:47:29 Setting LIMITS_MEMORY to 4Gi
2023-08-24 21:47:29 Setting REQUESTS_CPU to 1000m
2023-08-24 21:47:29 Setting LIMITS_CPU to 3000m
2023-08-24 21:47:29 Setting DEPLOYMENT_SUFFIX to sdc1
2023-08-24 21:47:29 Setting LOAD_BALANCER_HOSTNAME to aks.onefoursix.com
2023-08-24 21:47:29 Setting KEYSTORE to onefoursix.jks
2023-08-24 21:47:29 Setting BACKEND_PROTOCOL to HTTPS
2023-08-24 21:47:29 Done
```

#### Inspect the Deployment
You should see a new Deployment in Control Hub in a Deactivated state.  (You can uncomment the second to last line in the Python script to have Deployments autostart once you have confidence in the process.)

<img src="images/deployment.png" alt="deployment" width="700"/>

Inspect the stage libraries, labels, and all other configuration to confirm the Deployment was created as desired.  

When editing the Deployment you should be placed in the Advanced Kubernetes pane and you can inspect the generated manifest, including the Service and Ingress:

<img src="images/advanced.png" alt="advanced" width="700"/>

### Start the Deployment

Start the Deployment and it should transition to Activating and then to Active:

<img src="images/active.png" alt="active" width="700"/>

The Engine should register with Control Hub with its path-based URL:

<img src="images/engine.png" alt="engine" width="700"/>

The Data Collector should be accessible for Authoring:

<img src="images/accessible.png" alt="accessible" width="700"/>

### Create multiple deployments
Make the script <code>create-multiple-k8s-deployments.sh</code> executable: 

	$ chmod +x create-multiple-k8s-deployments.sh

Execute the script, passing it two args: the name of your StreamSets Kubernetes Environment, and a comma-delimited list of Deployment suffixes.  For example:

	$ ./create-multiple-k8s-deployments.sh aks-ns1 sdc1,sdc2,sdc3

If all goes well you should see console output like this:

```
$  ./create-multiple-k8s-deployments.sh aks-ns1 sdc1,sdc2,sdc3
---------
Creating StreamSets Deployments
Environment Name aks-ns1
Deployment Suffix List: sdc1,sdc2,sdc3
---------

---------
Creating StreamSets Deployment
Environment Name aks-ns1
SDC Suffix: sdc1
---------
2023-08-24 21:51:05 Connecting to Control Hub
2023-08-24 21:51:06 Getting the environment
2023-08-24 21:51:06 Found environment aks-ns1
2023-08-24 21:51:06 Using namespace ns1
2023-08-24 21:51:06 Creating a deployment builder
2023-08-24 21:51:06 Creating deployment aks-ns1-sdc1
2023-08-24 21:51:08 Adding Stage Libs: apache-kafka_3_4,aws,aws-secrets-manager-credentialstore,jdbc,jython_2_7,sdc-snowflake
2023-08-24 21:51:08 Loading sdc.properties
2023-08-24 21:51:08 Setting SDC URL to https://aks.onefoursix.com/sdc1/
2023-08-24 21:51:08 Setting SDC http.port to -1
2023-08-24 21:51:08 Setting SDC https.port to 18630
2023-08-24 21:51:08 Setting SDC keystore to onefoursix.jks
2023-08-24 21:51:08 Loading credential-stores.properties
2023-08-24 21:51:08 Loading security.policy
2023-08-24 21:51:08 Loading sdc-log4j2.properties
2023-08-24 21:51:08 Loading proxy.properties
2023-08-24 21:51:08 Using yaml template: yaml/sdc-service-ingress-keystore.yaml
2023-08-24 21:51:08 Setting DEP_ID to 88030393-9cce-4310-9c89-c2af4077caf3
2023-08-24 21:51:08 Setting NAMESPACE to ns1
2023-08-24 21:51:08 Setting SDC_VERSION to 5.6.0
2023-08-24 21:51:08 Setting ORG_ID to 8030c2e9-1a39-xxxx-xxxx-97c8d4369386
2023-08-24 21:51:08 Setting SCH_URL to https://na01.hub.streamsets.com
2023-08-24 21:51:08 Setting REQUESTS_MEMORY to 3Gi
2023-08-24 21:51:08 Setting LIMITS_MEMORY to 4Gi
2023-08-24 21:51:08 Setting REQUESTS_CPU to 1000m
2023-08-24 21:51:08 Setting LIMITS_CPU to 3000m
2023-08-24 21:51:08 Setting DEPLOYMENT_SUFFIX to sdc1
2023-08-24 21:51:08 Setting LOAD_BALANCER_HOSTNAME to aks.onefoursix.com
2023-08-24 21:51:08 Setting KEYSTORE to onefoursix.jks
2023-08-24 21:51:08 Setting BACKEND_PROTOCOL to HTTPS
2023-08-24 21:51:08 Done
---------
Creating StreamSets Deployment
Environment Name aks-ns1
SDC Suffix: sdc2
---------
2023-08-24 21:51:09 Connecting to Control Hub
2023-08-24 21:51:10 Getting the environment
2023-08-24 21:51:10 Found environment aks-ns1
2023-08-24 21:51:10 Using namespace ns1
2023-08-24 21:51:10 Creating a deployment builder
2023-08-24 21:51:10 Creating deployment aks-ns1-sdc2
2023-08-24 21:51:12 Adding Stage Libs: apache-kafka_3_4,aws,aws-secrets-manager-credentialstore,jdbc,jython_2_7,sdc-snowflake
2023-08-24 21:51:12 Loading sdc.properties
2023-08-24 21:51:12 Setting SDC URL to https://aks.onefoursix.com/sdc2/
2023-08-24 21:51:12 Setting SDC http.port to -1
2023-08-24 21:51:12 Setting SDC https.port to 18630
2023-08-24 21:51:12 Setting SDC keystore to onefoursix.jks
2023-08-24 21:51:12 Loading credential-stores.properties
2023-08-24 21:51:12 Loading security.policy
2023-08-24 21:51:12 Loading sdc-log4j2.properties
2023-08-24 21:51:12 Loading proxy.properties
2023-08-24 21:51:12 Using yaml template: yaml/sdc-service-ingress-keystore.yaml
2023-08-24 21:51:12 Setting DEP_ID to 4fbfa037-6fe2-4c15-b913-b3769a4ab669
2023-08-24 21:51:12 Setting NAMESPACE to ns1
2023-08-24 21:51:12 Setting SDC_VERSION to 5.6.0
2023-08-24 21:51:12 Setting ORG_ID to 8030c2e9-1a39-xxxx-xxxx-97c8d4369386
2023-08-24 21:51:12 Setting SCH_URL to https://na01.hub.streamsets.com
2023-08-24 21:51:12 Setting REQUESTS_MEMORY to 3Gi
2023-08-24 21:51:12 Setting LIMITS_MEMORY to 4Gi
2023-08-24 21:51:12 Setting REQUESTS_CPU to 1000m
2023-08-24 21:51:12 Setting LIMITS_CPU to 3000m
2023-08-24 21:51:12 Setting DEPLOYMENT_SUFFIX to sdc2
2023-08-24 21:51:12 Setting LOAD_BALANCER_HOSTNAME to aks.onefoursix.com
2023-08-24 21:51:12 Setting KEYSTORE to onefoursix.jks
2023-08-24 21:51:12 Setting BACKEND_PROTOCOL to HTTPS
2023-08-24 21:51:13 Done
---------
Creating StreamSets Deployment
Environment Name aks-ns1
SDC Suffix: sdc3
---------
2023-08-24 21:51:13 Connecting to Control Hub
2023-08-24 21:51:14 Getting the environment
2023-08-24 21:51:14 Found environment aks-ns1
2023-08-24 21:51:14 Using namespace ns1
2023-08-24 21:51:14 Creating a deployment builder
2023-08-24 21:51:14 Creating deployment aks-ns1-sdc3
2023-08-24 21:51:16 Adding Stage Libs: apache-kafka_3_4,aws,aws-secrets-manager-credentialstore,jdbc,jython_2_7,sdc-snowflake
2023-08-24 21:51:16 Loading sdc.properties
2023-08-24 21:51:16 Setting SDC URL to https://aks.onefoursix.com/sdc3/
2023-08-24 21:51:16 Setting SDC http.port to -1
2023-08-24 21:51:16 Setting SDC https.port to 18630
2023-08-24 21:51:16 Setting SDC keystore to onefoursix.jks
2023-08-24 21:51:16 Loading credential-stores.properties
2023-08-24 21:51:16 Loading security.policy
2023-08-24 21:51:16 Loading sdc-log4j2.properties
2023-08-24 21:51:16 Loading proxy.properties
2023-08-24 21:51:16 Using yaml template: yaml/sdc-service-ingress-keystore.yaml
2023-08-24 21:51:16 Setting DEP_ID to 7b9844db-cf45-4494-bcc7-1eab7f36fc67
2023-08-24 21:51:16 Setting NAMESPACE to ns1
2023-08-24 21:51:16 Setting SDC_VERSION to 5.6.0
2023-08-24 21:51:16 Setting ORG_ID to 8030c2e9-1a39-xxxx-xxxx-97c8d4369386
2023-08-24 21:51:16 Setting SCH_URL to https://na01.hub.streamsets.com
2023-08-24 21:51:16 Setting REQUESTS_MEMORY to 3Gi
2023-08-24 21:51:16 Setting LIMITS_MEMORY to 4Gi
2023-08-24 21:51:16 Setting REQUESTS_CPU to 1000m
2023-08-24 21:51:16 Setting LIMITS_CPU to 3000m
2023-08-24 21:51:16 Setting DEPLOYMENT_SUFFIX to sdc3
2023-08-24 21:51:16 Setting LOAD_BALANCER_HOSTNAME to aks.onefoursix.com
2023-08-24 21:51:16 Setting KEYSTORE to onefoursix.jks
2023-08-24 21:51:16 Setting BACKEND_PROTOCOL to HTTPS
2023-08-24 21:51:17 Done
```

#### Start the new Deployments

<img src="images/start3.png" alt="start3" width="700"/>

Wait for all of the Deployments to become Active:

<img src="images/active3.png" alt="active3" width="700"/>

Confirm all the engines are accessible:

<img src="images/access3.png" alt="access3" width="700"/>

