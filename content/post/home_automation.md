+++
categories = []
date = "2015-04-29T19:57:12+02:00"
description = ""
keywords = []
title = "home_automation"

+++

IOT is gaining traction these days, and I have had some spare time lately to do some automation at home. I wanted to control heating, and ventilation without doing anything with the mains.

There are many remote switches today that works with RF 433.92. I got some from a Swedish brand called NEXA. I got 5 of those, 2 rated at 2300W and some cheaper rated 1500W. These have a self learning system that allows for mixing and matching remotes.

To control these I have an arduino with a cheap rf receiver and rf transmitter. The receiver is used for sniffing the remote id, with this code from [arduino.cc](http://playground.arduino.cc/Code/HomeEasy). 
``` c
/*
 * AM-HRR3 receiver test
 *
 * Homeeasy protocol receiver for the new protocol.
 *
 * This protocol isn't documented so well, so here goes (this is largely from memory).
 *
 * The data is encoded on the wire (aerial) as a Manchester code.
 *
 * A latch of 275us high, 2675us low is sent before the data.
 * There is a gap of 10ms between each message.
 *
 * 0 = holding the line high for 275us then low for 275us.
 * 1 = holding the line high for 275us then low for 1225us.
 * 
 * The timings seem to vary quite noticably between devices.  HE300 devices have a
 * low for about 1300us for a 1 whereas HE303 devices seem to have a low of about
 * 1100us.  If this script does not detect your signals try relaxing the timing
 * conditions.
 * 
 * Each actual bit of data is encoded as two bits on the wire as:
 * Data 0 = Wire 01
 * Data 1 = Wire 10
 *
 * The actual message is 32 bits of data (64 wire bits):
 * bits 0-25: the group code - a 26bit number assigned to controllers.
 * bit 26: group flag
 * bit 27: on/off flag
 * bits 28-31: the device code - a 4bit number.
 *
 * The group flag just seems to be a separate set of addresses you can program devices
 * to and doesn't trigger the dim cycle when sending two ONs.
 *
 * There's extra support for setting devices to a specific dim-level that you get
 * with the HE100 ultimate remote control. I think this involves sending a Wire 11 at
 * the on/off flag position and then the message is longer with another 4 bits for dim
 * level.
 * Need to look into this.
 *
 * Barnaby Gray 12/2008
 * Peter Mead   09/2009
 */

int rxPin = 12;


void setup()
{	pinMode(rxPin, INPUT);
	Serial.begin(9600);
}


void loop()
{
	int i = 0;
	unsigned long t = 0;

	byte prevBit = 0;
	byte bit = 0;

	unsigned long sender = 0;
	bool group = false;
	bool on = false;
	unsigned int recipient = 0;

	// latch 1
	while((t < 9480 || t > 10350))
	{	t = pulseIn(rxPin, LOW, 1000000);
	}

	// latch 2
	while(t < 2550 || t > 2700)
	{	t = pulseIn(rxPin, LOW, 1000000);
	}

	// data
	while(i < 64)
	{
		t = pulseIn(rxPin, LOW, 1000000);

		if(t > 200 && t < 365)
		{	bit = 0;
		}
		else if(t > 1000 && t < 1360)
		{	bit = 1;
		}
		else
		{	i = 0;
			break;
		}

		if(i % 2 == 1)
		{
			if((prevBit ^ bit) == 0)
			{	// must be either 01 or 10, cannot be 00 or 11
				i = 0;
				break;
			}

			if(i < 53)
			{	// first 26 data bits
				sender <<= 1;
				sender |= prevBit;
			}	
			else if(i == 53)
			{	// 26th data bit
				group = prevBit;
			}
			else if(i == 55)
			{	// 27th data bit
				on = prevBit;
			}
			else
			{	// last 4 data bits
				recipient <<= 1;
				recipient |= prevBit;
			}
		}

		prevBit = bit;
		++i;
	}

	// interpret message
	if(i > 0)
	{	printResult(sender, group, on, recipient);
	}
}


void printResult(unsigned long sender, bool group, bool on, unsigned int recipient)
{
	Serial.print("sender ");
	Serial.println(sender);

	if(group)
	{	Serial.println("group command");
	}
	else
	{	Serial.println("no group");
	}

	if(on)
	{	Serial.println("on");
	}
	else
	{	Serial.println("off");
	}

	Serial.print("recipient ");
	Serial.println(recipient);

	Serial.println();
}
```
Then with an arduino the rf transmitter and an ethernet shield:
```c
#include <SPI.h>
#include <Ethernet.h>
#include <NexaTransmitter.h>
#include <avr/wdt.h>

NexaTransmitter remote(7,  15551665); // Create the nexa remote on pin7 with remote id, remote id can be found using the script mentioned earlier

// assign a MAC address for the ethernet controller.
// fill in your address here:
byte mac[] = {
  0x00, 0xAA, 0xBB, 0xCC, 0xDE, 0x02
};
// initialize the library instance:
EthernetClient client;

char server[] = "pi.sipp.io";

unsigned long lastConnectionTime = 0;             // last time you connected to the server, in milliseconds
const unsigned long postingInterval = 10L * 1000L; // delay between updates, in milliseconds
// the "L" is needed to use long type numbers
String readString;
int x = 0;
char lf = 10;
void setup() {
  // start serial port:
  Serial.begin(9600);
  while (!Serial) {
    ; // wait for serial port to connect. Needed for Leonardo only
  }

  // give the ethernet module time to boot up:
  delay(1000);
  // start the Ethernet connection using a fixed IP address and DNS server:
  if (Ethernet.begin(mac) == 0) {
    Serial.println("Failed to configure Ethernet using DHCP");
    // no point in carrying on, so do nothing forevermore:
    for (;;)
      ;
  }
  // print the Ethernet board/shield's IP address:
  Serial.print("My IP address: ");
  Serial.println(Ethernet.localIP());
  wdt_enable(WDTO_8S);
}

//Finding two crlf which separates the body from the header in the html response
String getBody() {
  readString = "";
  char c;
  char d;
  char e;
  char f;
  while (client.available()) {
    f = e;
    e = d;
    d = c;
    c = client.read();
    if (c == 10 && d == 13 && e == 10 && f == 13  ) {
      while (client.available()) {
        readString += (char)client.read();
      }
    }
  }
  return readString;
}

void loop() {
  wdt_reset();
  String body = getBody();
  if (body == "on temp") {
    remote.setSwitch(true, 0);
  }
  
  if (body == "off temp") {
    remote.setSwitch(false, 0);
  } 
  
  if (body == "on air") {
    remote.setSwitch(true, 1);
  }
  
  if (body == "off air") {
    remote.setSwitch(false, 1);
  } 

  if (millis() - lastConnectionTime > postingInterval) {
    httpRequest("GET /heater/ HTTP/1.1");
  }

}

// this method makes a HTTP connection to the server:
void httpRequest(String url) {
  // close any connection before send a new request.
  // This will free the socket on the WiFi shield
  client.stop();

  // if there's a successful connection:
  if (client.connect(server, 80)) {
  //  Serial.println("connecting...");
    // send the HTTP PUT request:
    client.println(url);
    client.println("Host: sipp.io");
    client.println("User-Agent: arduino-ethernet");
    client.println("Connection: close");
    client.println();

    // note the time that the connection was made:
    lastConnectionTime = millis();
  }
  else {
    // if you couldn't make a connection:
   // Serial.println("connection failed");
  }
}
```

And on the server the arduino is pulling from I have a go server, which reads my Netatmo station and changes the values according to the given parameters to ensure a healthy indoor climate.