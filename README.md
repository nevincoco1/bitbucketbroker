# Connecting Snyk UI with Bitbucket Server requires setting up a broker client. 

When setting up a Snyk broker, the client access is only enabled for Open Source manifest files and dockerfile access. However, to integrate the repositories with Snyk Code and IAC, the Snyk code agent is needed as well.

Note: The broker client and code agent MUST be present in the same network.

## Steps to set-up the broker client and code agent. 
In this example we will set up broker + code agent with Bitbucket Server. 

Before getting started with the following setup ensure that:
  - The broker support for the specific integration has been enabled by your Snyk point of contact or you have created it using API
  - Your Snyk Point of contact has enabled ``code-agent`` support. 


**Step 1** - Find your Bitbucket Server broker token in the Snyk UI under Integration Settings as seen below. 

![image](https://user-images.githubusercontent.com/89480245/155851373-a44c9962-a6d9-4bda-8b86-f72c1447880c.png)

You can view it by clicking **Show token**. 

![image](https://user-images.githubusercontent.com/89480245/155851584-87a8b36d-cae7-4d77-9ef6-d0313b8916c7.png)

Once you have the token please proceed with the following steps on your Docker server.

**Step 2** - Pull the appropriate broker client image. Details for individual image can be found here: https://github.com/snyk/broker

`` docker pull snyk/broker:bitbucket-server ``

**Step 3** - Pull the code-agent image. 

`` docker pull snyk/code-agent ``

**Step 4** - Create a network (this network will be run both code agent and broker client). 
Note: You can provide any name to the network - here I am calling it `` mySnykBrokerNetwork `` 

`` docker network create mySnykBrokerNetwork ``

You can confirm that it was created by running ``docker network ls``, this will show results like this:
```
NETWORK ID     NAME                 DRIVER     SCOPE
d1353a2b0f66   mySnykBrokerNetwork  bridge     local
```
**Step 5** - Put the ``accept.json`` file (at the bottom of this document) in a local directory on the Docker server (**/home/private** for our example) so that you can map it to the container. The included file contains both IAC and Code and can be edited to omit files from scanning.

**Step 6** - Run the broker container. The following environment variables are mandatory to configure the Broker client:

- **-p <*port:port*>** - port on Docker server and port it will map to on broker container
- **BROKER_TOKEN** - the snyk broker token, obtained from your Bitbucket Server integration settings view (From Step 1).
- **BITBUCKET_USERNAME** - the Bitbucket Server username.
- **BITBUCKET_PASSWORD** - the Bitbucket Server password.
- **BITBUCKET** - the hostname or IP address of your Bitbucket Server with port number, such as:
  - **bitbucket-server.domain.com:7990** or 
  - **10.10.1.100:7990**.
- **BITBUCKET_API** - the API endpoint of your Bitbucket Server deployment. Should be <*Value of BITBUCKET*>/rest/api/1.0. such as:
  - **bitbucket-server.domain.com:7990/rest/api/1.0** or
  - **10.10.1.100:7990/rest/api/1.0**
- **BROKER_CLIENT_URL** - the full URL of the Broker client server as it will be accessible by your Bitbucket Server for webhooks, such as:
  - **http://my.broker.client:7341** or
  - **http://10.10.1.99:7341**
- **PORT** - the local port at which the Broker client accepts connections. Default is 7341.

If you are using Code Agent:
- **GIT_CLIENT_URL** - URL pointing to the Docker container running the code agent such as:
  - **http://code-agent:3000**
- **--network** - to associate the client with a specific Docker network 

Syntax:
````
docker run --restart=always \
           -p 8000:8000 \
           -e BROKER_TOKEN=secret-broker-token \
           -e BITBUCKET_USERNAME=username \
           -e BITBUCKET_PASSWORD=password \
           -e BITBUCKET=bitbucket-server.domain.com:<port> \
           -e BITBUCKET_API=bitbucket-server.domain.com:<port>/rest/api/1.0 \
           -e BROKER_CLIENT_URL=http://my.broker.client:8000 \
           -e PORT=8000 \
           -e ACCEPT=/private/accept.json
           -v /local/path/to/private:/private \
           -e GIT_CLIENT_URL=http://code-agent:3000 \
           --network mySnykBrokerNetwork \
       snyk/broker:bitbucket-server
       
````
Add this line to the -e section if your Bitbucket server has a self-signed certificate and you want to ignore the error:
```
           -e NODE_TLS_REJECT_UNAUTHORIZED=0
```           
**Or**

#### Setup Git with an internal certificate ####
By default, the Broker client establishes HTTPS connections to the Git. If your Git is serving an internal certificate (signed by your own CA), you can provide the CA certificate to the Broker client.

For example, if your CA certificate is at **./private/ca.cert.pem**, provide it to the docker container by mounting the folder and using the **CA_CERT** environment variable:

```
           -e CA_CERT=/private/ca.cert.pem \
           -v /local/path/to/private:/private \
```
### If you are deploying the code agent ###
**Step 7** - Run the code-agent
````
 docker run --name code-agent \
    -p 3000:3000 \
    -e PORT=3000 -e SNYK_TOKEN=<token> --network mySnykBrokerNetwork \
     snyk/code-agent 
````

**Step 8** - At this point your logs for ``broker client`` should display ``"msg":"successfully established a websocket connection to the broker server"`` and in app.snyk.io under the integration you would see "Connected". Also, see the logs of the ``code-agent`` to confirm its working. 

8. You can now seemlessly import your repos on the UI. 
## Troubleshooting ##
Read the container logs (of the Broker and Code Agent) 
### Methodology: ### 
- Go to the Snyk UI and attempt to connect - this will generate a log entry
- Run docker logs <CONTAINER_ID> 
- Go to the latest log entry and review the results
- To troubleshoot code agent - import a single repo. Look at the import logs in the Snyk UI - If you get bundle creation failed then look at the logs.
- If using the Code Agent, ensure that the request is making it from the Broker to the Code Agent. You can do this by searching the SnykGit string.
- To make sure both containers are running on the same Docker network, use the command: ``docker network inspect mySnykBrokerNetwork``
