# LIS3DH 3-Axis Accelerometer

The [LIS3DH](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/datasheet/CD00274221.pdf) is a 3-Axis MEMS accelerometer. This sensor has extensive functionality and this class has not yet implemented all of it.

The LPS25H can interface over I&sup2;C or SPI. This class addresses only I&sup2;C for the time being.

To add this library to your project, add #require "LIS3DH.class.nut:1.0.0" to the top of your device code.

## Class Methods

### constructor(*i2c, [addr]*)

The class’ constructor takes one required parameter (a configured imp I&sup2;C bus) and an optional parameter (the I&sup2;C address of the accelerometer):

| Parameter     | Type         | Default | Description |
| ------------- | ------------ | ------- | ----------- |
| i2c           | hardware.i2c | N/A     | A pre-configured I&sup2;C bus |
| addr          | byte         | 0x30    | The I&sup2;C address of the accelerometer |

```squirrel
#require "LIS3DH.class.nut:1.0.0"

i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);

accel <- LIS3DH(i2c, 0x32); // using a non-default I2C address (SA0 pulled high)
```

### setDataRate(*rate_hz*)
The *setDataRate* method sets the Output Data Rate (ODR) of the accelerometer in Hz. The nearest supported data rate less than or equal to the requested rate will be used and returned. Supported datarates are 0 (Shutdown), 1, 10, 25, 50, 100, 200, 400, 1600, and 5000 Hz.

```squirrel
local rate = accel.setDataRate(100);
server.log(format("Accelerometer running at %d Hz",rate));
```

**NOTE:** The datarate must be set before reading the accelerometer with *getAccel()*.

### getAccel([*callback*])
The *getAccel* method reads and returns the latest measurement from the accelerometer as a table (in *G*s):

`{ x: <xData>, y: <yData>, z: <zData> }`

```Squirrel
// Create and enable the sensor
i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);
accel <- LIS3DH(i2c, 0x32);
accel.setDataRate(100);

local val = accel.getAccel()
server.log(format("Acceleration (G): (%0.2f, %0.2f, %0.2f)", val.x, val.y, val.z));
```

An optional callback (with a single parameter) can be passed to the *getAccel* method. If a callback is included, the class will read the sensor data, and pass the result to the callback method as the first parameter:

```Squirrel
// Create and enable the sensor
i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);
accel <- LIS3DH(i2c, 0x32);
accel.setDataRate(100);

accel.getAccel(function(val) {
    server.log(format("Acceleration (G): (%0.2f, %0.2f, %0.2f)", val.x, val.y, val.z));
});
```

### setRange(*range_g*)
The *setRange* method sets the measurement range of the sensor in *G*s. The default measurement range is +/- 2G. The nearest supported range less than or equal to the requested range will be used and returned. Supported ranges are (+/-) 2, 4, 6, 8, and 16 G.

```Squirrel
// set sensor range to +/- 6 G
local range = accel.setRange(6);
server.log(format("Range set to +/- %d G", range));
```

**NOTE:** If you are not using the default +/- 2G range, you must set the range with *setRange* before setting interrupt thresholds with *setInertialIntThreshold* or *setClickIntThreshold*.

### getRange()
The *getRange* method returns the currently-set measurement range of the sensor in *G*s.

```Squirrel
server.log(format("Current Sensor Range is +/- %d G", accel.getRange()));
```

### configureInertialInterrupt(*state, [threshold, duration, options]*)

Configures the Inertial Interrupt generator:

| parameter | type     | default                    | description |
| --------- | -------- | -------------------------- | ----------- |
| state     | bool     | n/a                        | `true` to enable, `false` to disable |
| threshold | float    | 1.0                        | Inertial interrupts threshold in *G*s. |
| duration  | int      | 5                          | Number of samples exceeding threshold required to generate interrupt |
| options   | bitfield | X_HIGH \| Y_HIGH \| Z_HIGH | See table below |

```squirrel
// Configure the Inertial interrupt generator to generate an interrupt
// when acceleration on all three exceeds 1G.
accel.configureInertialInt(true, 1.0, 10, LIS3DH.X_LOW | LIS3DH.Y_LOW | LIS3DH.Z_LOW | LIS3DH.AOI)
```

The default configuration for the Intertial Interrupt generator is to generate an interrupt when the acceleration on *any* axis exceeds 1G. This behavior can be changed by OR'ing together any of the following flags:

| flag   | description |
| ------ | ----------- |
| X_LOW  | Generates an interrupt when the x-axis acceleration goes below the threshold |
| X_HIGH | Generates an interrupt when the x-axis acceleration goes above the threshold |
| Y_LoW  | Generates an interrupt when the y-axis acceleration goes below the threshold |
| Y_HIGH | Generates an interrupt when the y-axis acceleration goes above the threshold |
| Z_LOW  | Generates an interrupt when the z-axis acceleration goes below the threshold |
| Z_HIGH | Generates an interrupt when the z-axis acceleration goes above the threshold |
| AOI    | Sets the AOI flag (see **Inertial Interrupt Modes** below) |
| SIX_D  | Sets the 6D flag (see **Inertial Interrupt Modes** below) |

#### Inertial Interrupt Modes

