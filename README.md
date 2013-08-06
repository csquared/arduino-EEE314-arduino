# Arduino EEE314 - Basic Networking


The Arduino Basic Networking class builds on a student's experience in the earlier
Arduino classes and focuses on connecting an Arduino to a web application and
controlling it from that app using an Ethernet Shield.

In this class, you will write an HTTP server application in Ruby or Javascript
and get your Arduino to communicate with the server by writing simple HTTP GET
and POST requests from the Arduino.
We’ll start with a simple non-networked program on the Arduino.
In the next step, we’ll add the ability to control an LED on the Arduino from the server.
After that we’ll send data from the Arduino to the server.
When the application is working on our local network, we'll deploy it to the web using Heroku's free tier.

Students that have completed the Displays or Outputs course can focus on controlling displays
and outputs via the server-side application.
Students that have completed the inputs class will learn how to send sensor data
from the Arduino to the web app.
Students with familiarity with web programming will enjoy learning how to leverage
HTTP as a communication mechanism for their Arduino projects.

Prerequisites:  You should be able to program on the Arduino, be familiar with Ruby or Javascript,
and be comfortable using the command line.
Arduino Level 1 and 2 classes are recommended, as is one of the Displays, Inputs, or Outputs
courses (if not all three).
We’ll be using Heroku to deploy our web applications,
so installing the [Heroku Toolbelt](https://toolbelt.heroku.com/)
ahead of time is recommended.

## Step 1 - Basic Button

For our first step in the demo we'll wire up a simple button circuit.

![Button Wiring](https://raw.github.com/csquared/arduino-EEE314-arduino/master/button_w_led.png)

We'll then take this circuit and write an arduino program where we use the button
to turn the led on an off.

In my code example, I have the LED on pin 7 and the button input going to pin 2.
We read the value of the current on pin 2 with the call to `digitalRead(buttonPin)`.
If that is `HIGH`, then we turn the led pin on.  If not, we turn it off.

```c++
// constants won't change. They're used here to
// set pin numbers:
const int buttonPin = 2;     // the number of the pushbutton pin
const int ledPin    = 7;     // the number of the LED pin

// variables will change:
int buttonState = 0;         // variable for reading the pushbutton status

void setup() {
  // initialize the LED pin as an output:
  pinMode(ledPin, OUTPUT);
  // initialize the pushbutton pin as an input:
  pinMode(buttonPin, INPUT);
  //Get Serial going
  Serial.begin(9600);
}

void loop(){
  // read the state of the pushbutton value:
  buttonState = digitalRead(buttonPin);
  Serial.print("Button State ");
  Serial.println(buttonState);

  // check if the pushbutton is pressed.
  // if it is, the buttonState is HIGH:
  if (buttonState == HIGH) {
    // turn LED on:
    digitalWrite(ledPin, HIGH);
  }
  else {
    // turn LED off:
    digitalWrite(ledPin, LOW);
  }
}
```

## Step 2 - HTTP GET data from the server

In this step we're going to ignore the button setting and read if the LED should be on
or off from the server.

### Step 2a - connecting the ethernet shield to the BBB board.

The Ethernet Shield is not specifically designed to work with the BBB board, but
we can get around that.
The easiest way to connect the Ethernet Shield is to directly connect it to the
SPI headers on the BBB board.  You can than jumper the pins into the top of
the board.

The SPI headers are the group of 6 pins next to the power port on the BBB board.
You can just place the matching connector on the bottom of the Ethernet Shield onto
this connector.  You can then jumper pins A0, A1, 10, 11, 12, and 13 into the
Shield itself.

![Ethernet Shield](https://raw.github.com/csquared/arduino-EEE314-arduino/master/ethernet_shield.JPG)

Now that we have the shield wired up, we can connect to the network.

I've [written an arduino library](https://github.com/csquared/arduino-restclient) that we can use
to setup the ethernet card and make HTTP requests.

Install it on mac with the following commands:

    > cd ~/Documents/Arduino
    > mkdir libraries
    > cd libraries
    > git clone https://github.com/csquared/arduino-restclient.git RestClient

Or you can click the Download link and unzip it into `Documents/Arduino/libraries/RestClient`.
Once it is there you'll need to restart the Arduino IDE so it recognizes the new library.

Open up File->Examples->RestClient->simple_GET and confirm you have the library installed
and can make GET requests to our test API.

### Step 3 - Create the LED API

Now we will create a web app to serve as an API for our LED.  It should follow the following
specification:

#### GET /
Returns an HTML page with the current LED status.

#### GET /led
Returns a response body with the current LED status.

#### POST /led
Set the led status to the request body.

Test the `GET /` endpoint by visiting `http://localhost:5000` in your browser.

We should be able to interact with the API using the command line client `curl`:

  > curl http://localhost:5000/led
  > on

To send a post request with body (the string `off`):

  > curl -X POST --data off http://localhost:5000/led
  > OK

If that worked, we shoud get `off` as the new LED status.

  > curl http://localhost:5000/led
  > off


#### Finished Examples

- [example server using Ruby and Sinatra](https://github.com/csquared/arduino-EEE314-ruby)
- [example server using Node.js and express](https://github.com/csquared/arduino-EEE314-node)

### Step 3a - Run it locally

If you have your arduino connected directly to your computer, you can run your service locally.
Assuming an IP address of `192.168.1.1` and the web service running on port 5000 you can connect via:

```c++
RestClient client = RestClient("192.168.1.1",5000);
```

### Step 3b - Deploy to the web

> heroku create my-app-name
> git push heroku

```c++
RestClient client = RestClient("my-app-name.herouapp.com");
```

## Step 4 - GET the button state

With our server in place, we can now connect from the Arduino.

We'll use the `get` method of the `RestClient` class to pull the button state from the
server.

You can decide on the protocol.  For this example, a response of `on` means turn the button on
and a response of anything else will turn the button off.

```c++

#include <SPI.h>
#include <Ethernet.h>
#include "RestClient.h"

// constants won't change. They're used here to
// set pin numbers:
const int buttonPin = 2;     // the number of the pushbutton pin
const int ledPin    = 7;     // the number of the LED pin
RestClient client = RestClient("10.0.1.47",5000);

// variables will change:
int buttonState = 0;         // variable for reading the pushbutton status

void setup() {
  // initialize the LED pin as an output:
  pinMode(ledPin, OUTPUT);
  // initialize the pushbutton pin as an input:
  pinMode(buttonPin, INPUT);
  //Get Serial going
  Serial.begin(9600);

  // start the Ethernet connection:
  client.dhcp();

  // print your local IP address:
  Serial.print("My IP address: ");
  Serial.println(Ethernet.localIP());
  Serial.println();
}


String response;

void loop(){
  response = "";
  // read the LED state from the web
  Serial.println("connect HTTP");
  if(client.get("/led", &response) == 0){
    Serial.println("connected");
    Serial.println(response);
    Serial.println(response.indexOf("on"));
    if(response.indexOf("on") > -1){
      Serial.println("Turn LED on");
      digitalWrite(ledPin, HIGH);
    }else{
      Serial.println("Turn LED off");
      digitalWrite(ledPin, LOW);
    }
  }
  delay(1000);

  // read the state of the pushbutton value:
  buttonState = digitalRead(buttonPin);
  Serial.print("Button State ");
  Serial.println(buttonState);
}
```

Once we are reading the button state from the web app, we can use the same curl commands
from above to change the state of the LED.

## Step 5 - POST the button state

Now let's bring the button back into it.  Instead of calling `digitalWrite` when the
button changes, we'll POST the state to the webapp and then let it change when
the newest GET request returns the state we just sent in.

We use the `post` method of the `RestClient` object to send the HTTP POST request
when the button changes state.

```c++

#include <SPI.h>
#include <Ethernet.h>
#include "RestClient.h"


// constants won't change. They're used here to
// set pin numbers:
const int buttonPin = 2;     // the number of the pushbutton pin
const int ledPin    = 7;     // the number of the LED pin
RestClient client = RestClient("10.0.1.47",5000);

// variables will change:
int buttonState = 0;         // variable for reading the pushbutton status
char* ledState[4];
boolean led_on = false;

void setup() {
  // initialize the LED pin as an output:
  pinMode(ledPin, OUTPUT);
  // initialize the pushbutton pin as an input:
  pinMode(buttonPin, INPUT);
  //Get Serial going
  Serial.begin(9600);

  // start the Ethernet connection:
  client.dhcp();

  // print your local IP address:
  Serial.print("My IP address: ");
  Serial.println(Ethernet.localIP());
  Serial.println();
}

String response;

void loop(){
  response = "";
  // read the LED state from the web
  Serial.println("connect HTTP");
  if(client.get("/led", &response) == 0){
    Serial.println("connected");
    Serial.println(response);
    Serial.println(response.indexOf("on"));
    if(response.indexOf("on") > -1){
      Serial.println("Turn LED on");
      digitalWrite(ledPin, HIGH);
      led_on = true;
    }else{
      Serial.println("Turn LED off");
      digitalWrite(ledPin, LOW);
    }
  }
  delay(250);

  // read the state of the pushbutton value:
  buttonState = digitalRead(buttonPin);
  Serial.print("Button State ");
  Serial.println(buttonState);

  // check if the pushbutton is pressed.
  // if it is, the buttonState is HIGH:
  if (buttonState == HIGH) {
    if(led_on){
      client.post("/led", "off");
      led_on = false;
    }else{
      client.post("/led", "on");
    }
    Serial.println("connect HTTP");
    delay(250);
  }
}
```


