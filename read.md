**EDGE2AI Workshop**

Introduction
============

In this workshop, you will build a full OT to IT workflow for an **IoT
Predictive Maintenance** use case. Below is the architecture diagram,
showing all the components you will setup over the next 8 lab exercises.
While the diagram divides the components according to their location
(factory, regional or datacenter level) in this workshop all such
components will reside in one single host.

![图片包含 地图, 文字
描述已自动生成](media/image1.jpeg){width="5.768055555555556in"
height="2.4569444444444444in"}

Troubleshooting
===============

-   Everything is **Case-Sensitive**.

MiNiFi Not Sending Messages
---------------------------

-   Make sure you pick S2S not RAW in Cloud Connection to NiFi

-   Make sure No Spaces Before or After Destination ID, URL, Names,
    Topics, Brokers, Etc...​

-   Make sure No Spaces Anywhere

-   Everything is Case-Sensitive. It's IoT, not iot.

-   Use User=Admin for CDSW and HUE

-   You must have HDFS User Created via HUE, Go there First

-   Check all your connections and spellings

-   Check **/opt/cloudera/cem/minifi/logs/minifi-app.log** if you can't
    find an issue

CEM doesn't pick up new NARs
----------------------------

1.  Delete the agent manifest manually using the EFM API:

2.  Verify each class has the same agent manifest ID:

> http://hostname:10080/efm/api/agent-classes
>
> \[{\"name\":\"iot1\",\"agentManifests\":\[\"agent-manifest-id\"\]},{\"name\":\"iot4\",\"agentManifests\":\[\"agent-manifest-id\"\]}\]

3.  Confirm the manifest doesn't have the NAR you installed

> http://hostname:10080/efm/api/agent-manifests?class=iot4
>
> \[{\"identifier\":\"agent-manifest-id\",\"agentType\":\"minifi-java\",\"version\":\"1\",\"buildInfo\":{\"timestamp\":1556628651811,\"compiler\":\"JDK
> 8\"},\"bundles\":\[{\"group\":\"default\",\"artifact\":\"system\",\"version\":\"unversioned\",\"componentManifest\":{\"controllerServices\":\[\],\"processors\":

4.  Call the API endpoint:

> http://hostname:10080/efm/swagger/

5.  Hit the DELETE - Delete the agent manifest specified by id button,
    and in the id field, enter agent-manifest-id

Labs summary
============

-   **Lab 0 -** Setup the sandbox environment and Connecting to your
    cluster.

-   **Lab 1 -** Configuring the NiFi flow and Consuming the Meetup RSVP
    data

-   **Lab 2 -** Using NiFi Registry to Manage SFDC

-   **Lab 3 -** Apache NiFi: setup machine sensors simulator

-   **Lab 4 -** Configuring Edge Flow Management

-   **Lab 5 -** Registering our schema in Schema Registry

-   **Lab 6 -** Configuring the NiFi flow and pushing data to Kafka

-   **Lab 7 -** Use SMM to confirm that the data is flowing correctly

-   **Lab 8 -** Update the edge flows to perform additional processing
    on the data

-   **Lab 9 -** Using Hue create Kudu table.

-   **Lab 10 -** CDSW: Train the model

-   **Lab 11 -** CDSW: Deploy the model

-   **Lab 12 -** Use NiFi to call the CDSW model endpoint and save to
    Kudu

-   **Lab 13 -** Use Spark to call a CDSW model endpoint and save to
    Kudu

-   **Lab 14 -** Fast analytics on fast data with Kudu and Impala

-   **Lab 15** - Use SRM to simplify Kafka Replication

Pre-requisites
==============

-   Laptop with a supported OS (Windows 7 not supported).

-   A modern browser like Google Chrome or Firefox (IE not supported).

Lab 0 - Setup the sandbox environment and Connecting to your cluster.
=====================================================================

Create a CDH+CDSW cluster or a CDP+CDSW cluster and PLEASE NOTE that due
to a minor MiNiFi bug, I installed
[[mosquitto]{.underline}](https://mosquitto.org/) in setup.sh. If you
have something wrong with MiNiFi, You will be prompted to explicitly
start MiNiFi. Check the Troubleshooting at the end of this document for
how to reset MiNiFi in case you forgot to do this step.

You should have 2 addresses for you one-node cluster: the public DNS
name and the public IP address. With those addresses you can test the
following connectivity to your cluster:

1)  Ensure you can SSH into the cluster (using either the DNS name or IP
    address)

2)  Ensure you can connect to the following service using your browser:

  Service             URL   Credentials
  ------------------- ----- -------------
  Cloudera Manager          admin/admin
  Edge Flow Manager         
  NiFi                      
  NiFi Registry             
  Schema Registry           
  SMM                       
  Hue                       
  CDSW                      

3)  Login into **Cloudera Manager** and familiarize yourself with the
    services installed

4)  Login into **Hue**. As you are the first user to login into Hue, you
    are granted admin privileges. At this point, you won't need to do
    anything on Hue, but by logging in, CDH has created your HDFS user
    and folder, which you will need for the next lab.

5)  Login into **cdsw**. As you are the first user to login into cdsw,
    you need to create a new user.

Below a screenshot of Chrome open with 8 tabs, one for each service.

![browser](media/image2.png){width="5.768055555555556in"
height="4.247222222222222in"}

Download the resource from github:

