---
layout: post
title: 'Prototyping an Asset Tracking System with MIOTY'
---

For this post I’d like to talk about an IoT system I prototyped for a local company here in Nürnberg, Germany as part of my bachelor’s thesis. This company specializes in the logistics of transport, set-up, and teardown of trade conferences all around Europe. The company was fortunate enough to grow their business to a point that it was becoming more and more difficult to manage their resources and assets effectively. Among these difficulties were issues with theft from conferences and trade shows, poor capacity and resource management due to complex communication channels, and a lack of an automatic reporting system to track assets and notify management of state changes. Therefore, the goal of this thesis was to build an asset tracking system based on IoT principles using Low-Power Wide-Area Network (LPWAN) modules. 

{% include image.html image="projects/proj-4/asset_tracking_system.png" %}



#### System requirements

From the point of view of the client company, there were four key system requirements that must be fulfilled for the system to be considered a success:

1.	The company must be able to find their inventory quickly and easily – whether the items are in transport, in storage (and in which storage facility for that matter), on the floor at a trade show, or, in the cases of theft, if the item is somewhere it shouldn’t be.

2.	The company must be able to query and track the current condition of the article. Conditions include whether an item is damaged, the current and past temperature, battery status, light intensity, etc.

3.	The company must be able to identify defective localization and sensing modules.

4.	The company must be able to continue using their current in-house Enterprise Resource Planning (ERP) system – i.e., the asset tracking system must interface with the ERP.

#### System design

{% include image.html image="projects/proj-4/overview.png" %}

Above is a high-level overview of the asset tracking system. The system comprises a network of transceivers (up to 200,000(!) are planned for the final implementation) attached to individual inventory items, an IoT communication protocol, two databases, and the ERP interface. 

##### Communication protocol

On the left are the LPWAN transceivers – these transceivers send messages about the location and the item’s state (only the location message is shown here) using the MIOTY protocol to a central location, the MQTT broker, to be distributed later to interested parties/devices. MQTT is a Machine-to-Machine (M2M) messaging protocol designed specifically for use by IoT devices and relies on a bidirectional publish-subscribe model to transport its messages. The transceivers send information to their respective base stations, the information is decoded, and then uploaded to the MQTT broker under a certain topic (location, state, etc.), which acts a distribution center. The MQTT broker then “publishes” the messages to each device (the client) that “subscribes” to the designated topic. MQTT works so well for this project because it doesn’t require that the various devices know how to communicate with one another – they only have to know how to communicate with the broker. This is an ideal scenario for modular systems. 

{% include image.html image="projects/proj-4/mqtt.png" %}

##### Data storage

There are two databases used in the asset tracking system. A relational SQL database (here: MySQL, but could be any other relational DB like PostgreSQL, etc.) and a nonrelational database, MongoDB. The idea is that some data transmitted by the transceivers will be very fluid, changing all the time, and may have missing data, while other types of data is quite structured, should ideally never change and should be sent without failure – hence, the two different databases. MongoDB is used to store all data sensor data and all localization data. A NoSQL database is a good tool here, because the needs of the client company may change from time to time, e.g., adding additional sensor fields. MongoDB allows changes to be made to the database without any downtime or loss of information because there is no schema that needs to be defined beforehand. Furthermore, one needs to keep in mind that no sensor and no transceiver is perfect – from time to time a sensor may fail and not deliver information. This is no problem for MongoDB, again, because there is no rigid schema that must be followed. On the other hand, the sensor and localization data are not the only data being transmitted. Inventory properties, such as the article ID, article type, etc., and transceiver information such as the MAC address and serial number must also be transmitted. This information doesn’t change and, assuming there is no failure in the transceiver, will always be transmitted. This type of data fits well in a rigid, structured table and is therefore ideal for a relational database. Both databases are connected with one another and with the MQTT broker using Python and its respective interfacing libraries – pymongo and mysqlconnector. 

##### Event driven actions

In the system overview, it can be seen that each message gets pushed to MongoDB. However, we also see that data is only transmitted to the ERP interface if certain events take place. These events prevent the system from constantly notifying management or users about uninteresting information, such as an item sitting in storage with no plans to move it – only important or requested information is passed to the ERP interface. There are two types of events. The cleverly named “manual events” are events that are directly executed by the user, while the equally uniquely named “automatic events” take place without any user interaction whatsoever. Each event records the time of the event and the most recent response time of a transceiver module. In doing so, the client company can quickly identify if a module has failed and needs to be replaced. For example, should a request time and response time differ by several hours or days, it’s highly likely that the transceiver module has failed.

Manual events are requests by a user for information regardless of urgency or “importance.” Essentially, manual events are database queries which lead to an entry in the ERP interface. There are three types of manual events:

1.	Inventory query – a status update (location, time of request, and all sensing values) for each article in the client company’s inventory is recorded in the ERP interface. This provides a comprehensive overview of all assets.

2.	Location query – the current status (location, time of request, and all sensing values) is recorded for each article that is located at the passed location, e.g., a location name or city. This serves as a location-based filter for all assets.

3.	MAC list query – the current status (location, time of request, and all sensing values) is recorded for each article that is individually passed to the system. Articles are passed according to their unique MAC ID and can take the form of a singular item or a group of items, e.g., a packing list. 


Automatic events, however, serve as the main monitoring tool of the system. There are two automatic events:

1.	Threshold monitor – a notification is sent to management and an entry is recorded in the ERP interface if a sensor value is outside of a defined threshold. And example could be a low battery status, very warm or cold temperature readings, or high acceleration readings followed by a sudden stop – indicating a falling action.

2.	Automatic status update – this update is the core monitoring feature. The user can set an update period, once per hour for example, and a status update for all items will be recorded in the ERP interface once the update period has passed. 

##### ERP interface

The final piece of the system is the ERP interface. The client company’s current ERP has a tabular structure, and as such, tabular inputs are ideal. Therefore, all data that is recorded by event triggers are placed into a CSV file, which acts as a simple and efficient ERP interface. The final view of the interface is shown below.

{% include image.html image="projects/proj-4/erp-interface.png" %}

#### Prototype and realization

To give the client company a visual representation of how the asset tracking would perform in a live environment, an Apache HTML server was setup and hosted on a local environment. A simple webpage was created to allow a user to pass values for a manual event/request. After registering a manual event, the user would be redirected to a second webpage showing the current state of the requested inventory. In addition, an interactive map using the leafly.js library was included to allow management to visually see the location of an item. The web pages can be seen below.

{% include image.html image="projects/proj-4/landing.png" %}
{% include image.html image="projects/proj-4/inventory.png" %}

Since the transceiver modules were not yet live at the time, pseudo-nodes were created, which would randomly draw from a previously created distribution of values for each sensor type and then send that data in predefined intervals. To simulate failure, each node was also assigned a probability of failure. These nodes continually passed information to the MQTT broker, which would then pass on the data to MongoDB and, if necessary, to the ERP interface. Each triggered manual or automatic event appended to a CSV file, which was available to download at any moment. All scripts for this thesis were written using Python with the exception of the interactive map, which was written using javascript. 

#### Final thoughts
This was a very educational experience, and I was thrilled to have a thesis that dealt so much on a practical topic, rather than a purely academic thesis. I learned a lot about wireless technologies, IoT devices and topics, and how to translate requests and ideas from management into useable code and features. The client company was also very pleased with the system for which I was also very pleased, since that had an obvious influence on my final grade 😉


