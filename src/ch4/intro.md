# Pure Zenoh


As noticed vrobot and vrobot pthon script is based on zenoh. Once can always use pure zenoh compeletely indepenedt to vrobots.

z_pub.py
```py
import time
import zenoh

# Configure Zenoh session
conf = zenoh.Config()
session = zenoh.open(conf)

# Declare a publisher on a key expression
key_expr = "demo/example"
publisher = session.declare_publisher(key_expr)

print(f"Publishing on '{key_expr}' ... Press Ctrl+C to stop.")
try:
    i = 0
    while True:
        msg = f"Hello Zenoh! {i}"
        publisher.put(msg)
        print(f">>> Published: {msg}")
        i += 1
        time.sleep(1)
except KeyboardInterrupt:
    pass

# Clean up
session.close()


```

z_sub.py
```py
import zenoh

# Configure Zenoh session
conf = zenoh.Config()
session = zenoh.open(conf)

# Callback for received messages
def listener(sample):
    print(f"<<< Received [{sample.key_expr}]: '{sample.payload.decode()}'")

# Subscribe to the same key expression
key_expr = "demo/example"
sub = session.declare_subscriber(key_expr, listener)

print(f"Subscribed on '{key_expr}' ... Press Ctrl+C to stop.")
try:
    while True:
        pass  # Keep the script alive
except KeyboardInterrupt:
    pass

# Clean up
session.close()

```