git clone
[[https://github.com/wangxf2000/edge2ai-workshop.git]{.underline}](https://github.com/wangxf2000/edge2ai-workshop.git)

mkdir -p /tmp/demo/

cp edge2ai-workshop/simulate.py /tmp/demo/

chown -R nifi:nifi /tmp/demo/

chmod +x /tmp/demo/simulate.py

Lab 1 - Configuring the NiFi flow and Consuming the Meetup RSVP data
====================================================================

In order to have a streaming source available for our workshop, we are
going to make use of the publicly available Meetup's API and connect to
their WebSocket.

The API documentation is available
\[[[here]{.underline}](https://www.meetup.com/meetup_api/docs/stream/2/event_comments/#websockets)\]: [[https://www.meetup.com/meetup\_api/docs/stream/2/event\_comments/\#websockets]{.underline}](https://www.meetup.com/meetup_api/docs/stream/2/event_comments/#websockets)

In this workshop we are going to stream all comments, for all topics,
into NiFi and classify each one of them into the 5 categories listed
below:

-   very negative

-   negative

-   neutral

-   positive

-   very positive

To do so we will score each comment against the Stanford CoreNLP's
sentiment model as we will see
\[[[later]{.underline}](https://github.com/wangxf2000/edge2ai-workshop#run-the-sentiment-analysis-model-as-a-rest-like-service)\]

In real-world use case we would probably filter by event of our interest
but for the sake of this workshop we won't and assume all comments are
given for the same event: the famous CDF workshop!

Let's get started...​ Open \[NiFi
UI\]([[http://\<public\_dns\>:8080/nifi/]{.underline}]()) and follow the
steps below:

1)  Drag on drop a Process Group on the root canvas and name it **CDF
    Workshop** 

![](media/image3.jpeg){width="5.768055555555556in"
height="3.642361111111111in"}

![图片包含 屏幕截图
描述已自动生成](media/image4.jpeg){width="5.768055555555556in"
height="3.6034722222222224in"}

2)  Get in the CDF Workshop (double click on the process group) and add
    a **ConnectWebSocket** processor to the canvas

![图片包含 屏幕截图
描述已自动生成](media/image5.jpeg){width="5.768055555555556in"
height="3.6708333333333334in"}

![图片包含 屏幕截图
描述已自动生成](media/image6.jpeg){width="5.768055555555556in"
height="3.6631944444444446in"}

-   Double click on the processor

-   On settings tab, check all relationships except **text message**

![图片包含 屏幕截图
描述已自动生成](media/image7.jpeg){width="5.768055555555556in"
height="3.6590277777777778in"}

-   Got to properties tab and select or
    create **JettyWebSocketClient** as the WebSocket Client
    ControllerService

![图片包含 屏幕截图
描述已自动生成](media/image8.jpeg){width="5.768055555555556in"
height="3.661111111111111in"}

-   Then configure the service (click on the arrow on the right)

-   Go to properties tab and add this
    value: ws://stream.meetup.com/2/event\_comments to
    property **WebSocket URI**

![图片包含 屏幕截图
描述已自动生成](media/image9.jpeg){width="5.768055555555556in"
height="3.6659722222222224in"}

-   Apply the change

![图片包含 屏幕截图
描述已自动生成](media/image10.jpeg){width="5.768055555555556in"
height="3.65625in"}

-   Enable the controller service (click on the thunder icon) and close
    the window

![图片包含 屏幕截图
描述已自动生成](media/image11.jpeg){width="5.768055555555556in"
height="3.6527777777777777in"}

-   Go to properties tab and give a value to **WebSocket Client
    Id** such as **demo** for example

![图片包含 屏幕截图
描述已自动生成](media/image12.jpeg){width="5.768055555555556in"
height="3.6458333333333335in"}

-   Apply changes

3)  Step 3: Add an **UpdateAttribute** connector to the canvas and link
    from **ConnectWebSocket** on **text message** relationship

-   Double click on the processor

-   On properties tab add new property **mime.type** clicking on + icon
    and give the value **application/json**. This will tell the next
    processor that the messages sent by the Meetup WebSocket is in JSON
    format.

![图片包含 屏幕截图
描述已自动生成](media/image13.jpeg){width="5.768055555555556in"
height="3.6555555555555554in"}

-   Add another property **event** to set an event name **CDF
    workshop** for the purpose of this exercise as explained before

-   Apply changes

![图片包含 屏幕截图
描述已自动生成](media/image14.jpeg){width="5.768055555555556in"
height="3.654861111111111in"}

4)  Step 4: Add **EvaluateJsonPath** to the canvas and link from
    **UpdateAttribute** on **success** relationship

-   Double click on the processor

-   On settings tab, check
    both **failure** and **unmatched** relationships

-   On properties tab, change **Destination** value
    to **flowfile-attribute**

-   And add properties as follow

comment: \$.comment

member: \$.member.member\_name

timestamp: \$.mtime

country: \$.group.country

![图片包含 屏幕截图
描述已自动生成](media/image15.jpeg){width="5.768055555555556in"
height="3.6805555555555554in"}

The messages coming out of the web sockets look like this:

json
{\"visibility\":\"public\",\"member\":{\"member\_id\":11643711,\"photo\":\"https:\\/\\/secure.meetupstatic.com\\/photos\\/member\\/3\\/1\\/6\\/8\\/thumb\_273072648.jpeg\",\"member\_name\":\"Loka
Murphy\"},\"comment\":\"I didn't when I registered but now thinking I
want to try and get one since it's only taking place
once.\",\"id\":-259414201,\"mtime\":1541557753087,\"event\":{\"event\_name\":\"Tunnel
to Viaduct 8k
Run\",\"event\_id\":\"256109695\"},\"table\_name\":\"event\_comment\",\"group\":{\"join\_mode\":\"open\",\"country\":\"us\",\"city\":\"Seattle\",\"name\":\"Seattle
Green Lake Running
Group\",\"group\_lon\":-122.34,\"id\":1608555,\"state\":\"WA\",\"urlname\":\"Seattle-Greenlake-Running-Group\",\"category\":{\"name\":\"fitness\",\"id\":9,\"shortname\":\"fitness\"},\"group\_photo\":{\"highres\_link\":\"https:\\/\\/secure.meetupstatic.com\\/photos\\/event\\/9\\/e\\/f\\/4\\/highres\_465640692.jpeg\",\"photo\_link\":\"https:\\/\\/secure.meetupstatic.com\\/photos\\/event\\/9\\/e\\/f\\/4\\/600\_465640692.jpeg\",\"photo\_id\":465640692,\"thumb\_link\":\"https:\\/\\/secure.meetupstatic.com\\/photos\\/event\\/9\\/e\\/f\\/4\\/thumb\_465640692.jpeg\"},\"group\_lat\":47.61},\"in\_reply\_to\":496130460,\"status\":\"active\"}

5)  Step 5: Add an **AttributesToCSV** processor to the canvas and link
    from **EvaluateJsonPath** on **matched** relationship

-   Double click on the processor

-   On settings tab, check **failure** relationship

-   Change **Destination** value to **flowfile-content**

-   Change **Attribute List** value to write only the above parsed
    attributes: **timestamp,event,member,comment,country**

-   Set Include Core Attributes to **false**

-   Set **Include Schema** to **true**

-   Apply changes

![图片包含 屏幕截图
描述已自动生成](media/image16.jpeg){width="5.768055555555556in"
height="3.640277777777778in"}

6)  Step 6: Add a **PutFile** processor to the canvas and link from
    **AttributesToCSV** on **success** relationship

-   Double click on the processor

-   On settings tab, check all relationships

-   Change **Directory** value to **/tmp/workshop**

-   Apply changes

![图片包含 屏幕截图
描述已自动生成](media/image16.jpeg){width="5.768055555555556in"
height="3.640277777777778in"}

7)  Step 7: Start the entire flow

![图片包含 屏幕截图
描述已自动生成](media/image17.jpeg){width="5.768055555555556in"
height="3.582638888888889in"}

SSH to the sandbox and explore the files created under /tmp/workshop.

On the NiFi UI, explore the FlowFiles\' attributes and content looking
at Data provenance.

**Once done, stop the flow and delete all files sudo rm -rf
/tmp/workshop/\***

Lab 2 - Using NiFi Registry to Manage SFDC
==========================================

> Step 1：We want to be able to version control the flows we will add to
> the Process Group. In order to do that, we first need to connect NiFi
> to the **NiFi Registry**. On the NiFi global menu, click on
> \"Controller Services\", navigate to the \"Registry Clients\" tab and
> add a Registry client with the following URL( The URL use your local
> URL):
>
> Name: NiFi Registry
>
> URL:
> [[http://edge2ai-1.dim.local:18080]{.underline}](http://edge2ai-1.dim.local:18080)

![图片包含 屏幕截图, 室内
描述已自动生成](media/image18.png){width="5.768055555555556in"
height="3.1125in"}

![图片包含 屏幕截图
描述已自动生成](media/image19.png){width="5.768055555555556in"
height="3.5479166666666666in"}

Step 2: Go to [[NiFi
Registry]{.underline}](http://demo.cloudera.com:61080/nifi-registry/explorer/grid-list) and
create a new bucket

-   Click on the little wrench icon **Settings** at the top right corner

-   Click on the **NEW BUCKET** button

-   Name the bucket **workshop**

![图片包含 屏幕截图
描述已自动生成](media/image20.png){width="5.768055555555556in"
height="3.5791666666666666in"}

Step 3: Go back to NiFi UI and right click on the previously created
process group

-   Click on Version \> Start version control

-   Then select the Bucket name as **workshop**, created before

-   provide at least a Flow Name

-   Click on Save

![图片包含 屏幕截图
描述已自动生成](media/image21.png){width="5.768055555555556in"
height="3.5944444444444446in"}

![图片包含 屏幕截图
描述已自动生成](media/image22.png){width="5.768055555555556in"
height="3.5819444444444444in"}

Step 4: Go back to NiFi Registry and check the version info.

-   Click NiFi Registry to ALL bucket

-   Click the bucket Workshop to check the version info.

![图片包含 屏幕截图
描述已自动生成](media/image23.png){width="5.768055555555556in"
height="3.5701388888888888in"}

Step 5: Go back to NiFi UI and modify something, then apply the Version
control and back to NiFi Registry to check the changes.

-   Modify something, like change the properties or change the
    positions.

-   Back to the root canvas,Right click processorgroup and then select
    the Version-\> Commit local changes

-   You can add some comments then save Flow Version

-   Back to the NiFi Registry to check the version info.

![图片包含 屏幕截图
描述已自动生成](media/image24.png){width="5.768055555555556in"
height="3.5868055555555554in"}

![图片包含 屏幕截图
描述已自动生成](media/image25.png){width="5.768055555555556in"
height="3.575in"}

![图片包含 屏幕截图
描述已自动生成](media/image26.png){width="5.768055555555556in"
height="3.5840277777777776in"}

Lab 3 - Apache NiFi: setup machine sensors simulator
====================================================

In this lab you will run a simple Python script that simulates IoT
sensor data from some hypothetical machines, and send the data to a MQTT
broker ([[mosquitto]{.underline}](https://mosquitto.org/)). The gateway
host is connected to many and different type of sensors, but they
generally all share the same transport protocol, \"mqtt\". You will go
to Apache NiFi and add a Processor (**ExecuteProcess**) to the canvas.
You will then right-click and set the properties shown below to run our
Python simulate script.

**STEP 1** : Create a **ExecuteProcess** Processor in Nifi

**Command**: python

**Command Arguments: **/opt/demo/simulate.py

![图片包含 屏幕截图
描述已自动生成](media/image27.jpeg){width="5.721738845144357in"
height="2.6059405074365705in"}

![图片包含 屏幕截图
描述已自动生成](media/image28.png){width="5.569444444444445in"
height="3.9166666666666665in"}

**STEP 2** : adjust Schedule.

In the **Scheduling** Tab, set to **Run Schedule**: 1 sec

You could set that to 1 sec, 30 sec, 1 min.

![图片包含 屏幕截图
描述已自动生成](media/image29.png){width="5.541666666666667in"
height="3.875in"}

Include no extra spaces!

**STEP 3** : adjust Settings

**Tab:** Settings

**Automatically Terminate Relationships**: \[x\] Success

Make sure you terminate so you can run.

![图片包含 屏幕截图
描述已自动生成](media/image30.png){width="5.555555555555555in"
height="3.888888888888889in"}

**STEP 4** : Start the simulator runner.

You can then right click to Start this simulator runner. You can press
stop after a few seconds and look at the provenance to see that it has
run a number of times and produced results.

You can also see detail of the data provenance, like provenance event ,
detail input and output, lineage and so on.

![图片包含 屏幕截图
描述已自动生成](media/image31.png){width="5.617390638670166in"
height="4.476358267716535in"}

![图片包含 屏幕截图
描述已自动生成](media/image32.png){width="5.768055555555556in"
height="3.5243055555555554in"}

![图片包含 屏幕截图
描述已自动生成](media/image33.png){width="5.768055555555556in"
height="3.5416666666666665in"}

![图片包含 屏幕截图
描述已自动生成](media/image34.png){width="5.768055555555556in"
height="3.557638888888889in"}

Lab 4 - Configuring Edge Flow Management
========================================

Cloudera Edge Flow Management gives you a visual overview of all MiNiFi
agents in your environment, and allows you to update the flow
configuration for each one, with versioning control thanks to the **NiFi
Registry** integration. In this lab, you will create the MiNiFi flow and
publish it for the MiNiFi agent to pick it up.

Open the EFM Web UI at . If the link doesn't work, back to the OS and
start the efm services. **systemctl start efm**. Ensure you see your
minifi agent's heartbeat messages in the **Events Monitor**.

![图片包含 屏幕截图
描述已自动生成](media/image35.jpeg){width="5.768055555555556in"
height="3.563888888888889in"}

![图片包含 屏幕截图
描述已自动生成](media/image36.png){width="5.768055555555556in"
height="3.329861111111111in"}

-   You can then select the **Flow Designer** tab (![flow designer
    icon](media/image37.jpeg){width="0.4173611111111111in"
    height="0.3736111111111111in"}). To build a dataflow, select the
    desired class (iot-1) from the table and click OPEN. Alternatively,
    you can double-click on the desired class.

**STEP 1** : Add ConsumeMQTT Processor

-   Add a *ConsumeMQTT* Processor to the canvas, by dragging the
    processor icon to the canvas, selecting
    the ***ConsumeMQTT*** processor type and clicking on
    the **Add** button. Once the processor is on the canvas,
    double-click it and configure it with below settings:

> Broker URI: tcp://edge2ai-1.dim.local:1883
>
> Client ID: minifi-iot
>
> Topic Filter: iot/\#
>
> Max Queue Size: 60

![add consumer mqtt](media/image38.png){width="5.768055555555556in"
height="4.247222222222222in"}

**STEP 2** : Add Remote Process Group(RPG)

-   Add a ***Remote Process Group* **(RPG) to the canvas and configure
    it as follows:

> URL: http://edge2ai-1.dim.local:8080/nifi

![add rpg](media/image39.png){width="5.768055555555556in"
height="4.247222222222222in"}

-   At this point you need to connect the **ConsumerMQTT** processor to
    the **RPG**. For this, you first need to add an **Input Port** to
    the remote NiFi server. Open the NiFi Web UI
    at http://\<public\_dns\>:8080/nifi/ and drag the ***Input
    Port* **to the canvas. Call it something like \"from Gateway\".

![add input port](media/image40.png){width="5.768055555555556in"
height="4.247222222222222in"}

**STEP 3** : Add Funnel

-   To terminate the NiFI *Input Port* let's, for now, add
    a ***Funnel*** to the canvas...​

![add funnel](media/image41.png){width="5.768055555555556in"
height="4.238194444444445in"}

-   ...​ and setup a connection from the Input Port to it. To setup a
    connection, hover the mouse over the Input Port until an arrow
    symbol is shown in the center. Click on the arrow, drag it and drop
    it on the Funnel to connect the two elements.

![connecting](media/image42.png){width="5.768055555555556in"
height="4.247222222222222in"}

-   Right-click on the Input Port and start it. Alternatively, click on
    the Input Port to select it and then press the start (\"play\")
    button on the Operate panel:

![operate panel](media/image43.png){width="4.165277777777778in"
height="2.5652777777777778in"}

-   You will need the ID of the *Input Port* to complete the
    **connection** of the ***ConsumeMQTT*** processor to the **RPG**
    (NiFi). Double-click on the ***Input Port* **and copy its **ID**.

![input port id](media/image44.png){width="5.768055555555556in"
height="4.247222222222222in"}

**STEP 4** : Connect ComsumeMQTT to RPG

-   Back to the Flow Designer, connect the ConsumeMQTT processor to the
    RPG. The connection requires an ID and you can paste here the ID you
    copied from the Input Port.

![connect to rpg](media/image45.png){width="5.768055555555556in"
height="4.247222222222222in"}

-   The Flow is now complete, but before publishing it, create the
    Bucket in the ***NiFi Registry*** so that all versions of your flows
    are stored for review and audit. Open the NiFi Registry
    at http://\<public\_dns\>:18080/nifi-registry, click on the
    wrench/spanner icon (![spanner
    icon](media/image46.jpeg){width="0.27847222222222223in"
    height="0.27847222222222223in"}) on the top-right corner on and
    create a bucket called IoT.

![create bucket](media/image47.png){width="5.768055555555556in"
height="4.247222222222222in"}

**STEP 5** : publish the flow

-   You can now publish the flow for the MiNiFi agent to automatically
    pick up. Click **Publish**, add a descriptive comment for your
    changes and click **Apply**.

![publish flow](media/image48.png){width="5.768055555555556in"
height="4.247222222222222in"}

![图片包含 屏幕截图
描述已自动生成](media/image49.png){width="5.704348206474191in"
height="3.7860717410323708in"}

-   Go back to the **NiFi Registry** Web UI and click on the **NiFi
    Registry** name, next to the Cloudera logo. If the flow publishing
    was successful, you should see the flow's version details in the
    NiFi Registry.

![图片包含 屏幕截图
描述已自动生成](media/image50.png){width="5.768055555555556in"
height="1.8479166666666667in"}

**STEP 6** : test the edge flow.

-   At this point, you can test the edge flow up until NiFi. Start the
    NiFi (**ExecuteProcess**) simulator again and confirm you can see
    the messages queued in NiFi.

![图片包含 屏幕截图
描述已自动生成](media/image51.png){width="5.152777777777778in"
height="2.263888888888889in"}

-   You can stop the simulator (Stop the NiFi processor) once you
    confirm that the flow is working correctly.

Lab 5 - Registering our schema in Schema Registry
=================================================

The data produced by the temperature sensors is described by the schema
in
file [[sensor.avsc]{.underline}](https://raw.githubusercontent.com/tspannhw/edge2ai-workshop/master/sensor.avsc).
In this lab we will register this schema in Schema Registry so that our
flows in NiFi can refer to schema using an unified service. This will
also allow us to evolve the schema in the future, if needed, keeping
older versions under version control, so that existing flows and
flowfiles will continue to work.

**STEP 1** : copy schema to **Schema Registry**

-   Go the following URL, which contains the schema definition we'll use
    for this lab. Select all contents of the page and copy it.

> [[https://github.com/wangxf2000/edge2ai-workshop/blob/master/sensor.avsc]{.underline}](https://github.com/wangxf2000/edge2ai-workshop/blob/master/sensor.avsc)

-   In the Schema Registry Web UI, click the + sign to register a new
    schema.

-   Click on a blank area in the **Schema Text** field and paste the
    contents you copied.

-   Complete the schema creation by filling the following properties:

Name: SensorReading

Description: Schema for the data generated by the IoT sensors

Type: Avro schema provider

Schema Group: Kafka

Compatibility: Backward

Evolve: checked

![图片包含 屏幕截图, 监视器, 室内, 墙壁
描述已自动生成](media/image52.png){width="5.768055555555556in"
height="4.246527777777778in"}

-   Save the schema

![图片包含 屏幕截图
描述已自动生成](media/image53.png){width="5.768055555555556in"
height="2.1819444444444445in"}

Lab 6 - Configuring the NiFi flow and pushing data to Kafka
===========================================================

In this lab, you will create a NiFi flow to receive the data from all
gateways and push it to **Kafka**.

**STEP 1** : Creating a Process Group

Before we start building our flow, let's create a Process Group to help
organizing the flows in the NiFi canvas and also to enable flow version
control.

1.  Open the NiFi Web UI, create a new Process Group and name it
    something like **Process Sensor Data**.

![图片包含 屏幕截图
描述已自动生成](media/image54.jpeg){width="4.965217629046369in"
height="2.268373797025372in"}

2.  On the **NiFi Registry** Web UI, add another bucket for storing the
    Sensor flow we're about to build\'. Call it **SensorFlows**:

![图片包含 屏幕截图
描述已自动生成](media/image55.png){width="5.768055555555556in"
height="4.246527777777778in"}

3.  Back on the **NiFi** Web UI, to enable version control for the
    Process Group, right-click on it and select **Version \> Start
    version control** and enter the details below. Once you complete,
    a ![](media/image56.png){width="0.18055555555555555in"
    height="0.1388888888888889in"} will appear on the Process Group,
    indicating that version control is now enabled for it.

> Registry: NiFi Registry
>
> Bucket: SensorFlows
>
> Flow Name: SensorProcessGroup

![图片包含 屏幕截图
描述已自动生成](media/image57.png){width="5.768055555555556in"
height="1.5381944444444444in"}

4.  Let's also enable processors in this Process Group to use schemas
    stored in Schema Registry. Right-click on the Process Group,
    select **Configure** and navigate to the **Controller
    Services** tab. Click the **+** icon and add
    a **HortonworksSchemaRegistry** service. After the service is added,
    click on the service's *cog* icon
    (![](media/image58.png){width="0.1527777777777778in"
    height="0.1388888888888889in"}), go to the **Properties** tab and
    configure it with the following **Schema Registry URL** and
    click **Apply**.

> URL: http://edge2ai-1.dim.local:7788/api/v1

![图片包含 屏幕截图, 监视器, 计算机, 笔记本电脑
描述已自动生成](media/image59.png){width="5.768055555555556in"
height="4.246527777777778in"}

5.  Click on the *lightning bolt* icon
    (![](media/image60.png){width="0.1111111111111111in"
    height="0.125in"})
    to **enable** the **HortonworksSchemaRegistry** Controller Service.

6.  Still on the **Controller Services** screen, let's add two
    additional services to handle the reading and writing of JSON
    records. Click on the
    ![](media/image61.jpeg){width="0.18055555555555555in"
    height="0.20833333333333334in"}button and add the following two
    services:

> **JsonTreeReader**, with the following properties:
>
> Schema Access Strategy: Use \'Schema Name\' Property
>
> Schema Registry: HortonworksSchemaRegistry
>
> Schema Name: \${schema.name} -\> already set by default!
>
> **JsonRecordSetWriter**, with the following properties:
>
> Schema Write Strategy: HWX Schema Reference Attributes
>
> Schema Access Strategy: Inherit Record Schema
>
> Schema Registry: HortonworksSchemaRegistry

7.  Enable the **JsonTreeReader** and
    the **JsonRecordSetWriter** Controller Services you just created, by
    clicking on their respective *lightning bolt* icons
    (![](media/image60.png){width="0.1111111111111111in"
    height="0.125in"}).

![图片包含 屏幕截图
描述已自动生成](media/image62.png){width="5.768055555555556in"
height="4.246527777777778in"}

**STEP 2** : Creating the flow

1.  Double-click on the newly created **process group** to expand it.

2.  Inside the process group, add a new *Input Port* and name it
    \"**Sensor Data**\"

3.  We need to tell NiFi which schema should be used to read and write
    the Sensor data. For this we'll use
    an ***UpdateAttribute*** processor to add an attribute to the
    FlowFile indicating the schema name.

> Add an ***UpdateAttribute*** processor by dragging the processor icon
> to the canvas:

![图片包含 屏幕截图
描述已自动生成](media/image63.png){width="5.768055555555556in"
height="4.2375in"}

4.  Double-click the ***UpdateAttribute*** processor and configure it as
    follows:

    i.  In the ***SETTINGS*** tab:

> Name: Set Schema Name

ii. In the *PROPERTIES* tab:

Click on the ![](media/image64.png){width="0.1527777777777778in"
height="0.1388888888888889in"} button and add the following property:

> Property Name: schema.name
>
> Property Value: SensorReading

iii. Click **Apply**

<!-- -->

5.  Connect the **Sensor Data** input port to the **Set Schema
    Name** processor.

6.  Add a ***PublishKafkaRecord\_2.0* **processor and configure it as
    follows:

> **SETTINGS** tab:
>
> Name: Publish to Kafka topic: iot
>
> **PROPERTIES** tab:
>
> Kafka Brokers: edge2ai-1.dim.local:9092
>
> Topic Name: iot
>
> Record Reader: JsonTreeReader
>
> Record Writer: JsonRecordSetWriter
>
> Use Transactions: false
>
> Attributes to Send as Headers (Regex): schema.\*

**Note**

Make sure you use the PublishKafkaRecord\_2.0 processor and not the
Publish Kafka\_2.0 one

7.  While still in the ***PROPERTIES*** tab of
    the ***PublishKafkaRecord\_2.0* **processor, click on
    the ![](media/image65.png){width="0.18055555555555555in"
    height="0.19444444444444445in"} button and add the following
    property:

> Property Name: client.id
>
> Property Value: nifi-sensor-data
>
> Later, this will help us clearly identify who is producing data into
> the Kafka topic.

8.  Connect the **Set Schema Name** processor to the **Publish to Kafka
    topic: iot** processor.

9.  Add a new ***Funnel*** to the canvas and connect the
    **PublishKafkaRecord** processor to it. When the \"Create
    connection\" dialog appears, select \"**failure**\" and
    click **Add**.

![图片包含 屏幕截图
描述已自动生成](media/image63.png){width="5.768055555555556in"
height="4.2375in"}

10. Double-click on the **Publish to Kafka topic: iot** processor, go to
    the **SETTINGS** tab, check the \"**success**\" relationship in
    the **AUTOMATICALLY TERMINATED RELATIONSHIPS** section.
    Click **Apply**.

![图片包含 屏幕截图, 监视器, 计算机, 室内
描述已自动生成](media/image66.png){width="5.768055555555556in"
height="4.246527777777778in"}

11. Start the input port and the two processors. Your canvas should now
    look like the one below:

![图片包含 屏幕截图, 监视器, 室内, 墙壁
描述已自动生成](media/image67.png){width="5.768055555555556in"
height="4.246527777777778in"}

12. The only thing that remains to be configured now is to finally
    connect the \"**from Gateway**\" Input Port to the flow in the
    \"**Processor Sensor Data**\" group. To do that, first go back to
    the root canvas by clicking on the **NiFi Flow** link on the status
    bar.

![图片包含 屏幕截图, 监视器, 计算机, 室内
描述已自动生成](media/image68.png){width="5.768055555555556in"
height="4.246527777777778in"}

13. Connect the Input Port to the **Process Sensor Data** Process Group
    by dragging the destination of the current connection from the
    funnel to the Process Group. When prompted, ensure the \"**To
    input**\" fields is set to the **Sensor data** Input Port.

![图片包含 屏幕截图, 监视器, 室内, 计算机
描述已自动生成](media/image69.png){width="5.768055555555556in"
height="4.246527777777778in"}

![图片包含 屏幕截图, 监视器, 室内, 计算机
描述已自动生成](media/image70.png){width="5.768055555555556in"
height="4.246527777777778in"}

14. Refresh the screen (Ctrl+R on Linux/Windows; Cmd+R on Mac) and you
    should see that the records that were queued on the \"**from
    Gateway**\" Input Port disappeared. They flowed into the **Process
    Sensor Data** flow. If you expand the Process Group you should see
    that those records were processed by
    the ***PublishKafkaRecord*** processor and there should be no
    records queued on the \"**failure**\" output queue.

![图片包含 屏幕截图, 监视器, 室内, 计算机
描述已自动生成](media/image70.png){width="5.768055555555556in"
height="4.246527777777778in"}

> At this point, the messages are already in the Kafka topic. You can
> add more processors as needed to process, split, duplicate or re-route
> your FlowFiles to all other destinations and processors.

15. To complete this Lab, let's commit and version the work we've just
    done. Go back to the NiFi root canvas, clicking on the \"Nifi Flow\"
    breadcrumb. Right-click on the **Process Sensor Data** Process Group
    and select **Version \> Commit local changes**. Enter a descriptive
    comment and save.

Lab 7 - Use SMM to confirm that the data is flowing correctly
=============================================================

Now that our NiFi flow is pushing data to Kafka, it would be good to
have a confirmation that everything is running as expected. In this lab
you will use Streams Messaging Manager (SMM) to check and monitor Kafka.

1.  Start the (NiFi **ExecuteProcess**) simulator again and confirm you
    can see the messages queued in NiFi. Leave it running.

2.  Go to the Stream Messaging Manager (SMM) Web UI and familiarize
    yourself with the options there. Notice the filters (blue boxes) at
    the top of the screen.

![图片包含 监视器, 屏幕截图
描述已自动生成](media/image71.png){width="5.768055555555556in"
height="4.246527777777778in"}

3.  Click on the **Producers** filter and select only
    the **nifi-sensor-data** producer. This will hide all the irrelevant
    topics and show only the ones that producer is writing to.

![图片包含 屏幕截图
描述已自动生成](media/image72.png){width="5.768055555555556in"
height="2.3361111111111112in"}

4.  If you filter by **Topic** instead and select the **iot** topic,
    you'll be able to see all the **producers** and **consumers** that
    are writing to and reading from it, respectively. Since we haven't
    implemented any consumers yet, the consumer list should be empty.

![图片包含 屏幕截图
描述已自动生成](media/image73.png){width="5.768055555555556in"
height="1.9055555555555554in"}

5.  Click on the topic to explore its details. You can see more details,
    metrics and the break down per partition. Click on one of the
    partitions and you'll see additional information and which producers
    and consumers interact with that partition.

![图片包含 屏幕截图, 监视器
描述已自动生成](media/image74.png){width="5.768055555555556in"
height="4.246527777777778in"}

6.  Click on the **EXPLORE** link to visualize the data in a particular
    partition. Confirm that there's data in the Kafka topic and it looks
    like the JSON produced by the sensor simulator.

![图片包含 屏幕截图
描述已自动生成](media/image75.png){width="5.768055555555556in"
height="3.004166666666667in"}

![图片包含 屏幕截图
描述已自动生成](media/image76.png){width="5.768055555555556in"
height="4.246527777777778in"}

7.  Check the data from the partition.

![图片包含 屏幕截图
描述已自动生成](media/image77.png){width="5.768055555555556in"
height="4.614583333333333in"}

8.  Stop the simulator with CTRL-C.

9.  In the next Lab we'll eliminate with these problematic measurements
    to avoid problems later in our data flow.

Lab 8 - Update the edge flows to perform additional processing on the data
==========================================================================

In the previous lab we noticed that some of the sensors were sending
erroneous measurements intermittently. If we let these measurements to
be processed by our data flow we might have problems with the quality of
our flow output and we want to avoid that.

We could use our **Process Sensor Data** flow in NiFi to filter out
those problematic measurements. However, if their volume is large we
could be wasting network bandwidth and causing additional overhead in
NiFi to process the bogus data. What we'd like to do instead is to push
additional logic to the edge to identify and filter those problems in
place and avoiding sending them to NiFi in the first place.

We've noticed that the problem always happen with the temperatures in
measurements **sensor\_0** and **sensor\_1**, only. If any of these two
temperatures are **greater than 500** we **must discard** the entire
sensor reading. If both of these temperatures are in the normal range
(\< 500) we can guarantee that all temperatures reported are correct and
can be sent to NiFi.

**STEP 1** : add **EvaluateJSONPath** processor

1.  Go to the CEM Web UI and add a new processor to the canvas. In the
    Filter box of the dialog that appears, type \"JsonPath\". Select
    the ***EvaluateJSONPath*** processor and click **Add**.

2.  Double-click on the new processor and configure it with the
    following properties:

> Processor Name: Extract sensor\_0 and sensor1 values
>
> Destination: flowfile-attribute

![图片包含 监视器, 屏幕截图
描述已自动生成](media/image78.png){width="5.768055555555556in"
height="4.246527777777778in"}

Click on the **Add Property** button and enter the following properties:

  Property Name   Property Value
  --------------- ----------------
  sensor\_0       \$.sensor\_0
  sensor\_1       \$.sensor\_1

![](media/image79.png){width="5.768055555555556in"
height="4.246527777777778in"}

3.  Click **Apply** to save the processor configuration.

**STEP 2** : add **RouteOnAttribute** processor

1.  Drag one more new processor to the canvas. In the Filter box of the
    dialog that appears, type \"Route\". Select
    the ***RouteOnAttribute*** processor and click **Add**.

![图片包含 监视器, 室内, 屏幕截图, 计算机
描述已自动生成](media/image80.png){width="5.768055555555556in"
height="4.246527777777778in"}

2.  Double-click on the new processor and configure it with the
    following properties:

> Processor Name: Filter Errors
>
> Route Strategy: Route to Property name

3.  Click on the **Add Property** button and enter the following
    properties:

  Property Name   Property Value
  --------------- -------------------------------------------------
  error           \${sensor\_0:ge(500):or(\${sensor\_1:ge(500)})}

![图片包含 监视器, 屏幕截图, 室内
描述已自动生成](media/image81.png){width="5.768055555555556in"
height="4.246527777777778in"}

4.  Click **Apply** to save the processor configuration.

**STEP 3** : connect

1.  Reconnect the ***ConsumeMQTT*** processor to the *Extract sensor\_0
    and sensor1 values* processor:

    i.  Click on the existing connection between ***ConsumeMQTT*** and
        the ***RPG*** to select it.

    ii. Drag the destination end of the connection to the ***Extract
        sensor\_0 and sensor1** values* processor.

![图片包含 屏幕截图, 监视器, 室内, 计算机
描述已自动生成](media/image82.png){width="5.768055555555556in"
height="4.246527777777778in"}

2.  Connect the *Extract sensor\_0 and sensor1 values* to the *Filter
    errors* processor. When the **Create Connection** dialog appear,
    select \"**matched**\" and click **Create**.

![图片包含 屏幕截图
描述已自动生成](media/image83.png){width="5.768055555555556in"
height="2.204861111111111in"}

3.  Double-click the *Extract sensor\_0 and sensor1 values* and check
    the following values in the **AUTOMATICALLY TERMINATED
    RELATIONSHIPS** section and click **Apply**:

    i.  failure

    ii. unmatched

    iii. sensor\_0

    iv. sensor\_1

![图片包含 监视器, 屏幕截图, 室内, 笔记本电脑
描述已自动生成](media/image84.png){width="5.768055555555556in"
height="4.246527777777778in"}

1.  Before creating the last connection, you will need (again) the ID of
    the NiFi *Input Port*. Go to the NiFi Web UI , double-click on the
    \"**from Gateway**\" *Input Port* and copy its ID.

![图片包含 屏幕截图, 监视器, 室内, 计算机
描述已自动生成](media/image44.png){width="5.768055555555556in"
height="4.246527777777778in"}

2.  Back on the CEM Web UI, connect the *Filter errors* processor to the
    RPG:

![图片包含 屏幕截图, 监视器, 计算机
描述已自动生成](media/image85.png){width="5.768055555555556in"
height="4.299305555555556in"}

3.  In the **Create Connection** dialog, check the \"**unmatched**\"
    checkbox and enter the copied input port ID, and click
    on **Create**:

![图片包含 屏幕截图
描述已自动生成](media/image86.png){width="5.768055555555556in"
height="2.7881944444444446in"}

4.  To ignore the errors, double-click on the *Filter errors* processor,
    check the **error** checkbox under the **AUTOMATICALLY TERMINATED
    RELATIONSHIPS** section and click **Apply**:

![图片包含 监视器, 屏幕截图, 笔记本电脑, 计算机
描述已自动生成](media/image87.png){width="5.768055555555556in"
height="4.246527777777778in"}

> **STEP 4** : publish CEM

1.  Finally, click on **ACTIONS \> Publish...​** on the CEM canvas,
    enter a descriptive comment like \"Added filtering of erroneous
    readings\" and click **Publish**.

2.  Start the simulator again.

3.  Go to the NiFi Web UI and confirm that the data is flowing without
    errors within the **Process Sensor Data** process group. Refresh a
    few times and check that the numbers are changing.

**STEP 5** : test

1.  Use the **EXPLORE** feature on the SMM Web UI to confirm that the
    bogus readings have been filtered out.

2.  Stop the simulator once you have verified the data.

Lab 9 - Using Hue create Kudu table.
====================================

**Step 1: Create the Kudu table**

1.  Go to the Hue Web UI and login. The first user to login to a Hue
    installation is automatically created and granted admin privileges
    in Hue.

2.  The Hue UI should open with the Impala Query Editor by default. If
    it doesn't, you can always find it by clicking on **Query button \>
    Editor → Impala**:

![图片包含 屏幕截图
描述已自动生成](media/image88.png){width="5.768055555555556in"
height="4.299305555555556in"}

**STEP 2** : Create Kudu table

1.  First, create the Kudu table. Login into Hue, and in the Impala
    Query, run this statement:

> CREATE TABLE sensors
>
> (
>
> sensor\_id INT,
>
> sensor\_ts TIMESTAMP,
>
> sensor\_0 DOUBLE,
>
> sensor\_1 DOUBLE,
>
> sensor\_2 DOUBLE,
>
> sensor\_3 DOUBLE,
>
> sensor\_4 DOUBLE,
>
> sensor\_5 DOUBLE,
>
> sensor\_6 DOUBLE,
>
> sensor\_7 DOUBLE,
>
> sensor\_8 DOUBLE,
>
> sensor\_9 DOUBLE,
>
> sensor\_10 DOUBLE,
>
> sensor\_11 DOUBLE,
>
> is\_healthy INT,
>
> PRIMARY KEY (sensor\_ID, sensor\_ts)
>
> )
>
> PARTITION BY HASH PARTITIONS 16
>
> STORED AS KUDU
>
> TBLPROPERTIES (\'kudu.num\_tablet\_replicas\' = \'1\');

![图片包含 屏幕截图
描述已自动生成](media/image89.png){width="5.768055555555556in"
height="6.253472222222222in"}

Lab 10 - CDSW: Train the model
==============================

In this and the following lab, you will wear the hat of a Data
Scientist. You will write the model code, train it several times and
finally deploy the model to Production. All within 30 minutes!

**STEP 0** : Configure CDSW

Open CDSW Web UI and click on **sign up for a new account**. As you're
the first user to login into CDSW, you are granted admin privileges.
Make sure you use the same username you used when you logged into HUE,
in Lab 0. The usernames here must match.

![图片包含 屏幕截图
描述已自动生成](media/image90.png){width="5.768055555555556in"
height="3.4027777777777777in"}

Navigate to the CDSW **Admin** page to fine tune the environment: - in
the **Engines** tab, add in *Engines Profiles* a new engine (docker
image) with 2 vCPUs and 4 GB RAM, while deleting the default engine. -
add the following in *Environmental Variables*: \` HADOOP\_CONF\_DIR =
/etc/hadoop/conf/ \`

![图片包含 屏幕截图, 室内
描述已自动生成](media/image91.png){width="5.768055555555556in"
height="3.9118055555555555in"}

Please note: this env variable is not required for a CDH 5 cluster.

**STEP 1** : Create the project

Return to the main page and click on **New Project**, using this GitHub
project as the
source: [[https://github.com/fabiog1901/IoT-predictive-maintenance]{.underline}](https://github.com/fabiog1901/IoT-predictive-maintenance).

![图片包含 屏幕截图
描述已自动生成](media/image92.png){width="5.768055555555556in"
height="2.676388888888889in"}

Now that your project has been created, click on **Open Workbench** and
start a Python3 Session

![图片包含 屏幕截图
描述已自动生成](media/image93.png){width="5.768055555555556in"
height="2.925in"}

Once the Engine is ready, run the following command to install some
required libraries:

!pip3 install \--upgrade pip scikit-learn

The project comes with a historical dataset. Copy this dataset into
HDFS:

!hdfs dfs -put data/historical\_iot.txt /user/\$HADOOP\_USER\_NAME

![图片包含 屏幕截图
描述已自动生成](media/image94.png){width="5.768055555555556in"
height="3.529861111111111in"}

You're now ready to run the Experiment to train the model on your
historical data.

You can stop the Engine at this point.

**STEP 2** : Examine cdsw.iot\_exp.py

Open the file cdsw.iot\_exp.py. This is a python program that builds a
model to predict machine failure (the likelihood that this machine is
going to fail). There is a dataset available on hdfs with customer data,
including a failure indicator field.

The program is going to build a failure prediction model using the
Random Forest algorithm. Random forests are ensembles of decision trees.
Random forests are one of the most successful machine learning models
for classification and regression. They combine many decision trees in
order to reduce the risk of overfitting. Like decision trees, random
forests handle categorical features, extend to the multiclass
classification setting, do not require feature scaling, and are able to
capture non-linearities and feature interactions.

spark.mllib supports random forests for binary and multiclass
classification and for regression, using both continuous and categorical
features. spark.mllib implements random forests using the existing
decision tree implementation. Please see the decision tree guide for
more information on trees.

The Random Forest algorithm expects a couple of parameters:

numTrees: Number of trees in the forest. Increasing the number of trees
will decrease the variance in predictions, improving the model's
test-time accuracy. Training time increases roughly linearly in the
number of trees.

maxDepth: Maximum depth of each tree in the forest. Increasing the depth
makes the model more expressive and powerful. However, deep trees take
longer to train and are also more prone to overfitting. In general, it
is acceptable to train deeper trees when using random forests than when
using a single decision tree. One tree is more likely to overfit than a
random forest (because of the variance reduction from averaging multiple
trees in the forest).

In the cdsw.iot\_exp.py program, these parameters can be passed to the
program at runtime, to these python variables:

param\_numTrees = int(sys.argv\[1\])

param\_maxDepth = int(sys.argv\[2\])

Also note the quality indicator for the Random Forest model, are written
back to the Data Science Workbench repository:

cdsw.track\_metric(\"auroc\", auroc)

cdsw.track\_metric(\"ap\", ap)

These indicators will show up later in the **Experiments** dashboard.

**STEP 3** : Run the experiment for the first time

Now, run the experiment using the following parameters:

numTrees = 20 numDepth = 20

From the menu, select Run → Run Experiments...​. Now, in the background,
the Data Science Workbench environment will spin up a new docker
container, where this program will run.

![图片包含 屏幕截图
描述已自动生成](media/image95.png){width="5.768055555555556in"
height="3.475in"}

Go back to the **Projects** page in CDSW, and hit
the **Experiments** button.

If the Status indicates 'Running', you have to wait till the run is
completed. In case the status is 'Build Failed' or 'Failed', check the
log information. This is accessible by clicking on the run number of
your experiments. There you can find the session log, as well as the
build information.

![图片包含 屏幕截图
描述已自动生成](media/image96.png){width="5.768055555555556in"
height="3.1819444444444445in"}

In case your status indicates 'Success', you should be able to see the
auroc (Area Under the Curve) model quality indicator. It might be that
this value is hidden by the CDSW user interface. In that case, click on
the '3 metrics' links, and select the auroc field. It might be needed to
de-select some other fields, since the interface can only show 3 metrics
at the same time.

![图片包含 屏幕截图
描述已自动生成](media/image97.png){width="5.768055555555556in"
height="1.6944444444444444in"}

In this example, \~0.8478. Not bad, but maybe there are better hyper
parameter values available.

**STEP 4** : Re-run the experiment several times

Go back to the Workbench and run the experiment 2 more times and try
different values for NumTrees and NumDepth. Try the following values:

NumTrees NumDepth

15 25

25 20

When all runs have completed successfully, check which parameters had
the best quality (best predictive value). This is represented by the
highest 'area under the curve', auroc metric.

![图片包含 屏幕截图, 道路
描述已自动生成](media/image98.png){width="5.768055555555556in"
height="1.525in"}

**STEP 5** : Save the best model to your environment

Select the run number with the best predictive value, in this case,
experiment 2. In the Overview screen of the experiment, you can see that
the model in spark format, is captured in the file iot\_model.pkl.
Select this file and hit the **Add to Project** button. This will copy
the model to your project directory.

![图片包含 屏幕截图
描述已自动生成](media/image99.png){width="5.768055555555556in"
height="4.06875in"}

![图片包含 屏幕截图
描述已自动生成](media/image100.png){width="5.768055555555556in"
height="5.363194444444445in"}

Lab 11 - CDSW: Deploy the model
===============================

**STEP 1** : Examine the program cdsw.iot\_model.py

Open the project you created in the previous lab, and examine the file
in the Workbench. This PySpark program uses the pickle.load mechanism to
deploy models. The model it refers to the iot\_modelf.pkl file, was
saved in the previous lab from the experiment with the best predictive
model.

There is a predict definition which is the function that calls the
model, using features, and will return a result variable.

Before deploying the model, try it out in the Workbench: launch a
Python3 engine and run the code in file cdsw.iot\_model.py. Then call
the predict() method from the prompt:

predict({\"feature\": \"0, 65, 0, 137, 21.95, 83, 19.42, 111, 9.4, 6,
3.43, 4\"})

![图片包含 屏幕截图
描述已自动生成](media/image101.png){width="5.768055555555556in"
height="1.78125in"}

The functions returns successfully, so we know we can now deploy the
model. You can now stop the engine.

**STEP 2** : Deploy the model

From the projects page of your project, select the **Models** button.
Select **New Model** and populate specify the following configuration:

Name: IoT Prediction Model

Description: IoT Prediction Model

File: cdsw.iot\_model.py

Function: predict

Example Input: {\"feature\": \"0, 65, 0, 137, 21.95, 83, 19.42, 111,
9.4, 6, 3.43, 4\"}

Kernel: Python 3

Engine: 2 vCPU / 4 GB Memory

Replicas: 1

![图片包含 屏幕截图
描述已自动生成](media/image102.png){width="5.768055555555556in"
height="5.758333333333334in"}

If all parameters are set, you can hit the **Deploy Model** button. Wait
till the model is deployed. This will take several minutes.

**STEP 3** : Test the deployed model

After several minutes, your model should get to the **Deployed** state.
Now, click on the Model Name link, to go to the Model Overview page.
From the that page, hit the **Test** button to check if the model is
working.

The green color with success is telling that our REST call to the model
is technically working. And if you examine the response: {\"result\":
1}, it returns a 1, which mean that machine with these features is
likely to stay healthy.

![图片包含 屏幕截图
描述已自动生成](media/image103.png){width="5.768055555555556in"
height="3.5506944444444444in"}

Now, lets change the input parameters and call the predict function
again. Put the following values in the Input field:

{

\"feature\": \"0, 95, 0, 88, 26.62, 75, 21.05, 115, 8.65, 5, 3.32, 3\"

}

With these input parameters, the model returns 0, which means that the
machine is likely to break. Take a note of the **AccessKey** as you will
need this for lab 10.

Lab 12 - Use NiFi to call the CDSW model endpoint and save to Kudu
==================================================================

In this lab, you will use NiFi to consume the Kafka messages containing
the IoT data we ingested in the previous lab, call a CDSW model API
endpoint to predict whether the machine where the readings came from is
likely to break or not.

In preparation for the workshop we trained and deployed a Machine
Learning model on the Cloudera Data Science Workbench (CDSW) running on
your cluster. The model API can take a feature vector with the reading
for the 12 temperature readings provided by the sensor and predict,
based on that vector, if the machine is likely to break or not.

**STEP 1** : Add new Controller Services

When the sensor data was sent to Kafka using
the ***PublishKafkaRecord*** processor, we chose to attach the schema
information in the header of Kafka messages. Now, instead of hard-coding
which schema we should use to read the message, we can leverage that
metadata to dynamically load the correct schema for each message.

To do this, though, we need to configure a
different ***JsonTreeReader*** that will use the schema properties in
the header, instead of the \${schema.name} attribute, as we did before.

We'll also add a new ***RestLookupService*** controller service to
perform the calls to the CDSW model API endpoint.

1.  If you're not in the **Process Sensor Data** process group,
    double-click on it to expand it. On the **Operate** panel (left-hand
    side), click on the *cog* icon
    (![](media/image104.png){width="0.19444444444444445in"
    height="0.20833333333333334in"}) to access the **Process Sensor
    Data** process group's configuration page.

![图片包含 屏幕截图
描述已自动生成](media/image105.png){width="5.768055555555556in"
height="3.6416666666666666in"}

2.  Click on the *plus* button
    (![](media/image106.png){width="0.18055555555555555in"
    height="0.16666666666666666in"}), add a new **JsonTreeReader**,
    configure it as shown below and click **Apply** when you're done:

> On the **SETTINGS** tab:
>
> Name: JsonTreeReader - With schema identifier
>
> On the **PROPERTIES** tab:
>
> Schema Access Strategy: HWX Schema Reference Attributes
>
> Schema Registry: HortonworksSchemaRegistry

3.  Click on the *lightning bolt* icon
    (![](media/image107.png){width="0.1527777777777778in"
    height="0.125in"}) to **enable** the **JsonTreeReader - With schema
    identifier** controller service.

4.  Click again on the *plus* button
    (![](media/image106.png){width="0.18055555555555555in"
    height="0.16666666666666666in"}), add
    a **RestLookupService** controller service, configure it as shown
    below and click **Apply** when you're done:

> On the **PROPERTIES** tab:
>
> URL:
> http://cdsw.\<YOUR\_CLUSTER\_PUBLIC\_IP\>.nip.io/api/altus-ds-1/models/call-model
>
> Record Reader: JsonTreeReader
>
> Record Path: /response

**Note**:

\<YOUR\_CLUSTER\_PUBLIC\_IP\> above must be replaced with your cluster's
public IP, **not** DNS name. The final URL should look something like
this: http://cdsw.12.34.56.78.nip.io/api/altus-ds-1/models/call-model

5.  Click on the *lightning bolt* icon
    (![](media/image107.png){width="0.1527777777777778in"
    height="0.125in"})
    to **enable** the **RestLookupService** controller service.

![图片包含 屏幕截图
描述已自动生成](media/image108.png){width="5.768055555555556in"
height="4.246527777777778in"}Close the **Process Sensor Data
Configuration** page.

**STEP 2** : Create the flow

We'll now create the flow to read the sensor data from Kafka, execute a
model prediction for each of them and write the results to Kudu. At the
end of this section you flow should look like the one below:

![图片包含 监视器, 室内, 计算机, 笔记本电脑
描述已自动生成](media/image109.png){width="5.768055555555556in"
height="4.246527777777778in"}

**ConsumeKafkaRecord\_2\_0** processor

1.  We'll add a new flow to the same canvas we were using before (inside
    the **Process Sensor Data** Process Group). Click on an empty area
    of the canvas and drag it to the side to give you more space to add
    new processors.

2.  Add a **ConsumeKafkaRecord\_2\_0** processor to the canvas and
    configure it as shown below:

> **SETTINGS** tab:
>
> Name: Consume Kafka iot messages
>
> **PROPERTIES** tab:
>
> Kafka Brokers: edge2ai-1.dim.local:9092
>
> Topic Name(s): iot
>
> Topic Name Format: names
>
> Record Reader: JsonTreeReader - With schema identifier
>
> Record Writer: JsonRecordSetWriter
>
> Honor Transactions: false
>
> Group ID: iot-sensor-consumer
>
> Offset Reset: latest
>
> Headers to Add as Attributes (Regex): schema.\*

3.  Add a new *Funnel* to the canvas and connect the **Consume Kafka iot
    messages** to it. When prompted, check
    the **parse.failure** relationship for this connection:

![图片包含 屏幕截图
描述已自动生成](media/image110.png){width="5.768055555555556in"
height="4.2652777777777775in"}

**STEP 3** : LookupRecord processor

1.  Add a **LookupRecord** processor to the canvas and configure it as
    shown below:

> **SETTINGS** tab:
>
> Name: Predict machine health
>
> **PROPERTIES** tab:
>
> Record Reader: JsonTreeReader - With schema identifier
>
> Record Writer: JsonRecordSetWriter
>
> Lookup Service: RestLookupService
>
> Result RecordPath: /response
>
> Routing Strategy: Route to \'success\'
>
> Record Result Contents: Insert Entire Record

2.  Add 3 more user-defined properties by clicking on the *plus* button
    (![](media/image106.png){width="0.18055555555555555in"
    height="0.16666666666666666in"}) for each of them:

> mime.type: toString(\'application/json\', \'UTF-8\')
>
> request.body: concat(\'{\"accessKey\":\"\', \'\${cdsw.access.key}\',
> \'\",\"request\":{\"feature\":\"\', /sensor\_0, \', \', /sensor\_1,
> \', \', /sensor\_2, \', \', /sensor\_3, \', \', /sensor\_4, \', \',
> /sensor\_5, \', \', /sensor\_6, \', \', /sensor\_7, \', \',
> /sensor\_8, \', \', /sensor\_9, \', \', /sensor\_10, \', \',
> /sensor\_11, \'\"}}\')
>
> request.method: toString(\'post\', \'UTF-8\')

3.  Click **Apply** to save the changes to the **Predict machine
    health** processor.

4.  Connect the **Consume Kafka iot messages** processor to
    the **Predict machine health** one. When prompted, check
    the **success** relationship for this connection.

5.  Connect the **Predict machine health** to the same *Funnel* you had
    created above. When prompted, check the **failure** relationship for
    this connection.

**STEP 4** : UpdateRecord processor

1.  Add a **UpdateRecord** processor to the canvas and configure it as
    shown below:

> **SETTINGS** tab:
>
> Name: Update health flag
>
> **PROPERTIES** tab:
>
> Record Reader: JsonTreeReader - With schema identifier
>
> Record Writer: JsonRecordSetWriter
>
> Replacement Value Strategy: Record Path Value

2.  Add one more user-defined propertie by clicking on the *plus* button
    (![](media/image106.png){width="0.18055555555555555in"
    height="0.16666666666666666in"}):

> /is\_healthy: /response/result

3.  Connect the **Predict machine health** processor to the **Update
    health flag** one. When prompted, check the **success** relationship
    for this connection.

4.  Connect the **Update health flag** to the same *Funnel* you had
    created above. When prompted, check the **failure** relationship for
    this connection.

**STEP 5** : PutKudu processor

1.  Add a **PutKudu** processor to the canvas and configure it as shown
    below:

> **SETTINGS** tab:
>
> Name: Write to Kudu
>
> **PROPERTIES** tab:
>
> Kudu Masters: edge2ai-1.dim.local:7051
>
> Table Name: impala::default.sensors
>
> Record Reader: JsonTreeReader - With schema identifier

2.  Connect the **Update health flag** processor to the **Write to
    Kudu** one. When prompted, check the **success** relationship for
    this connection.

3.  Connect the **Write to Kudu** to the same *Funnel* you had created
    above. When prompted, check the **failure** relationship for this
    connection.

4.  Double-click on the **Write to Kudu** processor, go to
    the **SETTINGS** tab, check the \"**success**\" relationship in
    the **AUTOMATICALLY TERMINATED RELATIONSHIPS** section.
    Click **Apply**.

**STEP 6** : CDSW Access Key

When we added the **Predict machine health** above, you may have noticed
that one of the properties (request.body) makes a reference to a
variable called cdsw.access.key. This is an application key required to
authenticate with the CDSW Model API when requesting predictions. So, we
need to provide the key to the *LookupRecord* processor by setting a
variable with its value.

1.  To get the Access Key, go to the CDSW Web UI and click
    on **Models \> Iot Prediction Model \> Settings**. Copy the Access
    Key.

![](media/image111.png){width="5.768055555555556in"
height="4.246527777777778in"}

2.  Go back to the NiFi Web UI, right-click on an empty area of
    the **Process Sensor Data** canvas, and click on **Variables**.

3.  Click on the *plus* button
    (![](media/image106.png){width="0.18055555555555555in"
    height="0.16666666666666666in"}) and add the following variable:

> Variable Name: cdsw.access.key
>
> Variable Value: \<key copied from CDSW\>

![](media/image112.png){width="5.768055555555556in"
height="4.246527777777778in"}

Click **Apply**

**STEP 7** : Running the flow

We're ready not to run and test our flow. Follow the steps below:

1.  Start all the processors in your flow.

2.  Refresh your NiFi page and you should see messages passing through
    your flow. The failure queues should have no records queued up.

![](media/image113.png){width="5.768055555555556in"
height="4.246527777777778in"}

Lab 13 - Use Spark to call a CDSW model endpoint and save to Kudu
=================================================================

Spark Streaming is a processing framework for (near) real-time data. In
this lab, you will use Spark to consume Kafka messages which contains
the IoT data from the machine, and call a CDSW model API endpoint to
predict whether, with those IoT values the machine sent, the machine is
likely to break. You'll then save the results to Kudu for fast
analytics.

**Step 1 : CDSW Access Key**

1.  To configure and run the Spark Streaming job, you will need a CDSW
    Access Key to connect to the model endpoint that has been deployed
    there. To get the Access Key, go to the CDSW Web UI and click
    on **Models \> Iot Prediction Model \> Settings**. Copy the Access
    Key.

![model access key](media/image114.png){width="5.768055555555556in"
height="4.247222222222222in"}

Step 2 :Running the Spark job

1.  Open a Terminal and SSH into the VM. The first is running the sensor
    data simulator, so you can't use it.

ACCESS\_KEY=\<put here your cdsw model access key\>

cd \~

wget
http://central.maven.org/maven2/org/apache/kudu/kudu-spark2\_2.11/1.9.0/kudu-spark2\_2.11-1.9.0.jar

wget
https://raw.githubusercontent.com/swordsmanliu/SparkStreamingHbase/master/lib/spark-core\_2.11-1.5.2.logging.jar

rm -rf \~/.m2 \~/.ivy2/

spark-submit \\

\--master local\[2\] \\

\--jars kudu-spark2\_2.11-1.9.0.jar,spark-core\_2.11-1.5.2.logging.jar
\\

\--packages org.apache.spark:spark-streaming-kafka\_2.11:1.6.3 \\

/opt/demo/spark.iot.py \$ACCESS\_KEY

2.  Spark Streaming will flood your screen with log messages, however,
    at a 5 seconds interval, you should be able to spot a table: these
    are the messages that were consumed from Kafka and processed by
    Spark. You can configure Spark for a smaller time window, however,
    for this exercise 5 seconds is sufficient.

![spark job output](media/image115.png){width="5.768055555555556in"
height="1.3041666666666667in"}

Lab 14 - Fast analytics on fast data with Kudu and Impala
=========================================================

In this lab, you will run some SQL queries using the Impala engine. You
can run a report to inform you which machines are likely to break in the
near future.

1.  Login into Hue and run the following queries in the Impala Query
    Editor:

> SELECT sensor\_id, sensor\_ts
>
> FROM sensors
>
> WHERE is\_healthy = 0;
>
> SELECT is\_healthy, count(\*) as occurrences
>
> FROM sensors
>
> GROUP BY is\_healthy;

2.  Run a few times the SQL statements and verify that the number of
    occurrences are increasing as the data is ingested by either NiFi or
    the Spark job. This allows you to build real-time reports for fast
    action.

![table select](media/image116.png){width="5.768055555555556in"
height="5.267361111111111in"}

 Resources
==========

-   [[Original blog by Abdelkrim
    Hadjidj]{.underline}](https://medium.freecodecamp.org/building-an-iiot-system-using-apache-nifi-mqtt-and-raspberry-pi-ce1d6ed565bc)

-   This workshop was based on the following work by Andre Araujo:

    -   [[https://github.com/asdaraujo/edge2ai-workshop]{.underline}](https://github.com/asdaraujo/edge2ai-workshop)

-   This workshop was based on the following work by Timothy Spann:

    -   [[https://github.com/tspannhw/edge2ai-workshop]{.underline}](https://github.com/tspannhw/edge2ai-workshop)

-   That workshop was based on the following work by Fabio Ghirardello:

    -   [[https://github.com/fabiog1901/IoT-predictive-maintenance]{.underline}](https://github.com/fabiog1901/IoT-predictive-maintenance)

    -   [[https://github.com/fabiog1901/OneNodeCDHCluster]{.underline}](https://github.com/fabiog1901/OneNodeCDHCluster)

-   [[Cloudera
    Documentation]{.underline}](https://www.cloudera.com/documentation.html)

Troubleshooting
===============

\*MiNiFi Not Sending Messages \*
--------------------------------

-   Make sure you pick S2S not RAW in Cloud Connection to NiFi

-   Make sure No Spaces Before or After Destination ID, URL, Names,
    Topics, Brokers, Etc...​

-   Make sure No Spaces Anywhere

-   Everything is Case-Sensitive. It's IoT, not iot.

-   Use **User=Admin** for CDSW and HUE

-   You must have HDFS User Created via HUE, Go there First

-   Check all your connections and spellings

-   Check **/opt/cloudera/cem/minifi/logs/minifi-app.log** if you can't
    find an issue

CEM doesn't pick up new NARs
----------------------------

1.  Delete the agent manifest manually using the EFM API:

2.  Verify each class has the same agent manifest ID:

3.  http://hostname:10080/efm/api/agent-classes

> \[{\"name\":\"iot1\",\"agentManifests\":\[\"agent-manifest-id\"\]},{\"name\":\"iot4\",\"agentManifests\":\[\"agent-manifest-id\"\]}\]

4.  Confirm the manifest doesn't have the NAR you installed

5.  http://hostname:10080/efm/api/agent-manifests?class=iot4

> \[{\"identifier\":\"agent-manifest-id\",\"agentType\":\"minifi-java\",\"version\":\"1\",\"buildInfo\":{\"timestamp\":1556628651811,\"compiler\":\"JDK
> 8\"},\"bundles\":\[{\"group\":\"default\",\"artifact\":\"system\",\"version\":\"unversioned\",\"componentManifest\":{\"controllerServices\":\[\],\"processors\":

6.  Call the API endpoint:

> http://hostname:10080/efm/swagger/

7.  Hit the **DELETE** - Delete the agent manifest specified by
    id button, and in the id field, enter agent-manifest-id
