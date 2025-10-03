# Supported Virtual Robots

## Currently Supported Robots
- Multirotor
- Car
- Omrover
- Heli

## Common Units

### Actuators
- **`pwm`**  
  Integer representing microseconds. The range is **1100–2000 µs**, which is a typical operation range of ESCs in real drones.

### Sensors
- **`accelerometer `**  
  Raw accelerometer values for x, y, and z axes.  
  - **1g (9.81 m/s²)** corresponds to **16384**.  
  - Conversion to m/s²:  
    ```
    (accelerometer * 9.81) / 16384
    ```

- **`gyroscope `**  
  250 deg/second corresponds to 32768.  
  - Conversion to rad/s:  
    ```
    (gyroscope * 250 / 32768) * (π / 180)
    ```

- **`magnetometer`**  
  Values are measured in microtesla (µT).

### States
- **`states.angVel`** — in radians per second (rad/s)  
- **`states.euler `** — in degrees (°)  
- **`states.linVel `** — in meters per second (m/s)  
- **`states.linPos `** — in meters (m)  
