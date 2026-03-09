# 🌱 Smart IoT Farm

[![Language: C](https://img.shields.io/badge/language-C-blue.svg)](https://en.wikipedia.org/wiki/C_(programming_language))
[![Language: Java](https://img.shields.io/badge/language-Java-orange.svg)](https://www.java.com/)
[![Protocol: CoAP](https://img.shields.io/badge/protocol-CoAP-green.svg)](https://coap.technology/)
[![Build: Maven](https://img.shields.io/badge/build-Maven-red.svg)](https://maven.apache.org/)
[![OS: Contiki](https://img.shields.io/badge/OS-Contiki-lightblue.svg)](http://www.contiki-os.org/)

An automated IoT-based crop management system — codenamed **CottonNet** — designed for smart farming. The system monitors environmental conditions in real time using lightweight CoAP sensors and autonomously controls irrigation and lighting through intelligent actuators, including a machine-learning-based irrigation model.

---

## 📑 Table of Contents

- [Overview](#-overview)
- [System Architecture](#-system-architecture)
- [Project Structure](#-project-structure)
- [Technologies Used](#-technologies-used)
- [Prerequisites](#-prerequisites)
- [Installation & Setup](#-installation--setup)
- [Usage](#-usage)
- [Machine Learning Model](#-machine-learning-model)
- [Configuration](#-configuration)
- [Documentation](#-documentation)

---

## 🔍 Overview

Smart IoT Farm is a complete end-to-end IoT solution that integrates:

- **Edge sensors** running on Contiki OS, publishing environmental data (CO₂, light intensity, growing phase, soil moisture, and temperature) via CoAP.
- **Intelligent actuators** that autonomously react to sensor data using rule-based logic (lighting) and a decision-tree ML model (irrigation).
- **A central registration server** (Java) that coordinates device registration, persists measurements to a MySQL database, and observes sensor streams.
- **A remote control application** (Java CLI) that allows operators to monitor real-time measurements and remotely configure or shut down devices.

### ✨ Key Features

| Feature | Description |
|---|---|
| 🌡️ Multi-sensor monitoring | Measures CO₂, light intensity, growing phase, soil moisture, and temperature |
| 💧 Intelligent irrigation | Decision-tree ML model drives autonomous sprinkler decisions |
| 💡 Adaptive lighting | Rule-based LED controller adjusts colour based on CO₂ and growing phase |
| 📡 Lightweight communication | CoAP (RFC 7252) over 6LoWPAN for constrained IoT devices |
| 🗄️ Persistent storage | MySQL database stores all sensor measurements and device records |
| 🖥️ Remote control | Java CLI for real-time monitoring, sampling rate configuration, and device shutdown |

---

## 🏗️ System Architecture

```
╔══════════════════════════════════════════════════════════════════╗
║                      EDGE LAYER (Contiki-OS)                    ║
╠══════════════════════════════════════════════════════════════════╣
║  ┌─────────────────────────┐  ┌─────────────────────────────┐   ║
║  │  Illumination Sensor    │  │       Soil Sensor           │   ║
║  │  • CO₂ level            │  │  • Soil moisture            │   ║
║  │  • Light intensity      │  │  • Temperature              │   ║
║  │  • Growing phase        │  │  Resources: /soil           │   ║
║  │  Resources: /co2        │  │             /sampling       │   ║
║  │             /light      │  │             /turn_off       │   ║
║  │             /phase      │  └────────────┬────────────────┘   ║
║  │             /sampling   │               │ CoAP observe        ║
║  │             /turn_off   │               ▼                     ║
║  └────────────┬────────────┘  ┌─────────────────────────────┐   ║
║               │ CoAP observe  │    Sprinkler Actuator        │   ║
║               ▼               │  • ML-based irrigation       │   ║
║  ┌─────────────────────────┐  │    (emlearn decision tree)   │   ║
║  │  Lighting Actuator      │  └─────────────────────────────┘   ║
║  │  • Rule-based LED ctrl  │                                     ║
║  │  • YELLOW/RED/GREEN/OFF │                                     ║
║  └─────────────────────────┘                                     ║
╠══════════════════════════════════════════════════════════════════╣
║                CoAP (UDP port 5683) over 6LoWPAN                ║
╠══════════════════════════════════════════════════════════════════╣
║                      SERVER LAYER (Java)                        ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  Registration Server                                     │   ║
║  │  • POST /registration  ← devices register on startup    │   ║
║  │  • CoapObserver        ← subscribes to sensor streams   │   ║
║  │  • DBManager           → persists data to MySQL         │   ║
║  └───────────────────────────┬──────────────────────────────┘   ║
╠═══════════════════════════════╪══════════════════════════════════╣
║         MySQL (JDBC)          │                                  ║
║  ┌────────────────────────────▼─────────────────────────────┐   ║
║  │  Database: CottonNet                                     │   ║
║  │  ├── devices      (name, address, type, sampling)        │   ║
║  │  ├── illumination (id, co2, light, phase)                │   ║
║  │  └── soil         (id, moisture, temperature)            │   ║
║  └──────────────────────────────────────────────────────────┘   ║
╠══════════════════════════════════════════════════════════════════╣
║                   APPLICATION LAYER (Java CLI)                  ║
║  ┌──────────────────────────────────────────────────────────┐   ║
║  │  Remote Control Application                              │   ║
║  │  [1] Show registered devices                             │   ║
║  │  [2] Set illumination sampling interval                  │   ║
║  │  [3] Set soil sampling interval                          │   ║
║  │  [4] Show real-time illumination data                    │   ║
║  │  [5] Show real-time soil data                            │   ║
║  │  [6] Turn off all devices                                │   ║
║  └──────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 📁 Project Structure

```
Smart-IoT-Farm/
├── CoAP network/                    # IoT edge devices (C / Contiki-OS)
│   ├── cJSON/                       # Embedded JSON library
│   ├── illumination sensor/         # CO₂, light & phase sensor
│   │   ├── sensore.c
│   │   ├── project-conf.h
│   │   ├── global_variables.h
│   │   └── resources/
│   │       ├── res_observation.c    # CoAP observe endpoint
│   │       ├── res_co2.c
│   │       ├── res_light.c
│   │       ├── res_phase.c
│   │       ├── res_sampling.c
│   │       └── res_turnoff.c
│   ├── soil sensor/                 # Moisture & temperature sensor
│   │   ├── sensore.c
│   │   └── resources/
│   │       ├── res_soil.c
│   │       ├── res_sampling.c
│   │       └── res_turnoff.c
│   ├── lighting controller/         # LED actuator (rule-based)
│   │   ├── attuatore.c
│   │   └── resources/res_turnoff.c
│   └── sprinkler/                   # Irrigation actuator (ML-based)
│       ├── attuatore.c
│       ├── irrigation_model.h       # Compiled eml-learn decision tree
│       ├── machine learning model.ipynb
│       └── resources/res_turnoff.c
├── registration-server/             # Central CoAP server (Java / Maven)
│   ├── pom.xml
│   └── src/main/java/com/example/
│       ├── RegistrationServer.java
│       ├── RegistrationResource.java
│       ├── CoapObserver.java
│       └── DBManager.java
├── remote-control-application/      # User CLI application (Java / Maven)
│   ├── pom.xml
│   └── src/main/java/com/example/
│       └── RemoteControlApp.java
├── Californium.properties           # CoAP framework configuration
├── CottonNet - Documentation.pdf    # Full technical documentation
└── FASI ACCENSIONE LED              # LED state lookup table
```

---

## 🛠️ Technologies Used

| Component | Technology | Version |
|---|---|---|
| Embedded sensors & actuators | C / Contiki-OS | – |
| IoT communication protocol | CoAP (RFC 7252) / 6LoWPAN | – |
| CoAP framework (Java) | Eclipse Californium | 1.1.0-SNAPSHOT |
| Server & client applications | Java | 1.7+ |
| Build tool | Apache Maven | 3+ |
| Database | MySQL | 8.0.30 |
| JDBC driver | MySQL Connector/J | 8.0.30 |
| JSON (embedded) | cJSON | – |
| ML model | emlearn (decision tree) | – |

---

## ✅ Prerequisites

- **Contiki-OS** development environment (for compiling and simulating sensor/actuator nodes)
- **Java JDK** 1.7 or later
- **Apache Maven** 3+
- **MySQL** 8.0+ (running locally on port `3306`)
- **GCC** C compiler (for building Contiki applications)
- **Make**

---

## 🚀 Installation & Setup

### 1. Clone the repository

```bash
git clone https://github.com/Vittori00/Smart-IoT-Farm.git
cd Smart-IoT-Farm
```

### 2. Set up the MySQL database

Create the `CottonNet` database and the required tables:

```sql
CREATE DATABASE CottonNet;
USE CottonNet;

CREATE TABLE devices (
    name     VARCHAR(50) PRIMARY KEY,
    address  VARCHAR(100),
    type     VARCHAR(20),
    sampling INT
);

CREATE TABLE illumination (
    id     INT AUTO_INCREMENT PRIMARY KEY,
    co2    FLOAT,
    light  FLOAT,
    phase  INT
);

CREATE TABLE soil (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    moisture    FLOAT,
    temperature FLOAT
);
```

> Default credentials used by the application: **user** `admin` / **password** `admin`.  
> Update `DBManager.java` if you use different credentials.

### 3. Build the Registration Server

```bash
cd registration-server
mvn clean package
```

### 4. Build the Remote Control Application

```bash
cd ../remote-control-application
mvn clean package
```

### 5. Compile sensor and actuator nodes

Each device is compiled individually using Make inside its Contiki project folder. For example:

```bash
cd "CoAP network/illumination sensor"
make TARGET=cooja
```

Repeat for `soil sensor`, `lighting controller`, and `sprinkler`.

---

## 📖 Usage

Follow this startup sequence to run the full system:

1. **Start the Registration Server**

    ```bash
    cd registration-server
    java -jar target/registration-server-*.jar
    ```

2. **Launch the Cooja simulator** and load the compiled sensor/actuator firmwares.  
   Start the border router, then power on the nodes.

3. **Register sensors first** — press the button on each sensor node.  
   Sensors will POST to `/registration` and the server will assign them an entry in the database.

4. **Register actuators** — press the button on each actuator node.  
   Actuators will obtain sensor IP addresses from the registration server and begin observing them.

    > ℹ️ It is also possible to start all nodes simultaneously; however, registering sensors before actuators is recommended.

5. **Start the Remote Control Application**

    ```bash
    cd remote-control-application
    java -jar target/remote-control-application-*.jar
    ```

6. **Use the CLI menu** to monitor measurements and control the system:

    ```
    ╔══════════════════════════════════════════╗
    ║         Remote Control Application       ║
    ╠══════════════════════════════════════════╣
    ║  [1] Show registered devices             ║
    ║  [2] Set illumination sampling interval  ║
    ║  [3] Set soil sampling interval          ║
    ║  [4] Show real-time illumination data    ║
    ║  [5] Show real-time soil data            ║
    ║  [6] Turn off all devices                ║
    ║  [7] Exit                                ║
    ╚══════════════════════════════════════════╝
    ```

7. **Shutdown** — use option **[6]** in the remote control application to gracefully turn off all connected devices before exiting.

---

## 🤖 Machine Learning Model

The **Sprinkler** actuator uses an embedded decision-tree model trained with [emlearn](https://emlearn.org/) to decide whether to activate irrigation based on current sensor readings.

- **Input features**: `[1, moisture, temperature]` (3 features)
- **Decision logic**: threshold on moisture value (`495.0`) → binary output (0 = OFF, 1 = ON)
- **Model file**: [`CoAP network/sprinkler/irrigation_model.h`](CoAP%20network/sprinkler/irrigation_model.h)
- **Training notebook**: [`CoAP network/sprinkler/machine learning model.ipynb`](CoAP%20network/sprinkler/machine%20learning%20model.ipynb)

To retrain the model, open the Jupyter notebook, update the training data, and export the new `irrigation_model.h` file.

---

## ⚙️ Configuration

### CoAP Settings (`Californium.properties`)

| Parameter | Default | Description |
|---|---|---|
| `COAP_PORT` | `5683` | Standard CoAP UDP port |
| `COAP_SECURE_PORT` | `5684` | DTLS-secured CoAP port |
| `MAX_MESSAGE_SIZE` | `1024` | Maximum CoAP packet size (bytes) |
| `MAX_RESOURCE_BODY_SIZE` | `2048` | Maximum resource payload size (bytes) |
| `ACK_TIMEOUT` | `2000` | Acknowledgment timeout (ms) |

### Default Sampling Intervals

| Sensor | Default Interval | Configurable |
|---|---|---|
| Illumination Sensor | 30 seconds | ✅ via remote app option [2] |
| Soil Sensor | 30 seconds | ✅ via remote app option [3] |

---

## 📄 Documentation

Full technical documentation is available in [`CottonNet - Documentation.pdf`](CottonNet%20-%20Documentation.pdf), which covers the detailed system design, protocol specifications, and implementation details.

The LED state lookup table for the lighting controller is available in [`FASI ACCENSIONE LED`](FASI%20ACCENSIONE%20LED).

