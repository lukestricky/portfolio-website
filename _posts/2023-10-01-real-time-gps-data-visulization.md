---
title: "Real Time GPS Data Visulization"
date: 2023-10-01
categories: [Category1]
tags: [GPS, Flutter, Arduino]
author: Luke
---

# Inspiration

I work as a sailing race coach in the summers, and I wanted to have some tangible data to reference when giving my athletes feedback on the water. I was having trouble finding an affordable solution to this, so I decided to make one, because that is what I’m going to school for this right?

# Introduction

Before staring development, I had to choose what data I wanted to measure as, and how I wanted to display it to the user. I decided to measure GPS speed over ground (SOG), GPS heading, along with accelerometer data. This would give the coach a good view of acceleration and boat rotation when plotted on a line graph over time. To display the data to the user I decided to user flutter as communication over WIFI seemed easy to do. Flutter also allowed me collect data, index it, and display it to the user dynamically.

# Getting GPS data

The goal with the GPS was to receive SOG, time, and heading data at approximately 4Hz. I decided on 4Hz as I planned to use TCP communication for this project and from experience that will run around 4Hz from an Arduino.

The GPS I chose to use was the BN-220. It has the capability to send 7 messages through its TX pin which contains various types and forms of information. Consulting the data sheet, I found that I only needed the information from the RMC message for my application. To increase the speed of the transmission to the microcontroller I disabled the other 6 messages.

The disabling of messages and changing update speed of the GPS must be done on every start up as the GPS uses volatile memory onboard. This is done by sending messages to the GPS’s on start up. I used a native function in the NMEAGPS library to disable the messages and wrote a function to change the frequency by sending data using the UBX protocol.

```c++
void sendUBX( const unsigned char *progmemBytes, size_t len )
{
  gpsPort.write( 0xB5 ); // SYNC1
  gpsPort.write( 0x62 ); // SYNC2

  uint8_t a = 0, b = 0;
  while (len-- > 0) {
    uint8_t c = pgm_read_byte( progmemBytes++ );
    a += c;
    b += a;
    gpsPort.write( c );
  }
  gpsPort.write( a ); // CHECKSUM A
  gpsPort.write( b ); // CHECKSUM B

}
```

Below is the startup script to disable all unused messages and change the refresh rate to 4Hz. Note that all messages must be disabled before the frequency of the GPS can be changed.

```c++
// Initiates GPS
  gpsPort.begin(9600);
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableRMC );
  delay( 250 );
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableGLL );
  delay( 250 );
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableGSV );
  delay( 250 );
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableGSA );
  delay( 250 );
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableGGA );
  delay( 250 );
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableVTG );
  delay( 250 );
  gps.send_P( &gpsPort, (const __FlashStringHelper *) disableZDA );
  delay( 500 );
  sendUBX( ubxRate4Hz, sizeof(ubxRate4Hz) );
  delay( 200 );
  //  Then re-enable the sentences that provide the pieces you want.
  gps.send_P( &gpsPort, (const __FlashStringHelper *) enableRMC );
  delay( 250 );

```

# Getting accelerometer data

# Communication data with flutter app

# Flutter

# Demo

# Further Development
