# LoRaWAN-Based Vehicle Emission Monitoring System

A vehicle emission monitoring and analytics project developed for a
5G/IoT hackathon. The project explores a system in which an IoT device
installed in a vehicle transmits pollutant measurements over a LoRaWAN
network. An MQTT consumer receives device uplinks, associates each
device with a registered vehicle, and stores measurements in PostgreSQL.
A React dashboard presents emission summaries and analytics for
administrators, RTO authorities, and vehicle owners.

> **Implementation note:** This README describes the code available in
> this repository. The repository contains the Node.js backend, MQTT
> consumer, PostgreSQL integration, and React frontend. It does not
> contain the vehicle-mounted hardware firmware, sensor wiring details,
> or the SQL database schema. Some values in the MQTT consumer are
> currently hard-coded placeholders; see [Known
> limitations](#known-limitations) before interpreting the data as real
> sensor measurements.

## Table of contents

-   [Project overview](#project-overview)
-   [System architecture](#system-architecture)
-   [Data flow](#data-flow)
-   [Features](#features)
-   [Pollutants monitored](#pollutants-monitored)
-   [Technology stack](#technology-stack)
-   [Repository structure](#repository-structure)
-   [Backend and MQTT integration](#backend-and-mqtt-integration)
-   [Database design](#database-design)
-   [REST API reference](#rest-api-reference)
-   [Frontend dashboards](#frontend-dashboards)
-   [Installation and setup](#installation-and-setup)
-   [Running the application](#running-the-application)
-   [Configuration](#configuration)
-   [Known limitations](#known-limitations)
-   [Future improvements](#future-improvements)

## Project overview

The goal of the project is to monitor vehicle emissions using IoT
devices installed in vehicles and make the collected information
available through a centralized dashboard.

The intended system associates each transmitting device with a vehicle.
When a device sends an uplink message through the LoRaWAN network, the
backend receives the message through MQTT, identifies the device using
its Device EUI (`devEUI`), finds the associated vehicle, and stores a
pollutant measurement record.

The web application provides different views for:

-   **Administrators:** Aggregate emission analytics and vehicle
    categories with measurements exceeding the system's threshold logic.
-   **RTO authorities:** A dashboard for reviewing vehicle emission
    information and threshold-crossing details.
-   **Vehicle owners:** Emission summaries and historical trends for
    vehicles associated with their account.

The repository is named `5G_hackathon`, while the communication
integration present in the source code uses LoRaWAN uplinks delivered
through MQTT.

## System architecture

``` text
+-----------------------------+
| Vehicle-mounted IoT device  |
| Pollutant sensors           |
+--------------+--------------+
               |
               | LoRaWAN uplink
               v
+-----------------------------+
| LoRaWAN network / gateway   |
+--------------+--------------+
               |
               | MQTT uplink message
               v
+-----------------------------+
| MQTT broker                 |
| lora.nebulae.io             |
+--------------+--------------+
               |
               v
+-----------------------------+
| Node.js MQTT consumer       |
| Backend/mqtt.js             |
| - Receives uplink messages  |
| - Decodes payload           |
| - Maps devEUI to vehicle    |
+--------------+--------------+
               |
               | SQL INSERT
               v
+-----------------------------+
| PostgreSQL                  |
| Vehicle and pollutant data  |
+--------------+--------------+
               |
               | SQL queries
               v
+-----------------------------+
| Express REST API            |
| Backend/app.js              |
| Authentication & analytics  |
+--------------+--------------+
               |
               | HTTP / JSON
               v
+-----------------------------+
| React dashboard             |
| Admin / RTO / User views    |
+-----------------------------+
```

The MQTT consumer and Express API are separate Node.js processes in the
current repository. The API server does not automatically start the MQTT
consumer.

## Data flow

The following flow is supported by the MQTT consumer and backend
implementation:

1.  **Device transmission:** A vehicle-mounted device is intended to
    collect pollutant measurements and transmit an uplink through the
    LoRaWAN network. The device firmware and sensor-acquisition code are
    not included in this repository.
2.  **MQTT delivery:** The MQTT client connects to `lora.nebulae.io` and
    subscribes to `application/87/device/+/uplink`.
3.  **Device identification:** The consumer reads the `devEUI` from the
    incoming message and queries the PostgreSQL `device` table to find
    the corresponding `vehicle_id`.
4.  **Payload conversion:** The consumer converts the message's Base64
    `data` field into hexadecimal and then into a decimal value.
5.  **Database insertion:** The consumer inserts a row into
    `pollutant_measurement`, associating the measurement with the
    vehicle, date, gateway ID, pollutant fields, and measurement time.
6.  **Analytics:** The Express API executes SQL queries over vehicle and
    pollutant records to calculate emission summaries, daily trends, and
    threshold-crossing counts.
7.  **Dashboard display:** The React frontend calls the API and renders
    the returned data using tables, statistic cards, and charts.

The precise hardware sampling interval, sensor models, LoRaWAN device
configuration, payload encoding specification, and gateway deployment
are not documented in the available source.

## Features

-   MQTT subscription to LoRaWAN device uplink messages.
-   Device-to-vehicle association using the device EUI.
-   PostgreSQL storage for vehicle pollutant measurements.
-   Role-based login flow for `USER`, `ADMIN`, and `RTO` roles.
-   Aggregate counts of vehicles with pollutant measurements above
    database-wide averages.
-   Vehicle-category breakdown for two-wheelers, four-wheelers, and
    heavy vehicles.
-   Detailed listing of vehicles whose measurements cross the
    implemented thresholds.
-   Daily pollutant distribution visualization.
-   Weekly pollutant trend visualization.
-   Vehicle-owner dashboard with per-vehicle summaries and historical
    charts.

## Pollutants monitored

The backend and frontend use the following pollutant fields:

  Field          Description in the application
  -------------- -------------------------------------
  `co_value`     Carbon monoxide (CO) measurement
  `no_value`     Nitrogen oxide (NO) measurement
  `hcho_value`   Formaldehyde (HCHO) measurement
  `pm`           Particulate matter (PM) measurement

The code does not document measurement units, sensor calibration, or the
specific PM size fraction. Those details should be added only after
confirming the hardware and sensor specifications.

## Technology stack

  Layer                  Technologies
  ---------------------- ------------------------------------------------------------
  Device communication   LoRaWAN uplinks and MQTT
  MQTT broker            `lora.nebulae.io`
  Backend API            Node.js, Express.js
  MQTT client            MQTT.js
  Database               PostgreSQL
  Database driver        `pg`
  Authentication         JSON Web Tokens (JWT)
  Frontend               React 18
  UI components          Material UI (MUI)
  Charts                 Nivo (`@nivo/bar`, `@nivo/line`, `@nivo/pie`, `@nivo/geo`)
  HTTP client            Axios
  Forms                  Formik and Yup

## Repository structure

``` text
5G_hackathon/
├── Backend/
│   ├── app.js
│   ├── db.js
│   ├── mqtt.js
│   ├── package.json
│   ├── controllers/
│   │   └── controller.js
│   ├── errors/
│   ├── middleware/
│   └── routes/
│       └── routes.js
│
└── Frontend/
    ├── package.json
    ├── public/
    └── src/
        ├── App.js
        ├── components/
        │   ├── BarChart.jsx
        │   ├── GeographyChart.jsx
        │   ├── LineChart.jsx
        │   ├── PieChart.jsx
        │   ├── Thresholdcrossedtable.js
        │   └── Top10table.js
        ├── data/
        ├── scenes/
        │   ├── dashboard/
        │   ├── userdashboard/
        │   ├── login/
        │   ├── bar/
        │   ├── line/
        │   ├── pie/
        │   ├── geography/
        │   └── thresholdcrossed/
        └── theme.js
```

`Backend/node_modules/` is included in the supplied archive but is
generated dependency content, not application source. It is omitted from
the structure above.

## Backend and MQTT integration

### Express API

The Express application is defined in `Backend/app.js` and listens on
port `3002`:

``` bash
cd Backend
npm install
npm start
```

The API routes are mounted under:

``` text
/api/v1/views/
```

### MQTT consumer

The MQTT integration is implemented separately in `Backend/mqtt.js`.

The current source configures:

-   Broker host: `lora.nebulae.io`
-   Protocol: MQTT
-   Port: `2096`
-   Subscription topic: `application/87/device/+/uplink`
-   Device identifier: `devEUI`
-   Payload field: `data` (Base64-encoded)

The consumer listens for messages, parses the JSON payload, converts the
Base64 data to a decimal value, looks up the vehicle associated with the
device EUI, and inserts a measurement into PostgreSQL.

Run it in a separate terminal:

``` bash
cd Backend
node mqtt.js
```

The MQTT broker address, port, topic, client ID, and credentials are
currently defined in the source file. Move these settings to environment
variables before deploying the application.

### Payload processing caveat

The current consumer does not decode four independent pollutant
measurements from the payload. It converts the complete `data` field to
one decimal value and uses that value for `co_value`. The values
inserted for `hcho_value`, `no_value`, and `pm` are constants in the
current implementation.

Consequently, the repository demonstrates an MQTT ingestion and database
insertion flow, but the available source does not establish a complete
production-ready multi-pollutant sensor decoder.

## Database design

The backend connects to a PostgreSQL database named `hackathon`. SQL
queries in the controllers and MQTT consumer reference the following
tables:

  -----------------------------------------------------------------------
  Table                               Purpose inferred from backend
                                      queries
  ----------------------------------- -----------------------------------
  `owner`                             Stores owner/authority information
                                      and is joined to user and vehicle
                                      records

  `role`                              Maps users to application roles

  `myuser`                            Stores login credentials and owner
                                      association

  `vehicle`                           Stores vehicle information, owner
                                      association, registration number,
                                      company, fuel type, and vehicle
                                      type

  `device`                            Maps a device EUI to a vehicle ID

  `pollutant_measurement`             Stores pollutant measurements,
                                      vehicle ID, gateway ID, date, and
                                      time
  -----------------------------------------------------------------------

The repository does not include a SQL schema or database migration file.
The table and column descriptions above are inferred from the SQL
statements in the backend; the complete schema, constraints, indexes,
and relationships must be recovered from the original database or
created separately.

## REST API reference

All routes below are mounted under `/api/v1/views`.

  ----------------------------------------------------------------------------------------------
  Method                  Endpoint                                       Purpose
  ----------------------- ---------------------------------------------- -----------------------
  `GET`                   `/`                                            Returns a test response
                                                                         containing measurement
                                                                         data for a fixed date
                                                                         range

  `POST`                  `/login`                                       Authenticates a user
                                                                         and returns a JWT and
                                                                         role

  `GET`                   `/getFailCounts`                               Returns counts of
                                                                         vehicles with
                                                                         measurements above the
                                                                         implemented averages,
                                                                         grouped by vehicle type

  `GET`                   `/getFailedDetails/:category`                  Returns vehicle and
                                                                         owner details for
                                                                         records in a vehicle
                                                                         category that cross the
                                                                         implemented thresholds

  `GET`                   `/getFailedDayWiseBarData`                     Returns day-wise counts
                                                                         of vehicles crossing
                                                                         thresholds

  `GET`                   `/dailyEmmitedGasPie`                          Returns the daily
                                                                         proportion of CO, NO,
                                                                         HCHO, and PM totals

  `GET`                   `/totalGasEmmitedPerDayPerUser`                Returns today's
                                                                         pollutant averages and
                                                                         recent trends for the
                                                                         authenticated user's
                                                                         vehicles

  `GET`                   `/getAverageEmmisionDayWiseLineChartData`      Returns overall daily
                                                                         pollutant averages for
                                                                         the recent seven-day
                                                                         period

  `GET`                   `/getAverageEmmisionDayWiseLineChartForUser`   Returns daily pollutant
                                                                         averages for the
                                                                         authenticated user's
                                                                         vehicles

  `GET`                   `/getUserName`                                 Returns the
                                                                         authenticated user's
                                                                         owner record

  `GET`                   `/getRTOName`                                  Returns the
                                                                         authenticated RTO
                                                                         owner's record

  `GET`                   `/getAdminName`                                Returns the
                                                                         authenticated admin
                                                                         owner's record
  ----------------------------------------------------------------------------------------------

The authenticated user-specific endpoints use JWT middleware. The token
is returned by `/login` and is expected in the `Authorization` header:

``` text
Authorization: Bearer <JWT>
```

The login endpoint returns a role value of `USER`, `ADMIN`, or `RTO`,
which the frontend uses to direct the user to the corresponding
dashboard.

## Frontend dashboards

The React application is located in `Frontend/` and uses React Router
for navigation.

### Administrator / RTO views

-   Dashboard summary of vehicles crossing the implemented pollutant
    thresholds.
-   Counts grouped by vehicle category: `Four-Wheeler`, `Two-Wheeler`,
    and `Heavy-vehicles`.
-   Threshold-crossing vehicle details, including owner, registration
    number, company, fuel type, and pollutant values.
-   Bar chart for day-wise threshold-crossing counts.
-   Pie chart showing the daily distribution of pollutant totals.
-   Line chart showing average pollutant measurements over recent days.

### Vehicle owner view

-   Vehicle-wise pollutant summary for the current day.
-   Recent trend charts for CO, NO, HCHO, and PM.
-   User-specific data retrieved through authenticated API endpoints.

### Static/demo dashboard elements

Some frontend elements are not connected to live database data:

-   The **Top 10 Pollutant Areas** table uses hard-coded sample rows.
-   The geography visualization uses mock geographic data.
-   Several chart datasets in `Frontend/src/data/mockData.js` are demo
    data and are not evidence of actual sensor readings.

## Installation and setup

### Prerequisites

-   Node.js and npm
-   PostgreSQL
-   A PostgreSQL database named `hackathon` with the required tables and
    columns
-   Access to the configured MQTT broker and LoRaWAN application, if
    running the ingestion component
-   The vehicle-mounted device and LoRaWAN network configuration, if
    testing end-to-end hardware ingestion

### 1. Clone the repository

``` bash
git clone https://github.com/shahkandarp/5G_hackathon.git
cd 5G_hackathon
```

### 2. Set up PostgreSQL

Create the database:

``` sql
CREATE DATABASE hackathon;
```

The application expects the tables and columns referenced in the SQL
queries in `Backend/controllers/controller.js` and `Backend/mqtt.js`.

**Important:** The repository does not provide the SQL schema or seed
data. Recover the original schema or create the required tables before
starting the backend.

### 3. Configure the backend

``` bash
cd Backend
npm install
```

Create a `.env` file for JWT configuration:

``` env
JWT_SECRET_USER=<strong-user-jwt-secret>
JWT_SECRET_ADMIN=<strong-admin-jwt-secret>
JWT_SECRET_RTO=<strong-rto-jwt-secret>
JWT_LIFETIME=30d
```

The current PostgreSQL connection settings are hard-coded in
`Backend/db.js`:

``` text
host: localhost
database: hackathon
port: 5432
```

Update the database configuration to use your PostgreSQL username and
password. Prefer reading these values from environment variables rather
than keeping credentials in source code.

### 4. Set up the frontend

Open a separate terminal:

``` bash
cd Frontend
npm install
```

The frontend currently sends API requests to `http://127.0.0.1:3002`.
Update these URLs if the backend is hosted elsewhere. For a maintainable
deployment, configure a single frontend API base URL through an
environment variable.

## Running the application

The backend API and MQTT consumer must be started separately.

**Terminal 1 --- Express API**

``` bash
cd Backend
npm start
```

The API listens on:

``` text
http://localhost:3002
```

**Terminal 2 --- MQTT consumer**

``` bash
cd Backend
node mqtt.js
```

This process connects to the configured MQTT broker and listens for
LoRaWAN uplink messages.

**Terminal 3 --- React frontend**

``` bash
cd Frontend
npm start
```

The React development server normally starts at:

``` text
http://localhost:3000
```

For end-to-end data flow, the PostgreSQL schema, MQTT broker
configuration, device-to-vehicle mapping, and compatible uplink payload
must all be in place.

## Configuration

The current source contains configuration values in multiple places:

  -----------------------------------------------------------------------
  File                                Current configuration
  ----------------------------------- -----------------------------------
  `Backend/db.js`                     PostgreSQL host, database,
                                      username, password, and port

  `Backend/.env`                      JWT secrets and token lifetime

  `Backend/mqtt.js`                   MQTT broker, port, topic, username,
                                      password, and client ID

  Frontend source files               Hard-coded backend API base URL
  -----------------------------------------------------------------------

For deployment, move sensitive and environment-specific values into
environment variables, and ensure that `.env` files are excluded from
version control.

## Known limitations

The following limitations are visible in the supplied repository:

1.  **Hardware firmware is absent.** There is no source for the
    vehicle-mounted device, sensor acquisition, or LoRaWAN radio
    configuration.
2.  **Database schema is absent.** The backend expects existing
    PostgreSQL tables, but no schema or migration scripts are included.
3.  **Partial payload decoding:** The MQTT consumer converts the Base64
    payload into a single decimal value. It does not parse separate CO,
    NO, HCHO, and PM fields.
4.  **Placeholder pollutant values:** HCHO, NO, and PM are inserted as
    fixed constants in the current MQTT consumer. These should not be
    interpreted as actual sensor measurements.
5.  **Separate MQTT process:** The MQTT listener is not started by the
    Express application. It must be run independently.
6.  **Threshold logic uses database averages:** The current
    implementation compares measurements with overall database averages
    rather than documented regulatory emission limits.
7.  **Static dashboard data:** The Top 10 Pollutant Areas table and
    geography visualization use mock/sample data.
8.  **Configuration is hard-coded:** Database credentials, MQTT
    settings, and frontend API URLs need environment-based
    configuration.
9.  **Measurement metadata is incomplete:** Sensor units, calibration,
    sampling frequency, and payload format are not documented.

## Future improvements

-   Implement and document the vehicle-mounted sensor hardware and
    firmware.
-   Define a versioned payload format and decode each pollutant
    measurement independently.
-   Replace hard-coded pollutant values with actual sensor readings.
-   Add PostgreSQL schema and migration scripts.
-   Use official emission thresholds and clearly document units and
    comparison rules.
-   Connect the Top 10 Pollutant Areas and geography views to real
    location-based data.
-   Add device health monitoring, message validation, and missing-data
    handling.
-   Add automated tests for MQTT ingestion, API routes, authentication,
    and analytics.
-   Centralize configuration and add deployment instructions for the
    MQTT consumer, API, database, and frontend.

## License

No project-level license file was present in the inspected repository.
Add a `LICENSE` file if you intend to specify terms for using,
modifying, and distributing this project.