The following is taken from the from [LIS3DH Datasheet](http://www.st.com/st-web-ui/static/active/en/resource/technical/document/datasheet/CD00274221.pdf) (section 8.21):

| AOI | 6D  | Interrupt Mode                     |
| --- | --- | ---------------------------------- |
|  0  |  0  | OR combination of interrupt events |
|  0  |  1  | 6 direction movement recognition   |
|  1  |  0  | AND combination of events          |
|  1  |  1  | 6 direction position recognition   |

**Movement Recognition (01)**: An interrupt is generate when orientation move from unknown zone to known zone. The interrupt signal stay for a duration ODR.

**Direction Recognition (11)**: An interrupt is generate when orientation is inside a known zone. The interrupt signal stay until orientation is inside the zone.

### configureFreeFallInt(*state, [threshold, duration]*)
The *configureFreeFallInt* configures the intertial interrupt generator to generate interrupts when the device is in free fall (acceleration on all axis appraoches 0). The default `threshold` is 0.5 Gs.The default `duration` is 5 sample.

```Squirrel
accel.configureFreeFallInt(true);
```

**Note:** This method will overwrite any settings configured with the *configureInertialInt*.

### configureClickInterrupt(*state, [clickType, threshold, timeLimit, latency, window]*)

Configures the Click Interrupt Generator:

| parameter | type       | default                    | description                                              |
| --------- | ---------- | -------------------------- | -------------------------------------------------------- |
| state     | bool       | n/a                        | `true` to enable, `false` to disable                     |
| clickType | CONST      | LIS3DH.SINGLE_CLICK        | LIS3DH.SINGLE_CLICK or LIS3DH.DOUBLE_CLICK               |
| threshold | float      | 1.1                        | Threshold that must be exceeded to be considered a click |
| timeLimit | float      | 5                          | Max time in *ms* the acceleration can spend above the threshold to be considered a click |
| latency   | float      | 10                         | Min time in *ms* between the end of one click event, and the start of another to be considred a DOUBLE_CLICK |
| window    | float      | 50                         | Max time in *ms* between the start of one click event, and end of another to be considered a DOUBLE_CLICK |

#### Single Click example
```squirrel
// Configure a single click interrupt
accel.configureClickInt(true, LIS3DH.SINGLE_CLICK);
```

#### Double Click Example
```squirrel
// configure a double click interrupt
accel.configureClickInt(true, LIS3DH.DOUBLE_CLICK);
```

### configureDataReadyInterrupt(*state*)
Enables (state = `true`) or disable (state = `false`) Data Ready interrupts on the INT1 line.

```Squirrel
accel.setDataRate(1); // 1 Hz
accel.configureDataReadyInt(true);
```

### setInterruptLatching(*state*)
Enables (state = `true`) or disables (state = `false`) interrupt latching. If interrupt latching is enabled, the interrupt signal will remain asserted until the interrupt source register is read by calling *getInterruptTable()*. If latching is disabled, the interrupt signal will remain asserted as long as the interrupt-generating condition persists.

Interrupt latching is disabled by default.

**See sample code in *getInterruptTable()*.**

### getInterruptTable()
The *getInterruptTable* method reads the INT1_SRC and CLICK_SRC registers, and returns the result as a table with the following fields:

```squirrel
{
    "int1": bool,           // true if INT1 created the interrupt
    "xLow": bool,           // true if a xLow condition is present
    "yLow": bool,           // true if a yLow condition is present
    "zLow": bool,           // true if a zLow condition is present
    "xHigh": bool,          // true if a xHigh condition is present
    "yHigh": bool,          // true if a yHigh condition is present
    "zHigh": bool,          // true if a zHigh condition is present
    "click": bool,          // true if any click created the interrupt
    "singleClick": bool,    // true if a single click created the interrupt
    "doubleClick": bool     // true if a double click created the interrupt
}
```

In the following example we setup interrupts for free fall detection, and double click detection, and check what caused the interrupt in our interrupt handler:

```Squirrel
function interruptHandler() {
    if (int.read() == 0) return;

    // Get + clear the interrupt + clear
    local data = accel.getInterruptTable();

    // Check what kind of interrupt it was
    if (data.int1) {
        server.log("Free Fall");
    }
    if (data.doubleClick) {
        server.log("Double Click");
    }
}

i2c <- hardware.i2c89;
i2c.configure(CLOCK_SPEED_400_KHZ);
accel <- LIS3DH(i2c, 0x32);

int <- hardware.pinB;
int.configure(DIGITAL_IN, interruptHandler);

// Configure accelerometer
accel.setDataRate(100);
accel.setInterruptLatching(true);

// Setup a free fall interrupt
accel.configureFreeFallInt(true);

// Setup a double click interrupt
accel.configureClickInt(true, LIS3DH.DOUBLE_CLICK);
```

### init()
The *init* method resets all registers to datasheet default values. The *init* method is called as part of the class constructor.

### getDeviceId()
Returns the 1-byte device ID of the sensor (from the WHO_AM_I register). The *getDeviceId* method is a simple way to test if your LIS3DH sensor is correctly connected.

```squirrel
server.log(format("Device ID: 0x%02X", accel.getDeviceId()));
```

### enable(*state*)
The *enable* methods enables (state = `true`) or disabled (state = `false`) the accelerometer. The accelerometer is enabled by default.

```squirrel
function goToSleep() {
    imp.onidle(function() {
        // set datarate to 0 and disable the accelerometer to save power
        accel.setDataRate(0);
        accel.enable(false);

        // sleep for 1 hour
        server.sleepfor(3600);
    });
}
```

### setLowPower(*state*)
The *setLowPower* method enables (state = `true`) or disables (state = `false`) low-power mode.

```Squirrel
// enable low-power mode
accel.setLowPower(true);
```

**Note:** setLowPower will change the data rate.

## License

The LIS3DH class is licensed under [MIT License](./LICENSE).
