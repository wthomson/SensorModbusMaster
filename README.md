# SensorModbusMaster

This library is designed to use an Arduino as a Modbus master to communicate with a sensor/slave via the [Modbus RTU](https://en.wikipedia.org/wiki/Modbus) communication protocol.  It's specifically written with lots of "higher-level" functions to help out users who are largely unfamiliar with the Modbus protocol and want an easy way to get information from a Modbus device.

### Contents:
- [Using the library](#usingLibrary)
- [Notes on modbus maps](#ModbusMaps)
- [Supported data types](#DataTypes)
- [Notes on TTL and RS485/RS322 electrical communications standards](#notesRS485)
- [Hardware interface suggestions for EnviroDIY Mayfly and other Arduino boards](#hardwareMayfly)

_____

## <a name="usingLibrary"></a>Using the library

To communicate with a Modbus sensor or other Modbus slave, first create a stream instance (ie, Serial1, SoftwareSerial, AltSoftSerial) and then an instance of the modbusMaster.

```cpp
// Create the stream instance
HardwareSerial modbusSerial = Serial1;  // ALWAYS use HardwareSerial if it's an option
// OR
// AltSoftSerial modbusSerial;  // AltSoftSerial should be your second choice, if your board is supported
// OR
// SoftwareSerial modbusSerial(txPin, rxPin);  // SoftwareSerial should be your last choice.

// Create the modbus instance
modbusMaster modbus;
```

Within the setup function begin both the serial instance and the modbusMaster instance.  The enable pin allows you to use an RS485 to TTL adapter that is half-duplex with a pin that enables data sending.

```cpp
// Start the stream
modbusSerial.begin(baudRate);

// start the modbus
modbus.begin(modbusSlaveID, modbusSerial);
// ^^ use this if you have an RS485 adapter with automatic flow control
// modbus.begin(modbusSlaveID, modbusSerial, enablePin);
// ^^ use this if you need to manually control flow direction on your RS485 adapter
```

Once you've created and begun these, getting data from or adding data to a register is very simple:

```cpp
// Retrieve data from a read-only input register
// Input registers are called with 0x04 - Input registers are always read-only
// This gets a 32-bit big endian float from input register 15
modbus.float32FromRegister(0x04, 15, bigEndian);

// Retrieve data from a read/write holding register
// Holding registers are called with 0x03
// Retrieve a signed 16-big endian float from holding register 12
modbus.int16FromRegister(0x03, 12, bigEndian);

// Write the value "56" to holding register 20 as a little-endian unsigned 16-bit integer
modbus.uint16ToRegister(20, 56, littleEndian);
```
_____


## <a name="ModbusMaps"></a>Modbus Maps

While Modbus RTU specifications define the format of a data frame and a very simple data structure for a master and slave, there are no specification for what types of data a slave stores, where it is stored, or in what format it is stored.  You **MUST** get this information from the manufacturer/programmer of your Modbus device.  Typically this information is shared in what is called a Modbus map.

You need the following data from your Modbus map:
- the baud rate the device communicates at (Modbus allows any baud rate)
- the parity the device uses on the serial line (Modbus technically allows 8O1, 8E1, and 8N2, though some devices may use 8N1)
    - Software serial libraries generally do not support 8O1, 8E1, or 8N2; you will need a hardware serial port available if your device uses one of these stop/parity configurations
- the type of register or coils the data you are interested is stored in (ie, holding register, input register, coil, or discrete input)
    - Note - This library does not currently support getting or setting values in coils.
    - All input registers are read-only.  If your map says a register is read/write (R/W), it is a holding register.
- the register or coil number the data is stored in
- the format of data within the registers/coils (ie, float, integer, bitmask, ascii text)
- whether multi-register numeric data is stored as ["big-endian" or "little-endian"](https://en.wikipedia.org/wiki/Endianness) values (That is, is it high _word_ first or low _word_ first.  There are no specifications for this.)
- whether single-register data is stored "big-endian" or "little-endian" (That is, is it high _byte_ first or low _byte_ first.  Modbus specifies that it should be high byte first (big-endian), but devices vary.)
    - Note - This library only supports data that is "fully" big or little endian.  That is, data must be both high byte and high word first or both low byte and low word first.  Mixed endianness is not supported.

Without this information, you have little hope of being able to communicate properly with the device.  You can use programs like [CAS Modbus scanner](http://www.chipkin.com/cas-modbus-scanner/) to find a device if its address, baud rate, and parity are unknown, but it may take some time to make a connection.  You can also use the "scanRegisters" utility in this library to get a view of all the registers, but if you don't have a pretty good idea of what you are looking for that will not be as helpful as you might hope.
_____


## <a name="DataTypes"></a>Supported Data Types

The following data types are supported:
- uint16 (16-bit unsigned integer)
    - Value must be within a single 16-bit register
    - bigEndian or littleEndian can be specified, bigEndian will be used by default
    - By default, the Modbus command for pre-setting a single register will be used (0x06).  Set the forceMultiple boolean flag to 'true' to force the use of the Modbus command for setting multiple resisters (0x10).
- int16 (16-bit signed integer)
    - Value must be within a single 16-bit register
    - bigEndian or littleEndian can be specified, bigEndian will be used by default
    - By default, the Modbus command for pre-setting a single register will be used (0x06).  Set the forceMultiple boolean flag to 'true' to force the use of the Modbus command for setting multiple resisters (0x10).
- float32 (32-bit float)
    - Value must be in two adjacent 16-bit registers
    - bigEndian or littleEndian can be specified, bigEndian will be used by default
    - Only "fully" big or little endianness is supported - that is both high byte and high word first or both low byte and low word first.
- uint32 (32-bit unsigned integer)
    - Value must be in two adjacent 16-bit registers
    - bigEndian or littleEndian can be specified, bigEndian will be used by default
    - Only "fully" big or little endianness is supported - that is both high byte and high word first or both low byte and low word first.
- int32 (32-bit signed integer)
    - Value must be in two adjacent 16-bit registers
    - bigEndian or littleEndian can be specified, bigEndian will be used by default
    - Only "fully" big or little endianness is supported - that is both high byte and high word first or both low byte and low word first.
- TAI64 (64-bit timestamp)
    - Value must be in four contiguous 16-bit registers
    - Value is always fully big endian
    - Supported as if it were a 32-bit unix timestamp because the first 16-bits of the TAI64 timestamp will be 0x40000000 until the year 2106.
    - See https://www.tai64.com/ for more details on this format type
- TAI64N (64-bit timestamp followed by a 16-bit nanosecond count)
    - Value must be in six contiguous 16-bit registers
    - Value is always fully big endian
    - Note that the seconds and nanoseconds are broken into two different fields.
    - See https://www.tai64.com/ for more details on this format type
- TAI64NA (64-bit timestamp followed by a 16-bit nanosecond count and then a 16-bit attosecond count)
    - Value must be in eight contiguous 16-bit registers
    - Value is always fully big endian
    - Note that the seconds, nanoseconds, and attoseconds are broken into three different fields.
    - See https://www.tai64.com/ for more details on this format type
- byte (8-bit unsigned integer or raw byte of data)
    - Must specify either the first or second 8-bit component of a single 16-bit register)
    - By default, the Modbus command for pre-setting a single register will be used (0x06).  Set the forceMultiple boolean flag to 'true' to force the use of the Modbus command for setting multiple resisters (0x10).
- char (c++/ASCII style characters)
    - Characters can be in one or more contiguous 16-bit registers
    - Length of the character array must be specified
    - By default, the Modbus command for pre-setting a single register will be used (0x06) if the character array has two or fewer characters.  Set the forceMultiple boolean flag to 'true' to force the use of the Modbus command for setting multiple resisters (0x10).
- String (Arduino Strings)
    - Characters can be in one or more contiguous 16-bit registers
    - By default, the Modbus command for pre-setting a single register will be used (0x06) if the String has two or fewer characters.  Set the forceMultiple boolean flag to 'true' to force the use of the Modbus command for setting multiple resisters (0x10).
- pointer (pointers to other registers)
    - Value must be within a single 16-bit register
    - By default, the Modbus command for pre-setting a single register will be used (0x06).  Set the forceMultiple boolean flag to 'true' to force the use of the Modbus command for setting multiple resisters (0x10).

There are also mid-level functions available to help to reduce serial traffic by calling many registers at once and low level functions to make raw Modbus calls.  See SensorModbusMaster.h for all the available functions and their required and optional inputs
_____


## <a name="notesRS485"></a>Notes on TTL and RS485/RS322 electrical communications standards
While Modbus RTU specifications define the format of a data frame transferred over a serial line, the type of serial signal is not defined.  Many Modbus sensors communicate over [RS-485](https://en.wikipedia.org/wiki/RS-485).  To interface between RS485 and the TTL used by standard Arduino-type boards, you will need an RS485-to-TTL adapter. There are a number of RS485-to-TTL adapters available.  When shopping for one, be mindful of the logic level of the TTL output by the adapter.  The MAX485, one of the most popular adapters, has a 5V logic level in the TTL signal.  This will _fry_ any board that can only use on 3.3V logic.  You would need a voltage shifter in between the Mayfly and the MAX485 to make it work.  Also note that most RS485-to-TTL adapters are implemented _without_ automatic flow control.  That is, you must manually set voltages on driver enable and receiver enable pins to control the data flow direction.  While this library includes functions for setting the enables, I've found commutation to be much more stable on adapters with built-in flow control.  You will also need a [hardware interface board](#hardwareMayfly) to communicate between an Arduino and any Modbus sensor that communicates over [RS422](https://en.wikipedia.org/wiki/RS-422) or [RS232](https://en.wikipedia.org/wiki/RS-232).  Again, mind your voltages and the method of direction control.

## <a name="hardwareMayfly"></a>Hardware interface suggestions for EnviroDIY Mayfly
For use with the [EnviroDIY Mayfly datalogger](https://github.com/EnviroDIY/EnviroDIY_Mayfly_Logger) and other Arduino framework boards, you will likely need some additional [Hardware for using ModbusRTU with RS-485 on Arduino](https://github.com/EnviroDIY/SensorModbusMaster/tree/master/hardware).

We have developed a [Modbus-Mayfly-WingShield](https://github.com/EnviroDIY/SensorModbusMaster/tree/master/hardware/Modbus-Mayfly_WingShield) that combines AltSoftSerial TTL to RS-485 conversion with an optional power boost to 9V or 12V. This has worked quite well for us for a variety of deployment configurations and sensors.
