/*
Pulse Clock system for radio time and Arduino
*/

#include <MSF60.h>
#include <Time.h>
#include <RadioClock.h>
#include <PrintTime.h>
#include <EEPROM.h>
#include <TimeAlarms.h>

typedef union {
  byte b[4];
  unsigned int i[2];
  unsigned long ul;
} eeprom_t; // to help with wear-levelling

const int signalPin    = 2;  // pin connected to the radio module output
const boolean inverted = true; // true if module inverts the output signal
const int signalLED    = 5; // flashes when radio time signal received
const int syncedLEDPin = 6;  // lights when time is synced with radio signal
const int errorLEDPin  = 7;  // lights if there is a decoding error
const int onPin = NO_PIN;    // optional pin to turn on radio module
const int onLevel = LOW;     // the level on the onPin to turn on the module

int pulseLength = 150; // length of pulse to clock
int pulseTime = 30; // how many seconds between clock pulses
int pulseWait = 250;


// uncomment the line that matches the protocol of the connected module
//DCF77 radioClock(signalPin,inverted,signalLED,syncedLEDPin,errorLEDPin,onPin,onLevel);
MSF60 radioClock(signalPin,inverted,signalLED,syncedLEDPin,errorLEDPin,onPin,onLevel);
//WWVB radioClock(signalPin,inverted,signalLED,syncedLEDPin,errorLEDPin,onPin,onLevel);


time_t prevDisplay = 0; // time of last display
time_t prevSync=0;      // time of last sync
eeprom_t lastPulse;

int deb30sec = 0;
int deb30min = 0;

void mySetTime(time_t syncedTime)
{
  // this is called from an interrupt so don't spend too much time here
  if ((syncedTime - prevSync) < 200 || prevSync == 0) { // require two times within 2min before trusting them
    setTime(syncedTime);           // this sets the time
  }
  prevSync = syncedTime;         // save the time
}

void setup()
{
  Serial.begin(9600);

  // optional timezone offset (in hours)
  radioClock.setTimeZoneOffset(0);       // only needed if radio time not correct for your timezone.
  radioClock.setTimeCallback(mySetTime);  // give the library the function to set the time

  radioClock.start();   // start the Radio Clock
  static int oldCount = -1;
  Serial.println("Waiting for start message...");
  while(radioClock.getStatus() != status_synced)
  {
     int count = radioClock.getTickCount();
     if(count != oldCount){
         Serial.print("count = "); Serial.println(count);
         oldCount = count;
     }
     radioClock.diags();
  }

  Serial.print("Time now - ");
  printTime(prevSync);
  Serial.println();

  // extract timestamp of last pulse from eeprom to work out where the
  // physical hands are
  // The two MSBs are used as in index into other parts of EEPROM for the LSBs for wear levelling.
  // This should give 80 years EEPROM life.
  lastPulse.b[3] = EEPROM.read(0);
  lastPulse.b[2] = EEPROM.read(1);
  lastPulse.b[1] = EEPROM.read(lastPulse.i[1] % 107 + 2);
  lastPulse.b[0] = EEPROM.read(lastPulse.i[1] % 914 + 107 + 2);
  Serial.print("Raw EEPROM ");
  Serial.print(lastPulse.ul, BIN);
  Serial.print(" - ");
  printTime(lastPulse.ul);
  Serial.println();
  prevSync = now(); // re-used variable to get a static value
  // Copy the 12-hour time from the EEPROM and the Date am/pm from now()
  lastPulse.ul = (lastPulse.ul % 43200UL) + (prevSync - (prevSync %43200UL));
  if ((lastPulse.ul - prevSync) > 600){ // it is quicker to wait if the "face time" is 10 mins fast
    lastPulse.ul -= 43200UL;
  }
  Serial.print("12-hour delta ");
  Serial.print(lastPulse.ul, BIN);
  Serial.print(" - ");
  printTime(lastPulse.ul);
  Serial.println();
  pinMode(3, OUTPUT); // SSR Control
  pinMode(5, OUTPUT); // The Other LED
  pinMode(11, INPUT_PULLUP); // 30 second advance
  pinMode(12, INPUT_PULLUP); // 30 minute advance
}

void loop()
{
  // check if time changed and if time has been syncrhonized at least once
  time_t currentTime = now();

  if (!digitalRead(11)) { // +30 seconds button
    deb30sec += 1;
    if (deb30sec == 2){
      digitalWrite(13, 1);
      delay(100);
      digitalWrite(13, 0);
      lastPulse.ul -= pulseTime;
      deb30sec = 0;
    }
  } else {
    deb30sec -= 1;
  }
  if (deb30sec > 4) deb30sec = 4;
  if (deb30sec < 0) deb30sec = 0;

  if (!digitalRead(12)) { // + 30 minutes button
    deb30min += 1;
    if (deb30min == 2){
      digitalWrite(13, 1);
      delay(500);
      digitalWrite(13, 0);
      lastPulse.ul -= 60 * pulseTime;
      deb30min = 0;
    }
  } else {
    deb30min -= 1;
  }

  if (deb30min > 4) deb30min = 4;
  if (deb30min < 0) deb30min = 0;

  if (currentTime > lastPulse.ul) {
    digitalWrite(3, 1);
    delay(pulseLength);
    digitalWrite(3, 0);
    lastPulse.ul += pulseTime;
    EEPROM.write(0, lastPulse.b[3]);
    EEPROM.write(1, lastPulse.b[2]);
    EEPROM.write(lastPulse.i[1] % 107 + 2, lastPulse.b[1]);
    EEPROM.write(lastPulse.i[1] % 915 + 107 + 2, lastPulse.b[0]);
    delay(pulseWait);
  }
  delay(100);
}
