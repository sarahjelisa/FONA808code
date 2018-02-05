#include "Adafruit_FONA.h"
 
#define FONA_RX 2
#define FONA_TX 3
#define FONA_RST 4
 
#include <SoftwareSerial.h>
SoftwareSerial fonaSS = SoftwareSerial(FONA_TX, FONA_RX);
SoftwareSerial *fonaSerial = &fonaSS;
 
Adafruit_FONA fona = Adafruit_FONA(FONA_RST);
 
char fonaInBuffer[64];          //for notifications from the FONA
 
void setup() {
 
  while (!Serial);
 
  Serial.begin(115200);
  Serial.println(F("FONA SMS caller ID test"));
  Serial.println(F("Initializing....(May take 3 seconds)"));
 
  // make it slow so its easy to read!
  fonaSerial->begin(4800);
  fonaSerial->println("ATZ");
  if (! fona.begin(*fonaSerial)) {
    Serial.println(F("Couldn't find FONA"));
	while(1);
  }
  Serial.println(F("FONA is OK"));
 
  // Print SIM card IMEI number.
  char imei[16] = {0}; // MUST use a 16 character buffer for IMEI!
  uint8_t imeiLen = fona.getIMEI(imei);
  if (imeiLen > 0) {
	Serial.print("SIM card IMEI:"); Serial.println(imei);
  }
 
  Serial.println("FONA Ready");
 
 
  Serial.println(F("Enabling GPS..."));
  fona.enableGPS(true);
}
 
void loop() {
  delay(2000);
 
  float latitude, longitude, speed_kph, heading, speed_mph, altitude;
 
  // if you ask for an altitude reading, getGPS will return false if there isn't a 3D fix
  boolean gps_success = fona.getGPS(&latitude, &longitude, &speed_kph, &heading, &altitude);
 
  if (gps_success) {
 
	Serial.print("GPS lat:");
	Serial.println(latitude, 6);
	Serial.print("GPS long:");
	Serial.println(longitude, 6);
	Serial.print("GPS speed KPH:");
	Serial.println(speed_kph);
	Serial.print("GPS speed MPH:");
	speed_mph = speed_kph * 0.621371192;
	Serial.println(speed_mph);
	Serial.print("GPS heading:");
	Serial.println(heading);
	Serial.print("GPS altitude:");
	Serial.println(altitude);
 
  } else {
    Serial.println("Waiting for FONA GPS 3D fix...");
  }
 
  // Check for network, then GPRS
  Serial.println(F("Checking for Cell network..."));
  if (fona.getNetworkStatus() == 1) {
	// network & GPRS? Great! Print out the GSM location to compare
	boolean gsmloc_success = fona.getGSMLoc(&latitude, &longitude);
 
	if (gsmloc_success) {
      Serial.print("GSMLoc lat:");
  	Serial.println(latitude, 6);
      Serial.print("GSMLoc long:");
      Serial.println(longitude, 6);
    } else {
  	Serial.println("GSM location failed...");
      Serial.println(F("Disabling GPRS"));
  	fona.enableGPRS(false);
      Serial.println(F("Enabling GPRS"));
  	if (!fona.enableGPRS(true)) {
    	Serial.println(F("Failed to turn GPRS on")); 
  	}
	}
  }
 
 
  char* bufPtr = fonaInBuffer;	//handy buffer pointer
 
  if (fona.available())  	//any data available from the FONA?
  {
	int slot = 0;        	//this will be the slot number of the SMS
    int charCount = 0;
	//Read the notification into fonaInBuffer
	do  {
  	*bufPtr = fona.read();
  	Serial.write(*bufPtr);
  	delay(1);
	} while ((*bufPtr++ != '\n') && (fona.available()) && (++charCount < (sizeof(fonaInBuffer)-1)));
	
	//Add a terminal NULL to the notification string
	*bufPtr = 0;
	
	//Scan the notification string for an SMS received notification.
	//  If it's an SMS message, we'll get the slot number in 'slot'
	if (1 == sscanf(fonaInBuffer, "+CMTI: \"SM\",%d", &slot)) {
  	Serial.print("slot: "); Serial.println(slot);
  	
  	char callerIDbuffer[32];  //we'll store the SMS sender number in here
  	
.
  	if (! fona.getSMSSender(slot, callerIDbuffer, 31)) {
        Serial.println("Didn't find SMS message in slot!");
  	}
      Serial.print(F("FROM: ")); Serial.println(callerIDbuffer);
 
  	char  fonamsm [160] ={'\0'};
  	snprintf (fonamsm, 160, "Hey my location is http://www.google.com/maps/place/%g,%g" , latitude, longitude);
	  
Serial.println("Sending reponse...");
  	if (!fona.sendSMS(callerIDbuffer, fonamsm)) {
        Serial.println(F("Failed"));
  	} else {
        Serial.println(F("Sent!"));
  	}
  	
if (fona.deleteSMS(slot)) {
        Serial.println(F("OK!"));
  	} else {
        Serial.println(F("Couldn't delete"));
  	}
	}
  }
}
