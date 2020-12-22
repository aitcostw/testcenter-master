# TAAS EdgeOS TestCenter Demo - Demo running in Lab on 8/11/20
# updated check: 10/20/2020 
## Overview

This is an advanced sample application that demonstrates many of the key capabilities enabled by EdgeOS, such as: 
 * The ability to deploy and run on edge devices, such as Raspberry Pi’s
 * The ability to distribute an application that shares consistency between nodes
 * The ability to aggregate data on a fabric at many levels
 * The ability to visualize data through a real-time UI

This application is deployed in the TestCenter office for monitoring our testers and serves as a demonstration of best practices. Therefore, the description of this application will be through a hypothetical TestCenter scenario even though the application could be applied to many use cases involving sensors, DUT testing real-time Data collection and/or a network of devices. 

aiTAAS Edge Project for TestCenter has two key technologies: EdgeOS and EdgeNC. 
  * EdgeOS is a real-time Data Classification & Analytics OS for IIOT sensors and DUT test data classification, reduction, analysis and prediction.  
  * EdgeNC is an AI/ML-enabled edge computing device with network data transfer capabilities, powered by CPU and FPGA that has Deep Neural Network (DNN) models; Inference with ML; Preprocess data; Transfer data over network to Cloud.

Benefits: This Solution has the benefits:
  * Faster: Process DUT Test Data locally: Transform in real-time; Combine all your data sources
  * Better: Reduce data loss
  * Advanced: Bring advanced ML capabilities to Adaptive Testing
  * Chapter: Develop own models using latest ML technologies
  * Easier:  Simple configuration - Effortless onboarding; Scale to massive data volume

Inside the TestCenter, there are a number of Raspberry Pi devices with various roles. These roles are: Tester Monitor, Automated Robot, and Data Aggregator. It’s important to note that all Raspberry Pi devices run the same software with minor configuration changes that define how they act and participate within the greenhouse network. This diagram provides a simple overview of the complete application and configurations.

