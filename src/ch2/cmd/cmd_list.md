# Command List

List of available commands.

multirotor-only command
```py
def update_cmd_multirotor(self, pwm: List[int])
```

heli-only command
```py
def update_cmd_heli(self, force: float)
```

car-only command
```py
def update_cmd_car(self, torque, brake, steer)
```

omr-only command
```py
def update_cmd_omrover(self, actuators: List[float])
```


For all virtual robots.

```py
""" override the body frame force and torque """
def update_cmd_set_force_torque_body(
        self,
        fx: float,
        fy: float,
        fz: float,
        tx: float,
        ty: float,
        tz: float,
)
```





























