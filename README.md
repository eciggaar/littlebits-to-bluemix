# How to connect your littleBits to Node-RED in IBM Bluemix
In this tutorial is described how you can integrate your littleBits with the Node-RED environment in IBM Bluemix. It discusses the minimum that is needed to complete this integration. Attached is a sample flow that was used to create the tweeting and singing Bluemix banana.

## What do you need?
###### At least
* An IBM Bluemix account. If you don’t have one, please register for a free account at http://ibm.biz/bluemixnl. Click 'SIGN UP' to create your free account.

* <img align="middle" src="./images/p3.png" height="60">
A littleBits USB input bit ( _p3 usb power_ ) + USB cable + power

* <img align="middle" src="./images/w20.png" height="60"> A littleBits Cloudbit connector bit ( _w20 cloud_ )

* <img align="middle" src="./images/o9.png" height="60"> A littleBits output bit ( _e.g. o9 bargraph_ )

###### Extra bits for the Banana sample flow
<p/>
* <img align="middle" src="./images/w14.png" height="60"> A littleBits Makey Makey input bit ( _w14 makey makey_ ) + connectors

* <img align="middle" src="./images/i25.png" height="60"> A littleBits mp3 player ( _i25 mp3 player_ )

* <img align="middle" src="./images/o24.png" height="60"> A littleBits Synth speaker output bit ( _o24 synth speaker_ )

## Connect your Cloudbit to the littleBits cloud
The first step is to connect your Cloudbit to the littleBits cloud, so that the Cloudbit is known to this cloud and can start sending and receiving data.

1. Go to http://control.littlebitscloud.cc/ and sign in with your userid and password. First time visitors need to register first.

2. Once you’re logged in, click on ‘+ New Cloudbit’ and assign a name to your Cloudbit. Click ‘Save’ to complete this step and to start the setup of your Cloudbit.

3. Follow Step 1 - 5 on the screen to complete the setup of your Cloudbit. If everything went fine, the Cloudbit’s status led should be green. This means the bit is connected to the littleBits cloud and is ready to send and receive data.

## Create a Node-RED environment in IBM Bluemix
The next step is to set up a Node-RED environment in IBM Bluemix. For this, click on the 'Deploy to Bluemix' button and follow the instructions.

[![Deploy to Bluemix](https://bluemix.net/deploy/button.png)](https://bluemix.net/deploy?repository=https://hub.jazz.net/git/eciggaar/bluemixbanana)

A Node-RED environment will be created for you in IBM Bluemix and the tweeting and singing banana sample flow will be deployed for you. Once the application successfully deployed, click on the 'VIEW YOUR APP' button to see your app. A new tab will be opened for this. On this page, click the 'Go to your Node-RED editor' button to open the Node-RED editor.

## Create your Node-RED flow
Now that both the Cloudbit and the Node-RED environment have been set up and configured, it is time to create your flow. To allow this flow to send data to and receive data from your littleBits Cloudbit, follow the steps described in the two paragraphs below.

###### Receiving data from the Cloudbit
One possible way of receiving data from the Cloudbit is to start your flow with a HTTP Input node. This node needs to be able to receive events that are sent by the Cloudbit. For this, you need to add your node as a subscriber to the Cloudbit. This is done via the following REST API call:
```
curl -X POST -H "Authorization: Bearer <your_access_token>" -H "Content-Type: application/json" -d '{"publisher_id": "<deviceid_cloudbit>","subscriber_id": "<subscriber_uri>"}' 'https://api-http.littlebitscloud.cc/v2/subscriptions'
```
where `<deviceid_cloudbit>` is the unique identifier of your Cloudbit and `<subscriber_id>` the URL of the subscribing HTTP input node. The `<access_token>` is your simple authentication token. Every REST call must have an Authorization header containing this token. It is listed under the ‘Settings’ of your registered Cloudbit on the littleBits Cloud Control site.

If the call was successful, you will get a response similar to:
```
{
  "publisher_id": "<deviceid_cloudbit>",
  "subscriber_id": "<subscriber_uri>",
  "publisher_events": [
    {
      "name": "amplitude:delta:ignite"
    }
  ]
}
```
To obtain a listing of subscribers to your Cloudbit, execute the following REST call:
```
curl -X GET -H "Authorization: Bearer <access_token>" 'https://api-http.littlebitscloud.cc/v2/subscriptions?publisher_id=<deviceid_cloudbit>'
```
Finally, the following call will remove your subscription from the Cloudbit. Only execute this call when your no longer wish to send your events to the Node-RED input node.
```
curl -X DELETE -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" -d '{"publisher_id": "<deviceid_cloudbit>","subscriber_id": "<subscriber_uri>"}' 'https://api-http.littlebitscloud.cc/v2/subscriptions'
```

###### Sending data to the Cloudbit.
Now that we are able to receive events from our Cloudbit, let’s see how we can trigger the Cloudbit by sending a response. The REST call to send a message to the Cloudbit is as follows:
```
curl -X POST -H "Authorization: Bearer <access_token>" -H "Content-Type: application/json" -d '{"percent": "90","duration_ms": "5000"}' 'https://api-http.littlebitscloud.cc/v2/devices/<device_id>/output'
```
Important is to specify both the ‘Authorization’ and ‘Content-Type’ header. The body is just an example payload. Both values are optional. If they are not specified, 100% is used as the default value for the percentage and 3000 (ms) as the default value for the duration.

To translate the above REST call in Node-RED nodes, we need an HTTP output node and a function node. They need to be linked with the HTTP input node as illustrated below.
![Node-RED flow](./images/flow.png)
The HTTP output node contains the URL and the request method. In this case the URL would be:
```
http.littlebitscloud.cc/v2/devices/<device_id>/output
```
and the method is set to POST. Replace `<device_id>` with the actual value of your Cloudbit.
The function node is used to prepare the message body and the request headers. The implementation of the node is as follows.
```
// First store percent value of received message
percentage = msg.payload.payload.percent;
msg={}; // Empty message for HTTP call to Cloudbit

node.log('percentage:' + percentage);

// Set headers for HTTP call to Cloudbit
msg.headers = {
    "Authorization" : "Bearer <your_access_token>",
    "Content-type" : "application/json"
};

// Set message body for HTTP call to Cloudbit
msg.payload = {
  "percent": percentage,
  "duration_ms": "5000"
}

return msg;
```
## The tweeting and singing Banana sample flow
The Node-RED flow that is deployed as sample flow illustrates the integration described above. It receives events sent by the Cloudbit. Next, using an iterator — which is reset every day — it tweets how many times it has been used that day. Finally, it invokes the Cloudbit to trigger the next step in the littleBits set-up. In our example the Cloudbit triggered a bargraph bit (O9) for 5 seconds, followed by playing a song from the MP3 player bit (i25). Because the initial Cloudbit event was triggered by a Makey Makey bit (w14), connected to a banana, this makes our banana a tweeting and singing banana powered by littleBits and IBM Bluemix.
<p align="center">
<img src="./images/bluemixbanana.jpg" height="300">
</p>
Make sure that you edit the nodes to use your own values for the Cloudbit and twitter account.

Have fun and **happy** coding!!
