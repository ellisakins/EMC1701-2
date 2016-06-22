# EMC1701-2 - High-Side Current Sensor

The [EMC1701-2](http://ww1.microchip.com/downloads/en/DeviceDoc/1701.pdf) is a high-side current and temperature sensor. Unless otherwise noted, all measurement units will be either Volts, Amps, Watts, or Celcius.

## Class Usage

### Constructor: EMC1701(*i2cBus[, i2cAddress][, senseResistorValue]*)

The class’ constructor takes one required parameter (a configured imp I&sup2;C bus) and 2 optional parameters (the I&sup2;C address of the sensor and the value of the sense resistor). The I&sup2;C address must be the address of your sensor or an I&sup2;C error will be thrown.

| Parameter     | Type         | Default | Description |
| ------------- | ------------ | ------- | ----------- |
| *i2cBus*      | hardware.i2c | N/A     | A pre-configured I&sup2;C bus |
| *i2cAddress*  | byte         | 0x30    | The I&sup2;C address |
| *senseResistorValue* | float | 0.01    | Value of sense resistor (in ohms) |

```squirrel
i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);

// Use non-default I2C settings
emc <- EMC1701(i2c, 0x32, 0.001);
```

## Class Methods

### setConversionRate(*rate*)

The *setConversionRate()* method sets the converion rate of the device in Hz. Supported datarates are 1/16, 1/8, 1/4, 1/2, 1, 2, 4, and 8 Hz. The requested data rate will be rounded up to the closest supported rate and the actual data rate will be returned.

The default data rate is 4 Hz.

```squirrel
local rate = emc.setConversionRate(7);
server.log(format("EMC1701 updating at %f Hz", rate));
// Displays 'EMC1701 updating at 8Hz'
```


### configureMode()

The *configureMode()* method sets mode of operation of the EMC1701, then returns the contents of the CONFIGURATION register. 

The default mode is Fully Active.

```squirrel
//must include these constants outside of the class
const FULLY_ACTIVE          = 0x00;
const CURRENT_ONLY          = 0x40;
const TEMP_ONLY             = 0x04;
const ALL_OFF               = 0x44;

local mode = emc.configureMode(CURRENT_ONLY);
server.log(format("EMC1701 mode: 0x%0X", mode));
// Displays 'EMC1701 mode: 0x40'
```


### configureVoltageSampler(*[numOutOfLimits][,numAverages][,peakDetectPin]*)

| Parameter     | Type         | Default | Description |
| ------------- | ------------ | ------- | ----------- |
| *numOutOfLimits*   | int | 1     | Number of consecutive measurements that VSOURCE must exceed the limits before flagging an interrupt |
| *numAverages*  | int         | 0    | The digital averaging that is applied to the source voltage measurement |
| *peakDetectPin* | boolean | false    | If false, the Peak Detector circuitry will assert the ALERT pin when a current spike is detected. The ALERT pin must be configured to operate in Comparator mode or it will not be asserted. If false, the Peak Detector circuitry will assert the THERM pin when a current spike is detected |


The *configureVoltageSampler()* method configures the device's voltage sampler. 

Choices for numOutOfLimits are 1 (default), 2, 3, and 4. Choices for numAverages are disabled (default), 2x, 4x, and 8x. Both numOutOfLimits and numAverages are rounded-up, bounded, and returned.

```Squirrel
// Configure voltage sampler with numOutOfLimits of 2 measurements, numAverages of 2x, and asserting the Peak Dector Circuitry to the THERM pin.
emc.configureVoltageSampler(2, 2, true);
//returns { "numOutOfLimits" : 2, "numAverages" : 2}
```


### configurePeakDetection(*[threshold][,duration]*)

| Parameter     | Type         | Default | Description |
| ------------- | ------------ | ------- | ----------- |
| *threshold*   | int          | 85      | The Peak Detector Threshold (in mV)|
| *duration*    | int          | 256     | The Peak Detector minimum time threshold (in ms) |

The *configurePeakDetection()* method controls the threshold and durations used by the Peak Detection circuitry. At all times, the Peak Detection threshold and duration are set by the values written into this register. The method will return the set threshold and duration.

```Squirrel
{ "threshold" : <_peakthreshold>,
  "duration"   : <_peakduration> }
```

### configureCurrentSampler(*[queue][,average][,time][,range]*)

| Parameter     | Type         | Default | Description |
| ------------- | ------------ | ------- | ----------- |
| *queue*       | int          | 1       | The number of consecutive measurements that the measured VSENSE must exceed the limits before flagging an interrupt|
| *average*     | int          | 1       | The number of averages that the Current Sensing Circuitry will perform |
| *time*        | int          | 82      | The sampling time of the Current Sensing Circuitry (in ms) |
| *range*       | int          | 80      | The Current Sense maximum expected voltage (full scale range) (in mV)|

The *configureCurrentSampler()* method The Current Sense Sampling Configuration register stores the controls for determining the Current Sense sampling / update time. The method will return the set parameters.

``` 
{
  "queue"        : int, 
  "averageing"   : int, 
  "samplingtime" : int, 
  "range"        : int
}
```


### read()

This method reads the present register values, and returns a measurement table:

```squirrel
local data = emc.read();
//data = { "current": float, "voltage": float, "power": float, "temp" : float }
```


### oneShot()

This method writes to the One Shot register, updating the measurement registers (in accordance with mode of operation) and returns a read() measurement:

```squirrel
local data = emc.oneShot();
//data = { "current": float, "voltage": float, "power": float, "temp" : float }
```


### fastRead()

This method performs a fast measurement by manipulating the peak detection register to find a rough estimate of the current, and returns a float measurement (in A):

```squirrel
local data = emc.fastRead();
//data = float
```


### getStatus()

This method returns the status of the EMC1701 in the following table:

```squirrel
{
  "busy": bool,  // True if one of the ADCs is currently converting. This bit does not cause either the ALERT or THERM pins to be asserted.
  "peak": bool,  // True if the Peak Detector circuitry has detected a current peak that is greater than the programmed threshold for longer than the programmed duration.
  "high": bool,  // True if any of the temperature, VSENSE, or VSOURCE channels meets or exceeds its programmed high limit.
  "low" : bool,  // True if any of the temperature, VSENSE, or VSOURCE channels drops below its programmed low limit.
  "crit": bool,  // True if any of the temperature, VSENSE, or VSOURCE channels meets or exceeds its programmed Tcrit limit.
}
```


### getVoltage()

This method returns a single voltage measurement.

```squirrel
// Get voltage
server.log(format("Voltage = %f Volts", emc.getVoltage()));
```


### getCurrent()

This method returns a single current measurement.

```squirrel
// Get voltage
server.log(format("Current = %f Amps", emc.getCurrent()));
```


### getPower()

This method returns a single power measurement.

```squirrel
// Get power
server.log(format("Power = %f Watts", emc.getPower()));
```


### getTemp([unit])

This method returns a single temperature measurement. Calling this method without any parameters will return a Celcius measurement; calling with "f" or "F" for paremeter will return the result in Fahrenheit.

```squirrel
// Get temp in Fahrenheit
server.log(format("Temperature = %f degF", emc.getTemp("F")));
```


### getProductID()

Returns the one-byte product ID of the sensor. This method is a simple way to test if your sensor is correctly connected.

```squirrel
server.log(format("Product ID: 0x%02X", emc.getProductID()));
```


### getSmscID()

Returns the one-byte manufacturer SMSC ID of the sensor.

```squirrel
server.log(format("SMSC ID: 0x%02X", emc.getSmscID()));
```

### getRevision()

Returns the one-byte die revision of the sensor.

```squirrel
local revision = emc.getRevision();
server.log(format("EMC1701 die revision: 0x%0X", revision));
```


## Examples

In this example, we will configure a single sensor with a default i2c address and sense resistor (10mOhm), and perform a default measurement.

```squirrel
i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);

emc <- EMC1701(i2c);
local data = emc.read();
local current = data.current;
local voltage = data.voltage;
local power = data.power;
local temp = data.temp;
```

In this example, we will configure multiple sensors and perform different types of measurents on each sensor.

```squirrel
i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);

emc1 <- EMC1701(i2c, 0x30, 0.02);
emc2 <- EMC1701(i2c, 0x4D, 0.01);
emc3 <- EMC1701(i2c, 0x4E, 0.001);

emc1.configureCurrentSampler(1, 2, 150, 40);
emc2.configureCurrentSampler(4, 8, 1000, 80);
emc3.configureCurrentSampler(2, 8, 150, 10);

local data1 = emc1.read();
local data2 = emc2.oneShot();
local data3 = emc3.fastRead();
```

In this example, we will configure one sensor and perform a fast read, then turn the sensor off.

```squirrel
i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);

emc <- EMC1701(i2c, 0x30, 0.02);

local fastReadData = emc.fastRead();
local data = emc.fastRead();
emc.configure(ALL_OFF);
// data = float
```


### To-Do

ALERT and THERM pin setup and callbacks, Temperature and Voltage limit configuration.
