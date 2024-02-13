# on-premise-deploy
Code and artefacts required to deploy PM4ML in an on-premise environment

## Running payment manager with testing toolkit included

![Payment manager with TTK](/assets/images/ttk-scenario1.png)

### What is Testing Toolkit?

Testing toolkit (TTK) is a tool for simulating and validating mojaloop implementations. We are using TTK in this scenario to simulate a mojaloop switch combined with payee fsp.

### Prerequisites

You need to have the following softwares on your machine
* git
* docker
* docker-compose

Please run the following commands to start the payment manager on your machine
```
git clone https://github.com/pm4ml/on-premise-deploy.git
cd on-premise-deploy/docker-compose
docker-compose up
```

Now a payment manager should be running on your machine with TTK.

### TTK UI

Open the URL http://localhost:6060 for TTK UI
Go to `Monitoring` page.

### Portal UI

You can see all the transfers executed in the payment manager portal.

Open the URL http://localhost:8081 for payment manager UI
Login with the username `test` and password `test`

### Keycloak UI

You can manage the users using the keycloak admin interface on http://localhost:8080
Login with username `admin` and password `admin`

---

## Sending transfer from core connector simulator acting as payer

### Initiate a transfer from command line

Execute the following curl command to send a http request to the simulator core connector for initiating a transfer

```
curl 'http://localhost:3003/sendmoney' -H 'content-type: application/json;charset=utf-8' --data-binary '{"from":{"displayName":"PayerFirst PayerLast","idType":"MSISDN","idValue":"22507008181"},"to":{"idType":"MSISDN","idValue":"22556999125"},"amountType":"SEND","currency":"USD","amount":"100","transactionType":"TRANSFER","note":"test payment","homeTransactionId":"12277380-9d94-4a8a-b49e-0fac3b8b3565"}'
```

Please observe the transfer state 'COMMITTED' indicating a successful transfer.

### Observe the http requests and callbacks in TTK

In the `Monitoring` page opened before, you can see the logs.

### Initiate a transfer from TTK

* Open the menu item `Test Runner` in TTK UI in a new tab.
* Click on `Collection Manager` button and import the file 'testing-toolkit/collections/payer-tests/sendmoney_auto_acceptance.json'
* Click on `Send` button
* You should see the test run successful
* Click on the edit button against the test case and observe the request sent and response received

You should see the logs about the transaction in the Monitoring page already opened in another tab.

---

## Sending transfer to core connector simulator acting as payee

* Open the menu item `Test Runner` in TTK UI.
* Click on `Collection Manager` button and import the folder 'testing-toolkit/collections/payee-tests'
* In `Collection Manager`, select the file `p2p_happy_path.json` and close the collection manager
* Click on `Send` button and see the result
* Click on the edit button against the test case and observe the request sent and response received

---

## Developing your own core connector

You can use this repository to help with the core connector development.
Developers can follow the following guidelines to run dependent services and simulators locally on their machines.

- Disable the services `simulator-core-connector` and `sim-backend` in docker-compose.yaml file

### Testing outbound transfer from core-connector

- Start the services using `docker-compose up` by following the first section in this document
- Open the TTK monitoring page on `http://localhost:6060/admin/monitoring`
- Try to send the following request from your core connector to the sdk-scheme-adapter service
```
curl 'http://localhost:4001/transfers' -H 'content-type: application/json;charset=utf-8'
  --data-binary '{"homeTransactionId":"abc123","from":{"idType":"MSISDN","idValue":"22507008181"},"to":{"idType":"MSISDN","idValue":"22556999125"},"amountType":"SEND","currency":"USD","amount":"10","transactionType":"TRANSFER","note":"string"}'
```
- You should get the response with transfer state as 'COMMITTED' and you should also able to see the request in TTK monitoring page

### Testing 3-phase outbound transfer from core-connector

In the above section the transfer is successful with a single HTTP call because the parameters `AUTO_ACCEPT_PARTY` and `AUTO_ACCEPT_QUOTES` are set to true in sdk-scheme-adapter section of `docker-compose.yaml` file.
If we want to give an option to the end user (Sender) to approve the party and then approve the quote, then we need to set these parameters to false and make 3 consecutive http calls to make the transfer successful. Please follow the following steps for this scenario.
- Set the parameters `AUTO_ACCEPT_PARTY` and `AUTO_ACCEPT_QUOTES` to false
- Restart the docker-compose with the commands `docker-compose down` and `docker-compose up`
- Send the above `POST /transfers` request and observe the response
- This time it only contains the party information and you don't see the transferState in the response
- Note down the transferId from the response
- Then to accept party, the following HTTP call should be made
```
 curl -X PUT 'http://localhost:4001/transfers/TRANSFERID_HERE' -H 'content-type: application/json;charset=utf-8' --data-binary '{"acceptParty": true}'
```
- Now you will get the quote information in the response
- To accept quote, the following HTTP call should be made
```
 curl -X PUT 'http://localhost:4001/transfers/TRANSFERID_HERE' -H 'content-type: application/json;charset=utf-8' --data-binary '{"acceptQuote": true}'
```
- You should finally see the transferState as `COMMITTED` in the response


### Testing inbound transfer to core-connector

- For inbound transfer, you need to configure `sdk-scheme-adapter` to point to your core connector hostname and IP to send HTTP calls
- Change the value of `BACKEND_ENDPOINT` in `sdk-scheme-adapter` section of docker-compose.yaml to the host and port at which you are running the core connector
  - Please note this host and port should be reachable from the docker container
  - Typically it should be the IP of the docker interface on your machine (Ex: 172.17.0.1)
  - For MacOS, you may use these host names - `docker.for.mac.host.internal` or `docker.for.mac.localhost`
  - For Windows, you may use the host name - `host.docker.internal`
  - If the above hostnames do not work for you, you need to find out the IP of docker interface (Ex: ``)
- Start the services using `docker-compose up` by following the first section in this document
- Send a transfer from TTK by following the section `Sending transfer to core connector simulator acting as payee`
- The testcase should be successful and you should get an inbound HTTP request on your core connector implementation.
