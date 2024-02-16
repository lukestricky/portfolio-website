---
title: "Real Time GPS Data Visulization"
date: 2023-10-01
categories: []
tags: [GPS, Flutter, Arduino]
author: Luke
---

# Introduction

I work as a sailing race coach in the summers, and I wanted to have some tangible data to reference when giving my athletes feedback on the water. This includes measurments such as speed, heading, acceleration, and heeling. _Heeling is a sailing term used to describe when the boat is tipping or leaning to one side_. If a sailing coach was able to record these data points and then convey them to a sailor in real time, it would be extremely beneficial to the sailor’s progression and improvement. This project addresses the need for a sailboat-specific GPS tracking solution that is cost-effective, sustainable, adaptable, and can provide some necessary information to coaches.

# Embedded System

The embedded system consists of a central microcontroller connected to two sensors and a power source. This is the extent of the hardware needed for the application of this project. There were multiple iterations of the physical system first through breadboard and finally implemented in a vector board prototype.

## MCU

The microcontroller use in the final design was an Arduino Rev 2 Wi-Fi, this allows for network communication and has enough processing power to handle all the sensor information at the speed that is required. The Arduino was powered by recycled 18650 cells, these cells output 3.7 V each, two cells connected in series gives 7.4 V which is in the 7 – 12 V voltage range of the Arduino’s voltage in pin.

## Wi-Fi Access point

The network communication was one of the most crucial parts used to send data to the frontend. It was established using the WiFiNINA library provided on Arduino’s website. The beginAP function was used to open an access point that could be accessed by anything with Wi-Fi capabilities, in this case, the frontend flutter app.

```c++
//add code
```

## GPS

The first sensor that was necessary was the GPS module to receive speed, direction, and time data at 4 Hz. This refresh rate was chosen as this project used TCP communication which reliably runs at 4 Hz off Arduino access point Wi-Fi. The GPS module chosen for this application was the BN-220 module, it connects to the Arduino through UART protocol. My GPS of choice the BN-220 is rated for a refresh rate of 1-18 Hz and has the capability to send seven unique GPS messages to the Arduino. The only message that was necessary to receive the relevant information was the RCM message. To increase speed and reliability the other six messages were disabled on start up. The refresh rate of the GPS was changed by first disabling all seven messages from the GPS, waiting for a short period of time, and then sending a message to the GPS to enable 4Hz. After the refresh rate has been updated the relevenant message can be enabled by sending yet another message to the GPS. Updating the refresh rate and disabling messages must be done every time the GPS was turned on as it used volatile memory.

After setting up the GPS, the data from the RCM message mentioned was read through the RX pin on the Arduino. The NMEAGPS Arduino library was used to easily parse the message into each separate values, and to determine when the GPS was updating. Each time the GPS updated the speed, direction, and time value was also updated.

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

The second and final sensor I used was a MPU-6050 module which contains a 3-Axis accelerometer chip as well as a 3-Axis gyroscope. This module is connected to the Arduino using the SDA and SCL pins as it uses I2C protocol. The gyroscope gives x, y, and z axis gyroscopic data that can be translated into roll and pitch degrees. When incorporating this into the Arduino code I added and calibration step which calibrates the Gyroscope to 0, 0, 0 right when you turn it on. This means that for the best reference point and accuracy the device should be turned on level surface. Both the calibration and the fetching and parsing of the data was done through the MPU6050_light library.

The start up code for the MPU-6050 is show below.

```c++
// Initiates gyro
Wire.begin();
mpu.begin();
delay(1000);
mpu.calcOffsets(); // gyro and accelerometer offsets calculation
Serial.println("Done!\n");
```

The very simple code is show below to get the roll, pitch, and yaw from the module using the library functions.

```c++
mpu.update();
Roll = mpu.getAngleX();
Pitch = mpu.getAngleY();
Yaw = mpu.getAngleZ();

```

# Communication data with flutter app

# Flutter

# Demo

# Further Development
