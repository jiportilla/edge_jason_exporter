# JSON Exporter Service

This is an example of creating and using an edge exporter service which scrapes JSON by JSONPath for [prometheus](https://prometheus.io/) monitoring

Based on [https://github.com/prometheus-community/json_exporter](https://github.com/prometheus-community/json_exporter)

- [Preconditions for Using the JSON Exporter Service](#preconditions)
- [Configuring the JSON Exporter Service](#configuring)
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

If you have not done so already, you must do these steps before proceeding with the JSON Exporter service:

1. Update `config.yaml`
2. 


## <a id=using-JSON-exporter></a> Using the JSON Exporter Service with Deployment Policy

1. Register your edge node with Horizon to use the JSON exporter edge service:

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