![Demo Application Map](https://raw.githubusercontent.com/aimwts/aiTAAS-Edge/master/Taas_logo_s.jpg "Demo Application Map")


![Demo Application Map](https://raw.githubusercontent.com/aimwts/aiTAAS-Edge/master/Edge%20map.jpg "Demo Application Map")

![Demo Application Map](https://raw.githubusercontent.com/aimwts/aiTAAS-Edge/master/STH%20STC1000.jpg "Demo Application Map")

![Demo Application Map](https://raw.githubusercontent.com/aimwts/aiTAAS-Edge/master/TC%20IP%20config.jpg "Demo Application Map")

Each device is also running a minimal NodeJS server alongside the EdgeOS Web Agents. Node is used to serve the various status pages used by the web app and as a data bridge from the sensors into EdgeOS. Unlike a traditional web application where all the pages are hosted in a central place, EdgeOS web applications can be hosted from each device. This ensures the UI is showing the real time data possible without the latency of being routed through various databases and/or cloud services.

A *Tester Monitor* device, or *TesterMon* for short, is a single Raspberry Pi running both EDgeOS and Node. The device's role as a TesterMon is to report the current state of a single Tsensor to which its attached. That Pi is connected to an Arduino via a serial connection which is read by a service in Node. That Arduino has one or more sensors attached to it and simply sends the values of each sensor over the serial connection as a JSON message. Those messages are picked up by the bridge, parsed and sent into EdgeOS. In this case the bridge is in Node but it can also be in Java or Python. Each sensor will automatically get its own Sensor Web Agent inside EdgeOS. That Sensor Web Agent will have Lanes which hold the latest sensor value, a history of that sensor value, Alert Lanes and Alert Threshold Lanes which manage triggering alerts based on each sensor value.

An *Automated Robot*, or just Bot for short, is functionally just a raspberry Pi with a [SenseHat attachment](https://www.raspberrypi.org/products/sense-hat/). Bots sit idle within the TestCenter until an alert is triggered on one or more of of the TesterMon sensors. Alerts are automatically assigned by an Aggregator to the next idle Bot, and that Bot's task is cleared when the alert is cancelled. Like TesterMon, each Bot device runs both EdgeOS and Node. In this case the data is bridged into EdgeOS via Node which uses an third party JavaScript library to read the sensor values from the SensHat and pipe those values into EdgeOS. 

The *Aggregator* device sits on the network and receives data from every tester and bot on the network. Like TesterMon and Bot, the Aggregator runs both EdgeOS and Node however unlike the other two devices there is not a data bridge running. The role of the Aggregator is to monitor everything configured to report to it and provide a UI which displays the status of all the connected devices. The Aggregator is also responsible for assigning 'tasks' to Bots when one or more TesterMon devices report an alert. When one more alerts are sent from a TesterMon on the network, the aggregator will selected an idle Bot on that same network and assign the alert to that bot. At that point, hypothetically the bot is acting on the alert. In practice the bot can only acknowledge the alert and wait for the sensor value to go back up outside of the threshold range. Acknowledgement is shown by the animated EdgeOS logo on the Bot's SenseHat screen turning from blue to red. Once an alert is cleared by either resolving the issue (such as turning on the fan) or by adjusting the alert threshold, the 'task' will be cleared from the Aggregator and the Bot will return to an available status turning the Swim logo back to blue.

Every device on the network has a corresponding webpage which shows the current status of that particular device. As a user you are able to navigate between testers, bots, and the aggregator as if it were a single webapp despite each page being served from each device. The details of how each device acts and connects on the network is defined within the config files found in /config. This allows for every device to run identical software with only minor configuration changes defining how the device behaves.

## Getting Started

All the code required to deploy and run this application is located in this repository, including bash scripts to get things up and running. You will need to be running in a Unix like environment to be able to run this demo. We use both Debian and Ubuntu but you are not limited to just those distributions. For Windows you can setup an compatible environment: the [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/install-win10) (Prefer to run Ubuntu 18.04 LTS). NodeJS 9.11.1 or greater, Java 9 or greater, Git, and Gradle are all required in order to run the demo. Be sure those are setup before continuing.

To get started:
1. Clone the repo `git clone https://github.com/aimwts/testcenter.git`
2. CD into the new directory `cd testcenter`
3. from the main project directory run:
   * `./bin/buildAll.sh config=raspi2`, this will build both Swim and install NodeJS dependencies
   * `./bin/startAll.sh config=raspi2`, this will start both Swim and NodeJS
4. Navigate your browser to the address and port you configured your device to live at and verify that it serves out a page. If so you are setup!

## Configuration
  Each device gets two configuration files which define how that device acts on the network of devices. One file defines what is needed for the EdgeOS Web Agent and the other holds what is needed for NodeJS. Within the /config folder these files live in /java/ and /node respectively. You can define which set of configuration files to use at startup using the 'config' command line parameter. See the Scripts section below for more information about how to use this parameter. It is important to follow the naming convention shown in the example files when creating configuration files for new devices. An overview of the options within both files is below.

  1. config/java/'*deviceName*'-app.properties - used by the EdgeOS service
     * *aggregate.host.uri* - this should be the websocket address of the device acting as the aggregator on the network
     * *device.name* - this the 'Bot Name' and is used in a number of places. Importantly the name needs to have 'Agg', 'Plant', or 'Bot'. Those are used in a number of places to decide what type of bot the current device is. The IP address is inside the name is used by the UI for linking between devices this mean the name format must be something like nameOfBot|ip.of.bot with a pipe separating the name and IP.
     * *device.host.uri* -  this is the address of the current device. This is also used for linking devices together on the network.
     * *swim.port* - port of the EdgeOS service. default is 5620
     * *sensor.resolution* - The time in milliseconds over which we will sample time series data.

  2. config/node/'*deviceName*'Config.js

     * httpEnabled - this boolean value is used to turn on/off the express http server in NodeJS
     * showDebug - boolean which turns on debug output in the codebase
     * botName - name of the bot used app.properties
     * hostUrl - this is the address of the current device on the network
     * aggregateHost - this is the address of the device which is acting as the aggregator on the network
     * hostPort - this is the port which the http server will serve pages over
     * swimPort - post used by the SWIM service
     * senseHatConfig - SenseHats are the 'Bots' on the network
        * bot - configuration for the senseHat bot 
          * enabled - boolean to turn on/off the senseHat Bot
        * service - configuration for senseHat service
          * enabled - boolean to turn on/off the senseHat Service 
          * polling - used to define if polling should be used to handle serial data and what interval in milliseconds the polling interval will be
     * plantConfig - configuration which defines the TesterMon
        * bot - configuration for the TesterMon bot
          * enabled - boolean to turn on/off the plant monitor
        * service - configuration for TesterMon service
          * enabled - boolean to turn on/off the senseHat Service 
          * arduinoAddress - address of the serial port the arduino is connected on.
          * baud - the baud rate at which to communicate over serial with arduino
          * polling - used to define if polling should be used to handle serial data and what interval in milliseconds the polling interval will be
      
## Running

This example has a number of bash scripts which are used to manage both node and EdgeOS services as well as convenience scripts to manage both at the same time. The scripts expect to the run from the base directory of the project on your local machine and may throw errors of run anywhere else. The scripts will handle packaging, publishing, and deploying your local code to one or more remote devices.

In order to simplify using all the various scripts a parent bash script called AppManager is used to build, deploy, and start the app both locally and on remote devices.

The information below is also available on the command line by calling `./bin/appManager.sh help`

## AppManager Usage 

**./bin/appManager.sh [`command`] [`"list of raspi IDs"`] [`remotePassword`]**

* [`command`] - required. See below for the full list of commands
* [`"list of raspi IDs"`] - (remote only) list of pis to be managed. Does lookup into iplist in global.sh
* [`remotePassword`] - (remote only) ssh password for the device(s)

### AppManager Commands

*Local App Management Commands*

* **`start`** - start up both node and EdgeOS
* **`startNode`** - start up just node
* **`startSwim`** - start up just EdgeOS
* **`stop`** - stop node, EdgeOS, and chromium
* **`stopNode`** - stop just node
* **`stopSwim`** - stop just EDgeOS
* **`build`** - build both node (npm) and swim (gradle)
* **`buildNode`** - build node with npm
* **`buildSwim`** - build EDgeOS with gradle and untar to dist
* **`package`** - creates a gzip package file to the /dist directory of both node and EdgeOS codebase which is then used for remote deployment
* **`packageNode`** - creates a package of just the node side of the app
* **`packageSwim`** - creates a package of just the EdgeOS side of the app

*Remote AppManager Commands*

* **`testssh`** - used to verify that you are able to use sshpass to ssh into the remote device
* **`publish`** - Pushes the package file to each device and unpacks it into the runtime folder
* **`publishNode`** - Pushes just the node the package file to each device and unpacks it into the runtime folder
* **`publishSwim`** - Pushes just the EDgeOS the package file to each device and unpacks it into the runtime folder
* **`all`** - Does a full package, publish and build to each device. Removes target install and build folders before publish and restarts the device once coplete
* **`startRemote`** - Stops and restarts target device. Cleans out temp files and swim db before start up.
* **`stopRemote`** - Stops target device.
* **`updateRemote`** - Pushes the full package file to each remote device, unpacks over the existing runtime, builds swim again and restarts the device$
* **`updateNode`** - Pushes the just node and python to each remote device, unpacks over the existing runtime and restarts the device$


*AppManager Examples*

* Build, package, deploy the app to raspi3, 4, 5 , and 10 followed by restarting each device

  `./bin/appManager.sh all "3 10 4 5 6" MyPassWord` 

* Like the 'all' command but does not remove existing files or build node again. Faster then running all.

  `./bin/appManager.sh updateRemote "3 10 4 5 6" MyPassWord`

* Stop and restart raspi3, 4, 5 , and 10

  `./bin/appManager.sh startRemote "3 10 4 5 6" MyPassWord`

* Start up the full app locally

  `./bin/appManager.sh start`

* Build the full app locally

  `./bin/appManager.sh build`

* Build just the swim side of the app locally

  `./bin/appManager.sh buildSwim`


## Data Bridges

This example provides a number of example data bridges which bring the the sensor data into EdgeOS. Fundamentally all the bridges simply call command lanes on the EdgeOS services to inject data into Edge.

## EdgeOS Services

The exact same bundle is deployed to every device. Consequently, every device contains the exact same EDgeOS `Service` *definitions*; varying the configuration only affects the `Service` *instantiations*. In this project, every service descibed here can also be considered a *EDgeOS Web Agent*.

1. **`SensorService`** - A physical sensor's digital twin. It exposes both the **latest** value and a **history** of values measured by its corresponding sensor. A configurable **threshold** can be defined within each `SensorService`, and deviating from it will raise an **alert**.
2. **`DeviceService`** - A device's digital twin. It exposes the **latest** value from all sensors attached to the device and contains the logic to **communicate** with any bots assigned to diagnose issues with these sensors.
3. **`BotService`** - A bot's digital twin. It directly **observes** `SensorServices` for alerts, then resolves them via **communication** with the reporting `SensorServices'` `DeviceServices`.
4. **`AggregateService`** - The greenhouse's digital twin. Data-wise, it provides **summary statistics** across all fields being observed by all devices. Operation-wise, it reports the **status** of all bots and **health** of all devices in its TestCenter.

## NodeJS Application Structure

The NodeJS server has three parts. First is a simple [Express Server](https://expressjs.com/) to serve out the various webpages which make up the web app. The second part is called 'services' and essentially act as the data bridges into EDgeOS. The third part are called 'bots' and they can act upon data derived from the various EDgeOS Lanes the EDgeOS service provides.


