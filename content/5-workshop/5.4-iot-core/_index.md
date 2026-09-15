---
title: "Setting Up AWS IoT Core"
weight: 4
chapter: false
pre: "<b>5.4. </b>"
---

# Setting Up AWS IoT Core

In this section, AWS IoT Core is configured to receive data from the IoT Gateway. Home Assistant and Mosquitto use an X.509 certificate to establish a secure connection to AWS and publish `telemetry` and `events` messages.

## 5.4.1. Objectives

After completing this section, the following resources are available:

| Component | Value |
|---|---|
| AWS Region | `ap-southeast-1` – Singapore |
| IoT Thing | `ha-gateway-home01` |
| MQTT client ID | `ha-gateway-home01` |
| IoT policy | `SmartHomeGatewayPolicy` |
| Telemetry topic | `smarthome/home01/esp32-01/telemetry` |
| Events topic | `smarthome/home01/esp32-01/events` |
| Protocol | MQTT/TLS on port `8883` |

The IoT Thing represents the gateway in AWS. The X.509 certificate authenticates the gateway, while the IoT policy determines which MQTT operations and topics the gateway may use.

## 5.4.2. Open AWS IoT Core

Sign in to the AWS Management Console using the authorized IAM account and verify the Region:

```text
Asia Pacific (Singapore) – ap-southeast-1
```

Search for and open:

```text
AWS IoT Core
```

The root account should not be used for routine deployment operations.

## 5.4.3. Create the IoT Thing

In AWS IoT Core, open:

```text
Manage → All devices → Things → Create things
```

Complete the following steps:

1. Select **Create single thing**.
2. Enter `ha-gateway-home01` as the Thing name.
3. A Device Shadow is not required for this workshop.
4. Select **Next**.

The name identifies the Home Assistant Gateway for `home01` and provides a consistent naming convention for future gateways.

![The ha-gateway-home01 IoT Thing](/images/5.4.3.png)

## 5.4.4. Create the IoT Policy

Open:

```text
Security → Policies → Create policy
```

Enter the policy name:

```text
SmartHomeGatewayPolicy
```

Switch to the JSON editor and enter the following policy. Replace `<ACCOUNT_ID>` with the AWS Account ID used for the deployment.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "iot:Connect",
      "Resource": "arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:client/ha-gateway-home01"
    },
    {
      "Effect": "Allow",
      "Action": "iot:Publish",
      "Resource": [
        "arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:topic/smarthome/home01/esp32-01/telemetry",
        "arn:aws:iot:ap-southeast-1:<ACCOUNT_ID>:topic/smarthome/home01/esp32-01/events"
      ]
    }
  ]
}
```

This policy follows the principle of least privilege:

- `iot:Connect` permits only the `ha-gateway-home01` client ID to connect.
- `iot:Publish` permits publishing only to the two topics used by `esp32-01`.
- The gateway receives no AWS IoT administration permissions or access to unrelated topics.


![The SmartHomeGatewayPolicy policy](/images/5.4.4.png)

## 5.4.5. Create the X.509 Certificate

During the certificate configuration step for the Thing, select:

```text
Auto-generate a new certificate
```

After AWS creates the certificate, download these files:

```text
device.pem.crt
private.pem.key
AmazonRootCA1.pem
```

Each file has a different purpose:

| File | Purpose |
|---|---|
| `device.pem.crt` | Identifies the IoT Gateway |
| `private.pem.key` | Authenticates the gateway |
| `AmazonRootCA1.pem` | Verifies the AWS IoT server certificate |

Set the certificate status to **Active**, and then select the existing policy:

```text
SmartHomeGatewayPolicy
```

Complete the Thing creation process. If the certificate was created separately, open the certificate and use **Attach to things** to associate it with `ha-gateway-home01`.


![Active certificate with the attached policy](/images/5.4.5.png)

The relationship between the three components is:

```text
IoT Thing: ha-gateway-home01
        ↕
X.509 certificate: Active
        ↕
IoT policy: SmartHomeGatewayPolicy
```

## 5.4.6. Retrieve the Device Data Endpoint

In AWS IoT Core, open:

```text
Settings → Device data endpoint
```

The endpoint used by this project is:

```text
akq6wc7dn4aef-ats.iot.ap-southeast-1.amazonaws.com
```

Mosquitto Bridge uses the following connection settings:

| Property | Value |
|---|---|
| Host | `akq6wc7dn4aef-ats.iot.ap-southeast-1.amazonaws.com` |
| Port | `8883` |
| Protocol | MQTT over TLS 1.2 |
| Client ID | `ha-gateway-home01` |

![AWS IoT Device Data Endpoint](/images/5.4.6.png)

## 5.4.7. Verify the Configuration

The AWS IoT Core setup is complete when:

- The `ha-gateway-home01` Thing exists.
- The X.509 certificate has the `Active` status.
- The certificate is associated with the `ha-gateway-home01` Thing.
- The `SmartHomeGatewayPolicy` policy is attached to the certificate.
- The policy permits publishing to the `telemetry` and `events` topics.
- The Device Data Endpoint for `ap-southeast-1` has been recorded.

In the next section, the endpoint and the three certificate files are used to establish the MQTT/TLS connection between the IoT Gateway and AWS IoT Core.
