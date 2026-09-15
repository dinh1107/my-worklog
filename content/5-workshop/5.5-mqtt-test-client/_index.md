---
title: "Verifying MQTT Data with the MQTT Test Client"
date: 2026-09-14
weight: 5
chapter: false
pre: "<b>5.5. </b>"
---

# Verifying MQTT Data with the MQTT Test Client

In this section, we will use the AWS IoT Core MQTT Test Client to verify the data published by the IoT Gateway.

This test confirms that:

- The gateway is connected to AWS IoT Core.
- The certificate and IoT policy are working.
- Telemetry is published periodically.
- Alert events are published immediately.
- MQTT topics and JSON payloads match the design.

## Topics to verify

| Data type | MQTT topic |
|---|---|
| Telemetry | `smarthome/home01/esp32-01/telemetry` |
| Events | `smarthome/home01/esp32-01/events` |

---

## 5.5.1. Open the MQTT Test Client

Sign in to the AWS Management Console using:

```text
dinh-fcj
```

Verify the Region:

```text
Asia Pacific (Singapore)
ap-southeast-1
```

Open:

```text
AWS IoT Core
→ Test
→ MQTT test client
```

Select:

```text
Subscribe to a topic
```

The MQTT Test Client subscribes to MQTT topics and displays messages when they are received by AWS IoT Core.

---

## 5.5.2. Subscribe to the telemetry topic

Under **Topic filter**, enter:

```text
smarthome/home01/esp32-01/telemetry
```

Select:

```text
Subscribe
```

The topic should appear in the **Subscriptions** list.

> MQTT topics are case-sensitive. The topic must exactly match the system configuration.

---

## 5.5.3. Subscribe to the events topic

Return to **Subscribe to a topic** and enter:

```text
smarthome/home01/esp32-01/events
```

Select:

```text
Subscribe
```

The **Subscriptions** list should now contain both topics.

![Subscribed MQTT topics](/images/5.5.3.png)

### Why are the topics subscribed separately?

Separate subscriptions make it easier to:

- Monitor each data type.
- Distinguish periodic data from urgent events.
- Avoid mistakes when creating AWS IoT Rules.
- Identify the source of each message.

---

## 5.5.4. Verify telemetry

Select:

```text
smarthome/home01/esp32-01/telemetry
```

Wait up to approximately 30 seconds for the next message.

Each message should contain information such as:

| Field | Description |
|---|---|
| Home ID | Identifies `home01` |
| Device ID | Identifies `esp32-01` |
| Timestamp | Time when the data was generated |
| Temperature | Room temperature |
| Humidity | Room humidity |
| Gas value | Raw gas sensor ADC value |
| Device states | Door, light, or fan state |

The actual JSON payload can contain additional state fields depending on the system version.

![Telemetry message in the MQTT Test Client](/images/5.5.3.png)

### Expected result

- The message appears under the correct topic.
- The payload is valid JSON.
- Home ID and Device ID are correct.
- Temperature and humidity values are available.
- Gas is represented as a raw ADC value.
- New messages appear approximately every 30 seconds.

> Do not describe the gas value as ppm unless the sensor has been calibrated using a reference gas.

---

## 5.5.5. Verify events

Select:

```text
smarthome/home01/esp32-01/events
```

Generate a new event from the previously prepared smart home system. Device-side event generation is outside the scope of this AWS workshop.

The project uses the following event types:

| Event type | Description |
|---|---|
| `gas_alarm_started` | Gas value exceeds the warning threshold |
| `gas_alarm_cleared` | Gas value returns to the safe range |
| `door_brute_force_detected` | Multiple incorrect door-password attempts |

The event should appear immediately and should not wait for the 30-second telemetry interval.

![Event message in the MQTT Test Client](/images/5.5.5.png)

### Expected result

- The event appears under the correct topic.
- The payload is valid JSON.
- Device identification is available.
- The event type and timestamp are available.
- No door password or secret information is included.
- The event is received almost immediately.

> The MQTT Test Client mainly displays messages received after the subscription begins. Previous events will not automatically reappear unless they were published as retained messages.

---

## 5.5.6. Use a wildcard for troubleshooting

If the publishing topic is unknown, temporarily subscribe to:

```text
smarthome/home01/esp32-01/#
```

The `#` character matches every topic level after the specified prefix. This filter receives both:

```text
smarthome/home01/esp32-01/telemetry
smarthome/home01/esp32-01/events
```

After identifying the correct topics, return to separate exact subscriptions.



---

## Troubleshooting missing messages

If no message appears:

1. Verify that the Region is `ap-southeast-1`.
2. Check that the topic is entered exactly.
3. Verify that the certificate status is `Active`.
4. Confirm that the IoT policy allows the required Client ID and topics.
5. Verify that the gateway uses the correct Device data endpoint.
6. Confirm that the source system is publishing data.

The following filter can be used temporarily:

```text
#
```

If messages appear under another topic, review the topic configuration.

---

## Completion checklist

- [ ] The MQTT Test Client is open in the Singapore Region.
- [ ] The telemetry topic has been subscribed.
- [ ] The events topic has been subscribed.
- [ ] Telemetry appears approximately every 30 seconds.
- [ ] Events appear immediately when generated.
- [ ] The received payload is valid JSON.
- [ ] Gas values are described as raw ADC data.
- [ ] No passwords or secret information appear in the payload.

## Conclusion

In this section, we confirmed that AWS IoT Core receives both periodic telemetry and immediate alert events.

This verifies that the endpoint, certificate, IoT policy, and MQTT connection are operating correctly. The data is now ready to be processed and stored by the AWS services configured in the following sections.

Next, we will create an Amazon DynamoDB table for telemetry and event data.

## References

- [AWS – View MQTT messages with the MQTT Test Client](https://docs.aws.amazon.com/iot/latest/developerguide/view-mqtt-messages.html)
- [AWS – MQTT in AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/mqtt.html)