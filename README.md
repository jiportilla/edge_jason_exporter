# JSON Exporter Service

This is an example of creating and using an edge exporter service which scrapes JSON by JSONPath for [prometheus](https://prometheus.io/) monitoring

Based on [https://github.com/prometheus-community/json_exporter](https://github.com/prometheus-community/json_exporter)

- [Preconditions for Using the JSON Exporter Service](#preconditions)
- [Configuring the JSON Exporter Service](#configuring)
- [Building and Publishing the JSON Exporter Service](#building)
- [Using the JSON Exporter Service with Deployment Policy](#using-JSON-exporter)
![Prometheus architecture ](docs/prometheus-design.png)

## <a id=preconditions></a> Preconditions for Using the JSON Exporter Service

If you have not done so already, you must do these steps before proceeding with the JSON Exporter service:

1. Install the Horizon management infrastructure (exchange and agbot).

	*Also see [one-click Management Hub installation example](https://github.com/open-horizon/devops/blob/master/mgmt-hub/README.md)

2. Install the Horizon agent on your edge device and configure it to point to your Horizon exchange.

3. As part of the infrasctucture installation process for IBM Edge Computing Manager a file called `agent-install.cfg` was created that contains the values for `HZN_ORG_ID` and the exchange and css url values. Locate this file and set those environment variables in your shell now:

```bash
eval export $(cat agent-install.cfg)
```

 - **Note**: if for some reason you disconnected from ssh or your command line closes, run the above command again to set the required environment variables.

4. In addition to the file above, an API key associated with your Horizon instance would have been created, set the exchange user credentials, and verify them:

```bash
export HZN_EXCHANGE_USER_AUTH=iamapikey:<horizon-API-key>
hzn exchange user list
```

5. Choose an ID and token for your edge node, create it, and verify it:

```bash
export HZN_EXCHANGE_NODE_AUTH="<choose-any-node-id>:<choose-any-node-token>"
hzn exchange node create -n $HZN_EXCHANGE_NODE_AUTH
hzn exchange node confirm
```

## <a id=configuring></a> Configuring the JSON Exporter Service

You should complete these steps before proceeding with the JSON Exporter service:

1. List the event logs for the current or all registrations in the edge device with:

`hzn eventlog list -l`

2. Verify is a valid JSON format, for example:

```json
{
    "record_id": "1",
    "timestamp": "2020-11-03 15:32:35 +0000 UTC",
    "severity": "info",
    "message": "Workload service containers for mycluster/json.exporter are up and running.",
    "event_code": "container_running",
    "source_type": "agreement",
    "event_source": {
      "agreement_id": "f10d230a3dcdd3e6beee137e89f485daeb0da78e27c6eb87589a26b00402242c",
      "workload_to_run": {
        "url": "json.exporter",
        "org": "mycluster",
        "version": "1.0.0",
        "arch": "amd64"
      },
      "dependent_services": [],
      "consumer_id": "IBM/mycluster-agbot",
      "agreement_protocol": "Basic"
    }
  }
```

3. Update the provided `config.yml` file to select which JSON elements will be exposed by the JSON exporter service using JSONPath, for example for messages with **severity=info** use

```
- name: EventLog
  type: object
  path: $[*]?(@.severity == "info")
  labels:
    environment: edge_development
    id: $.record_id
    source_type: $.source_type
    event_code: $.event_code
    message: $.message
  values:
    info: 1
    timestamp: $.timestamp
```

## <a id=building></a>Building and Publishing the JSON Exporter Service

1. Clone this git repository:

```bash
cd ~   # or wherever you want
git clone git@github.com:jiportilla/edge_json_exporter.git
cd ~/edge_json_exporter/
```

2. Set the values in `horizon/hzn/json` to your liking. These variables are used in the service file. They are also used in some of the commands in this procedure. After editing `horizon/hzn.json`, set the variables in your environment:

```bash
export ARCH=$(hzn architecture)
eval $(hzn util configconv -f horizon/hzn.json)
```

3. Build the docker image:

```bash
make build
```

For example, when using the default values provided in this github repo [hnz.json](https://github.com/jiportilla/edge_json_exporter/blob/master/horizon/hzn.json) configuration file:


```bash
docker build --network="host" -t iportilla/jexporter_amd64:1.0.0 -f ./Dockerfile.amd64 .
```

3. You are now ready to publish your edge service, so that it can be deployed to real edge nodes. Instruct Horizon to push your docker image to your registry and publish your service in the Horizon Exchange:

```bash
hzn exchange service publish -f horizon/service.definition.json
hzn exchange service list
```

See [Developing an edge service for devices](https://www-03preprod.ibm.com/support/knowledgecenter/SSFKVV_4.1/devices/developing/developing.html) for additional details.

## <a id=using-JSON-exporter></a> Using the JSON Exporter Service with Deployment Policy

The Horizon Policy mechanism offers an alternative to using Deployment Patterns. Policies provide much finer control over the deployment placement of edge services. Policies also provide a greater separation of concerns, allowing Edge Nodes owners, Service code developers, and Business owners to each independently articulate their own Policies. There are three types of Horizon Policies:

1. Node Policy (provided at registration time by the node owner)

2. Service Policy (may be applied to a published Service in the Exchange)

3. Deployment Policy (which approximately corresponds to a Deployment Pattern)

### Node Policy

- As an alternative to specifying a Deployment Pattern when you register your Edge Node, you may register with a Node Policy.


1. Below is the file provided in `policy/node.policy.json` with this example:

```json
{
  "properties": [
     {
      "name": "state",
      "value": "configured"
    },
    {
      "name": "openhorizon.allowPrivileged",
      "value": true
    },
  ],
  "constraints": [
  	"purpose == monitoring"
  ]
}
```

- It provides values for two `properties` (**state** and **openhorizon.allowPrivileged**), that will affect which service(s) get deployed to this edge device, and states one `constraint` (**purpose == monitoring**).

The node registration step will be completed in the next section.


### Service Policy

Like the other two Policy types, Service Policy contains a set of `properties` and a set of `constraints`. The `properties` of a Service Policy could state characteristics of the Service code that Node Policy authors or Business Policy authors may find relevant. The `constraints` of a Service Policy can be used to restrict where this Service can be run. The Service developer could, for example, assert that this Service requires a particular hardware setup such as CPU/GPU constraints, memory constraints, specific sensors, actuators or other peripheral devices required, etc.


1. Below is the file provided in  `policy/service.policy.json` with this example:

```json
{
  "properties": [
  	 {
      "name": "purpose",
      "value": "monitoring"
    }
  ],
  "constraints": [
       "openhorizon.arch == amd64"
  ]
}
```

- Note this simple Service Policy provides one `property`, and it states one `constraint`. This example `constraint` is one that a Service developer might add, stating that their Service must only run on cpu architecture named `amd64`. If you recall the Node Policy we used above, the openhorizon.arch `property` is set to `amd64` during registration, so this Service should be compatible with our Edge device.

2. If needed, run the following commands to set the environment variables needed by the `service.policy.json` file in your shell:

```bash
export ARCH=$(hzn architecture)
eval $(hzn util configconv -f horizon/hzn.json)
```

3. Optionally, add or replace the service policy in the Horizon Exchange for this JSON Exporter Service:

```bash
make publish-service-policy
```
For example:
```bash
hzn exchange service addpolicy -f policy/service.policy.json json.exporter_1.0.0_amd64

```

4. View the pubished service policy attached to `json.exporter` edge service:

```bash
hzn exchange service listpolicy json.exporter_1.0.0_amd64
```

- Notice that Horizon has again automatically added some additional `properties` to your Policy. These generated property values can be used in `constraints` in Node Policies and Deployment Policies.

- Now that you have set up the Policy for your Edge Node and the published Service policy is in the exchange, we can move on to the final step of defining a Deployment Policy to tie them all together and cause software to be automatically deployed on your Edge Node.


### Deployment Policy

Deployment Policy (sometimes called Business Policy) is what ties together Edge Nodes, Published Services, and the Policies defined for each of those, making it roughly analogous to the Deployment Patterns you have previously worked with.

Deployment Policy, like the other two Policy types, contains a set of `properties` and a set of `constraints`, but it contains other things as well. For example, it explicitly identifies the Service it will cause to be deployed onto Edge Nodes if negotiation is successful, in addition to configuration variable values, performing the equivalent function to the `-f horizon/userinput.json` clause of a Deployment Pattern `hzn register ...` command. The Deployment Policy approach for configuration values is more powerful because this operation can be performed centrally (no need to connect directly to the Edge Node).

1. Below is the file provided in  `policy/deployment.policy.json` with this example:

```json
{
  "label": "Deployment policy for $SERVICE_NAME",
  "description": "A super-simple JSON Exporter Service demo",
  "service": {
    "name": "$SERVICE_NAME",
    "org": "$HZN_ORG_ID",
    "arch": "$ARCH",
    "serviceVersions": [
      {
        "version": "$SERVICE_VERSION",
        "priority":{}
      }
    ]
  },
  "properties": [],
  "constraints": [
        "state == configured"
  ],
  "userInput": [
    {
      "serviceOrgid": "$HZN_ORG_ID",
      "serviceUrl": "$SERVICE_NAME",
      "serviceVersionRange": "[0.0.0,INFINITY)",
      "inputs": [
      ]
    }
  ]
}
```

- This simple example of a Deployment Policy provides one `constraint` (`state`) that is satisfied by one of the `properties` set in the `node.policy.json` file, so this Deployment Policy should successfully deploy our JSON Exporter Service onto the Edge device.

- At the end, the userInput section has the same purpose as the `horizon/userinput.json` files provided for other examples if the given services requires them. In this case the example service defines does not have configuration variables.

2. If needed, run the following commands to set the environment variables needed by the `deployment policy.json` file in your shell:

```bash
export ARCH=$(hzn architecture)
eval $(hzn util configconv -f horizon/hzn.json)

optional: eval export $(cat agent-install.cfg)
```

3. Publish this Deployment Policy to the Exchange to deploy the `json.exporter` service to the Edge device (give it a memorable name):


**todo: update Makefile **

```bash
make publish-business-policy
```

For example:
```bash
hzn exchange deployment addpolicy -f policy/deployment.policy.json json.exporter.dp

```

4. Verify the Deployment policy:

```bash
hzn exchange deployment listpolicy json.exporter.dp
```

- The results should look very similar to your original `deployment.policy.json` file, except that `owner`, `created`, and `lastUpdated` and a few other fields have been added.




( --------- **verify** device registration cmd with policy.json)



```bash
hzn register -policy f horizon/node.policy.json
```
 - **Note**: using the `-s` flag with the `hzn register` command will cause Horizon to wait until agreements are formed and the service is running on your edge node to exit, or alert you of any errors encountered during the registration process.

 
 Publish json service with horizon/service.definition.json
 
 
 Publish deployment policy with *constraint*
 
 ```
 "constraints": [
      "state == configured"
    ],
 ```
 
 ![Policy Example ](docs/edge-monitoring.png)

2. After the agreement is made, list the docker container edge service that has been started as a result:

``` bash
sudo docker ps

CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS              PORTS                  NAMES
fdf7d0260303        iportilla/jexporter_amd64   "/bin/json_exporter …"   13 days ago         Up 3 minutes                               8060a586134d59c1e4e53d5eac1142475b46bd4a3e1afa675da6689ae0f8749d-json.exporter
```

3. See the Monitoring service output:

``` bash
curl localhost:7979/eventlog
```
 - **Note**: Press **Ctrl C** to stop the command output.

