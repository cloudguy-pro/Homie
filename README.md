# Homie – A language understanding bot for smart homes

If you want to build your own bot running on Azure to manage your smart home, then follow the following tutorial to get started with a bot which is able to turn your lights on and off.

If you want to know more about Homie's story then please take a look at http://www.cloudguy.pro/homie-a-language-understanding-bot-for-smart-homes/

## Solution overview

<p align="center"> 
<img src="images/homie_overview2.0.png" />
</p>

### Azure Bot Creation
First of all we need to create an Azure Bot service, which is responsible for communicating with the user. So head over to the Azure portal (https://portal.azure.com) and create a new bot.

In the Azure portal click on "+" and search for bot service. You can either go with a "Web App Bot" or with a "Functions Bot". In this tutorial we will use a Web App bot as this is a great way to provide a stable bot platform without the need of dealing with backend services and other stuff so you will be able to develop and deploy your bot faster.

<p align="center"> 
<img src="images/bot_creation1.png"/>
</p>

Now we need to fill out some basic properties to give the bot a name and chosse the location as well as a pricing tier (F0 should be sufficient for developing and testing as you can upgrade when you want to deploy the bot later on). For the Bot template make sure to choose "Node.js" and then "Language understanding" as this brings a lot of preconfigured attributes within our bot solution as well as predefined code snippets which allow us to get started with the LUIS Cognitive Services API quicker. Be sure to set your App service plan and location correctly and give your Azure Storage a meaningful name.

<p align="center"> 
<img src="images/bot_creation2.png"/>
</p>

After your bot has been created you can instantly test it when going to your bot within the Azure portal and select "Test in Web Chat" where you can type in a message to see if your bot is working. Btw you see that your bot already recognizes some basic intents like greetings as these are built in intents into your newly created LUIS app which we'll see later on.

<p align="center"> 
<img src="images/bot_creation3.png"/>
</p>

Now we will need to adapt our bot's code to make it smarter. Therefore, the Azure portal offers a pretty nice feature as you can edit your code with the code editor built right into the portal. So head over to "Build" and the choose "Open online code editor".

<p align="center"> 
<img src="images/bot_creation4.png"/>
</p>

Now that you opened up the App Service Editor you can select your app.js file and start developing your bot's app logic.

<p align="center"> 
<img src="images/bot_creation5.png"/>
</p>

We can already add some of our Node modules and install the dependencies here, just click on "Open console" in your App Service Editor and install the following three Node modules as we need them later on in our code:

```{r, engine='bash'}
npm install azure-storage
npm install azure-iothub
npm install azure-iot-common
```

But before we can do that, we need to setup our other services like LUIS or our Azure IoT Hub before we can edit our code...

### LUIS
Let's start with the Cognitive Services API LUIS which is responsible for enabling our bot to detect sentiments of the users' input messages. Browse to https://www.luis.ai/applications login and select the App which has the same name as your bot to start adding entities and intents. The good thing is that there is already a prebuilt domain for HomeAutomation available which can be added to our LUIS app which saves a lot of time and effort. So click on "Prebuilt Domains" and add the "HomeAutomation" domain to our app:

<p align="center"> 
<img src="images/luis_creation1.png"/>
</p>

Now we need to train our LUIS app and publish it right from the LUIS portal (you might need to add a key here - choose your Azure subscription and the corresponding key. If you have no keys available go back to the Azure portal and create a new Cognitive Service API of type LUIS as instructed here: https://docs.microsoft.com/en-us/azure/cognitive-services/luis/azureibizasubscription).

Now the bot is able to detect the sentiment for turning on or off the lights or other devices in our smart home. Now we need to create the next Azure service for enabling device to cloud communication...

### Azure IoT Hub
Now head back to the Azure portal and create a new service the IoT Hub with the following attributes (Note: you can go with the F1 pricing tier for testing but you should choose at least S1 for production as you cannot switch between the free and paid tiers later on):

<p align="center"> 
<img src="images/iothub_creation1.png"/>
</p>

After the IoT Hub has been provisioned you need to add your devices to it. Go to "IoT Devices" and add the devices you want to communicate with. You only need to set the Device ID and make note of the keys and the connection string after creation as we will need that later on:

<p align="center"> 
<img src="images/iothub_creation2.png"/>
</p>

Now as we have set up all our Azure services we need to go down to our devices to make them IoT enabled...

### Raspberry PI 3

So in my case I have bought the following 2 pieces from Amazon (you can find the links below) as these are pretty cheap and have everything we need to setup the desired solution. Note: I have not implemented a real light bulb in this scenario as this is not that handy for demo cases but you are good to go to connect an IoT enabled light bulb to your Pi to turn that on and off as well (should not be too much of a code change on the Pi).

<p align="center"> 
<img src="images/raspberry_setup1.png"/>
</p>

**Vilros Rasperry Pi Ultimate Starter Kit 3**
https://www.amazon.de/gp/product/B01CYQJP9O/ref=oh_aui_detailpage_o01_s00?ie=UTF8&psc=1

**kwmobile 8x32 LED Matrix Module for Raspberry Pi and Arduino**
https://www.amazon.de/gp/product/B06XJ9ZX17/ref=oh_aui_detailpage_o03_s00?ie=UTF8&psc=1

So now when you get your Pi which you orderd you should already have the Raspbian image pre-installed on your SD card so you are good to go to start setting it up. I won't go into detail on how to prepare your Pi here as there are lots of blog posts which describe how to setup wifi and other related stuff like Node.Js which should usually be installed already (follow e.g.: https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md for instructions with the wifi setup). When you have setup your device we need to first of all connect our device with the LED matrix. So plugin the led matrix cable into your Pi's GPIO pinout as follows:

Name | Remarks | Raspberry Pi GPIO Pin | Function
------------ | ------------- | ------------- | -------------
VCC | +5V Power | 2 | 5V0
GND | Ground | 6 | GND
DIN | Data In | 19 | GPIO 10 (MOSI)
CS | Chip Select | 24 | GPIO 8 (SPI CS0)
CLK | Clock | 23 | GPIO 11 (SPI CLK)

Thanks to https://tutorials-raspberrypi.de/led-dot-matrix-zusammenbau-und-installation/ for this GPIO pinout description.

Now it's time for us to actually write some code on the Pi which allows us to connect the Pi with our IoT Hub and exchange messages with it. So take the following Node.JS code and create a new .js file on your Pi (I have created a "Homie-Device01.js" file as this is easy to remember the names when you have multiple devices or usecases):

```javascript
'use strict';

var clientFromConnectionString = require('azure-iot-device-mqtt').clientFromConnectionString;
var Message = require('azure-iot-device').Message;
var connectionString = [INSERT YOUR IOT HUB CONNECTION STRING HERE];
var client = clientFromConnectionString(connectionString);

var max7219LedMatrix = require('node-max7219-led-matrix');
var max7219 = new max7219LedMatrix.max7219("/dev/spidev0.0");

// GPIO pin of the led
var configPin = 7;
// Blinking interval in usec
var configTimeout = 1000;

var isLedOn = 0;

function printResultFor(op) {
  return function printResult(err, res) {
    if (err) console.log(op + ' error: ' + err.toString());
  };
}

var connectCallback = function (err) {
  if (err) {
    console.log('Could not connect: ' + err);
  } else {
    console.log('Client connected');
    client.on('message', function (msg) {
        if(msg.data == "lightsOnOffice"){
			isLedOn = +!isLedOn;
			max7219.showMessage('O');
			max7219.clear();
			console.log('\x1b[33m%s\x1b[0m', msg.data + " - turning the lights on in the office...");
        } else if(msg.data == "lightsOffOffice"){
			isLedOn = 0;
			max7219.showMessage(' ');
			max7219.clear();
			max7219.showMessage(' ');
			max7219.clear();           
			console.log('\x1b[31m%s\x1b[0m', msg.data + " - turning the lights off in the office...");
        } else if(msg.data == "lightsOnLivingRoom"){
			isLedOn = +!isLedOn;
			max7219.showMessage('O');
			max7219.clear();
			console.log('\x1b[33m%s\x1b[0m', msg.data + " - turning the lights on in the living room...");
        } else if(msg.data == "lightsOffLivingRoom"){
			isLedOn = 0;
			max7219.showMessage(' ');
			max7219.clear();
			max7219.showMessage(' ');
			max7219.clear();
			console.log('\x1b[31m%s\x1b[0m', msg.data + " - turning the lights off in the living room...");
        }
      client.complete(msg, printResultFor('completed'));
    });
  }
};

client.open(connectCallback);
```

**Note: Please add your IoT Hub connection string in the very beginning of the file to make it connect to the Iot Hub successfully**

After you have saved the file we need to install our Node modules with the following command from a terminal window in the path where you created the Node.JS file:

```{r, engine='bash'}
npm install azure-iot-device-mqtt --save
npm install azure-iot-device --save
npm install node-max7219-led-matrix --save
```

Now it's time to test the connection between your device and Azure, so open up a terminal on your Pi and execute the following command

```{r, engine='bash'}
sudo node [Path to your Node.JS file]/Homie-Device01.js
```

If you have done everything correctly you should see this screen:

<p align="center"> 
<img src="images/raspberry_setup2.png"/>
</p>

### Azure Bot Development

Now when everything is ready we can start adding some code to our bot. We actually need to replace the default code within the App Service Editor of the **app.js** file with the following code

```javascript
var restify = require('restify');
var builder = require('botbuilder');
var botbuilder_azure = require("botbuilder-azure");

// Setup Restify Server
var server = restify.createServer();
server.listen(process.env.port || process.env.PORT || 3978, function () {
   console.log('%s listening to %s', server.name, server.url); 
});

// Setup IoT Stuff
var Client = require('azure-iothub').Client;
var Message = require('azure-iot-common').Message;
var connectionString = [INSERT YOUR IOT HUB CONNECTION STRING HERE];
var targetDevice = 'Homie-Device01';
var serviceClient = Client.fromConnectionString(connectionString);
  
// Create chat connector for communicating with the Bot Framework Service
var connector = new builder.ChatConnector({
    appId: process.env.MicrosoftAppId,
    appPassword: process.env.MicrosoftAppPassword,
    openIdMetadata: process.env.BotOpenIdMetadata 
});

// Listen for messages from users 
server.post('/api/messages', connector.listen());

/*----------------------------------------------------------------------------------------
* Bot Storage: This is a great spot to register the private state storage for your bot. 
* We provide adapters for Azure Table, CosmosDb, SQL Azure, or you can implement your own!
* For samples and documentation, see: https://github.com/Microsoft/BotBuilder-Azure
* ---------------------------------------------------------------------------------------- */

var tableName = 'botdata';
var azureTableClient = new botbuilder_azure.AzureTableClient(tableName, process.env['AzureWebJobsStorage']);
var tableStorage = new botbuilder_azure.AzureBotStorage({ gzipData: false }, azureTableClient);

// Create your bot with a function to receive messages from the user
var bot = new builder.UniversalBot(connector);
bot.set('storage', tableStorage);

// Make sure you add code to validate these fields
var luisAppId = process.env.LuisAppId;
var luisAPIKey = process.env.LuisAPIKey;
var luisAPIHostName = process.env.LuisAPIHostName || 'westus.api.cognitive.microsoft.com';

const LuisModelUrl = 'https://' + luisAPIHostName + '/luis/v1/application?id=' + luisAppId + '&subscription-key=' + luisAPIKey;

// Main dialog with LUIS
var recognizer = new builder.LuisRecognizer(LuisModelUrl);
var intents = new builder.IntentDialog({ recognizers: [recognizer] })
.matches('Greeting', (session) => {
    session.send('You reached Greeting intent, you said \'%s\'.', session.message.text);
})
.matches('Help', (session) => {
    session.send('You reached Help intent, you said \'%s\'.', session.message.text);
})
.matches('Cancel', (session) => {
    session.send('You reached Cancel intent, you said \'%s\'.', session.message.text);
})

// Intent for turning on the lights
.matches('HomeAutomation.TurnOn', (session, args) => {
    session.sendTyping();
    if(args.entities[0]) {
        var deviceEntity = builder.EntityRecognizer.findEntity(args.entities, 'HomeAutomation.Device');
        var roomEntity = builder.EntityRecognizer.findEntity(args.entities, 'HomeAutomation.Room');
        if (roomEntity){
            session.send("Ok, I will turn the %s on in the %s", deviceEntity['entity'], roomEntity['entity']);
            session.endDialog();
			if (deviceEntity == "lights" && roomEntity == "office"){
				var myCommand = "lightsOnOffice";
				var message = new Message(myCommand);
				message.ack = 'full';
				message.messageId = "My Message ID";
				console.log('Sending message: ' + message.getData());
				serviceClient.send(targetDevice, message);
			}
        }
        
    }
})
/*
.matches('<yourIntent>')... See details at http://docs.botframework.com/builder/node/guides/understanding-natural-language/
*/
.onDefault((session) => {
    session.send('Sorry, I did not understand \'%s\'.', session.message.text);
});

bot.dialog('/', intents);    
```

**Note: Make sure to insert your Iot Hub connection string and change the device name if you chose another name for your Iot device!**

No when you run the Node.JS file on your PI and type in the message **"Turn the lights on in the office"** into your bot's emulator (just click on "Test in Web Chat" in the Azure portal on your bot's site) you should see the following on your device:

<p align="center"> 
<img src="images/raspberry_setup3.png"/>
</p>

And the bot should send you the following message back in your emulator:

<p align="center"> 
<img src="images/bot_creation6.png"/>
</p>
