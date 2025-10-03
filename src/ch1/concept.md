# concept

## Backgrounds

Ubicoders' Virtual Rorobts simulation is based on publish-subscribe pattern.


## Default publishers

Each virtual robot in the sim publishes topics like

- vr/[system_id]/states
- vr/[system_id]/cam/left/720p

## Default subscribers

By default, the sim subscribes command inputs like

- vr/[system_id]/cmd

## Service request

The service is a one-time request-response transport.

The sim has default service servier at

- vr/service





