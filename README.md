# Tutorial: Revolution Pi + Docker + MQTT + Grafana

  

## Introduction

  

This tutorial is written for **PLC programmers** who are familiar with classic automation tasks but want to explore modern IT technologies like **Docker** and **Grafana** on industrial controllers.

  

The **Revolution Pi** is sometimes described as a *“smart PLC”*:

- It can run classic control tasks (reading sensors, writing outputs).

- At the same time, it can execute **arbitrary software** (databases, dashboards, analytics, AI models).

  

This combination opens up new possibilities in **Industrial IoT (IIoT)** and **Edge Computing**.

  

---

  

## Why Docker?

  

On normal PCs, you may know the problem of **version conflicts**:

- One program runs only with an old Windows version.
- Another program requires the new version.
- You can not run both on the same computer.
  

**Docker solves this**:

- Each application runs in its own **container**, with its own libraries.

- Containers are isolated but can talk to each other over the network.

- Updating or rolling back one container does not affect the others.

  

---

  

## What is Edge Computing?

  

In the cloud (data centers), containers are used at massive scale.

But sometimes it makes more sense to run software **locally at the edge** on the RevPi, directly on the device close to the machine.

  

**Edge Computing** means:

- Processing happens **near the source of the data**, not only in the cloud.

- Examples:

- Local pre-processing of sensor data (e.g. Fourier transformation, anomaly detection).

- Local dashboards that remain available even if the internet is down.

  

**Our example** is an edge use case:

- The Revolution Pi measures internal values (CPU temperature, frequency).

- These values are sent out via **MQTT**.

- A local **Grafana dashboard** visualizes them in the browser — no cloud required.

  

---

  

## What is MQTT?

  

**MQTT** = *Message Queuing Telemetry Transport*

- A lightweight publish/subscribe protocol, widely used in IoT.

- Devices **publish** messages to topics.

- Other devices or apps **subscribe** to these topics.

- A central component, the **MQTT broker**, handles the message distribution.

  

Example:

- Publisher: “CPU temperature = 61 °C” on topic `revpi/mqtt_client/io/Core_Temperature`.

- Subscriber: Grafana, which listens to that topic and shows the values.

  

---

  

## What is Grafana?

  

**Grafana** is an open-source dashboard platform.

- It connects to different data sources (databases, MQTT, time series stores).

- It displays data as **graphs, gauges, tables**.

- In our example, Grafana will visualize the values published by the RevPi.

  

Result: You open `http://<revpi-ip>:3000` in a browser and see live data.

  

---

  

## Tutorial Steps

  

### 1. System preparation

Update the system and install tools for testing MQTT:

```bash

sudo apt update

sudo apt upgrade -y

sudo apt install -y curl mosquitto-clients

```

  

### 2. Install Docker

Docker provides the container runtime.

```bash

curl -fsSL https://get.docker.com | sh

sudo usermod -aG docker pi

sudo reboot

```

  

### 3. Set up Mosquitto (MQTT broker)

We will run Mosquitto in Docker. It acts as the “message switchboard” between publisher and subscriber.

  

**Create config and directories:**

```bash

sudo mkdir -p /opt/mosquitto/{config,data,log}

  

sudo tee /opt/mosquitto/config/mosquitto.conf > /dev/null <<'EOF'

listener 1883

allow_anonymous false

password_file /mosquitto/config/passwords

persistence true

persistence_location /mosquitto/data/

log_dest stdout

EOF

```

  

**Create user and password (pi / 5vneql):**

```bash

docker run --rm -v /opt/mosquitto/config:/mosquitto/config eclipse-mosquitto mosquitto_passwd -b /mosquitto/config/passwords pi 5vneql

  

sudo chown root:root /opt/mosquitto/config/passwords

sudo chmod 600 /opt/mosquitto/config/passwords

sudo chown -R 1883:1883 /opt/mosquitto/{data,log}

```

  

**Run Mosquitto:**

```bash

docker run -d --name mosquitto -p 1884:1883 -v /opt/mosquitto/config:/mosquitto/config -v /opt/mosquitto/data:/mosquitto/data -v /opt/mosquitto/log:/mosquitto/log eclipse-mosquitto

```

  

**Test with publish/subscribe:**

```bash

mosquitto_pub -h localhost -p 1884 -u pi -P 5vneql -t test/ping -m 1

mosquitto_sub -h localhost -p 1884 -u pi -P 5vneql -t test/# -C 1 -v

```

  

---

  

### 4. Configure RevPi MQTT client

The RevPi has a built-in MQTT client. In **PiCtory** configure:

  

- **Broker**: localhost

- **Port**: 1884

- **Username**: pi

- **Password**: 5vneql

- **Base topic**: revpi/mqtt_client

- Export IOs you want to send

  

Save → Reset Driver.

  

Restart service:

```bash

sudo systemctl restart revpipyload

```

  

Check published messages:

```bash

mosquitto_sub -h localhost -p 1884 -u pi -P 5vneql -t "revpi/mqtt_client/#" -v

```

  

---

  

### 5. Run Grafana

Grafana provides the dashboard.

```bash

docker run -d --name=grafana -p 3000:3000 -v grafana-storage:/var/lib/grafana grafana/grafana-oss:latest

```

  

---

  

### 6. Connect Grafana and Mosquitto

Put both containers in the same network so Grafana can reach Mosquitto by hostname:

```bash

docker network create revpi-net

docker network connect revpi-net mosquitto

docker network connect revpi-net grafana

```

  

---

  

### 7. Install MQTT plugin in Grafana

```bash

docker exec -it grafana grafana cli plugins install grafana-mqtt-datasource

docker restart grafana

```

  

---

  

### 8. Create Dashboard in Grafana

1. Open browser: `http://<revpi-ip>:3000` (login: admin / admin)

2. Add data source: **MQTT**

- Server: `tcp://mosquitto:1883`

- Username: pi

- Password: 5vneql

- Save & Test

3. Create panels:

- Panel 1: `revpi/mqtt_client/io/Core_Temperature` → Time series

- Panel 2: `revpi/mqtt_client/io/Core_Frequency` → Time series

  

---

  

## Result

  

- The RevPi publishes its internal values via MQTT.

- Mosquitto delivers them to subscribers.

- Grafana subscribes and displays the values in real time.

  

This small example shows:

- How **Docker** isolates and manages software components.

- How **MQTT** provides lightweight data exchange.

- How **Grafana dashboards** visualize data locally without cloud.

- And how **Edge Computing** brings IT tools to PLC-like devices.
