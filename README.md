Got it 🚀 — you want a clean **GitHub README.md** that documents the **whole flow of creating, running, and deploying a simple CAP Java API**.
Here’s a nicely structured README you can drop directly into your repo:

---

````markdown
# 🚀 Simple CAP Java API

This project demonstrates how to build and deploy a **CAP (Cloud Application Programming) Java service** to **SAP BTP** with persistence in **SAP HANA Cloud** and authentication via **XSUAA**.

---

## 📌 Prerequisites

Ensure the following tools are installed and versions match:

```bash
java -version
# openjdk 17.0.16 (SapMachine)

mvn -version
# Apache Maven 3.9.x, Java 17 (SapMachine)

node -v
# v20.x

npm -v
# 10.x

cds --version
# @sap/cds: 9.x

cf --version
# cf version 8.x

mbt --version
# Cloud MTA Build Tool version 1.x
````

⚠️ The **Java version and Maven Java version must match** and use **SapMachine 17 LTS**.

---

## 📂 Project Setup

### 1. Initialize Project

```bash
cds init my-simple-api --add java
cd my-simple-api
```

### 2. Define the Data Model

Create `db/schema.cds`:

```cds
namespace my.simple.api;

entity Users {
  key ID : UUID;
  firstName  : String(50);
  lastName   : String(50);
  email      : String(100);
}
```

### 3. Define the Service

Create `srv/service.cds`:

```cds
using { my.simple.api as db } from '../db/schema';

service UserService {
    entity Users as projection on db.Users;
}
```

---

## ▶️ Run Locally

Start the service:

```bash
mvn spring-boot:run
```

Open [http://localhost:8080/odata/v4/UserService/Users](http://localhost:8080/odata/v4/UserService/Users)

You should see:

```json
{
  "@odata.context": "$metadata#Users",
  "value": []
}
```

Stop with `Ctrl + C`.

---

## ☁️ Prepare for BTP Deployment

Add deployment artifacts:

```bash
cds add mta
cds add hana
cds add xsuaa
```

Your `mta.yaml` will now orchestrate:

* `my-simple-api-srv` → Java CAP service
* `my-simple-api-db-deployer` → HANA Cloud schema deployer
* `my-simple-api-uaa` → Authentication service (XSUAA)

---

## 🚀 Build & Deploy

Build MTA:

```bash
mbt build
```

Login to Cloud Foundry:

```bash
cf login
```

Deploy to BTP:

```bash
cf deploy mta_archives/my-simple-api_1.0.0-SNAPSHOT.mtar
```

---

## ✅ Verify Deployment

In your **BTP Subaccount → Space → Instances** you should see:

* `my-simple-api-srv` → **started** (your live API)
* `my-simple-api-db-deployer` → **stopped** (expected; runs once to create tables)
* `my-simple-api-uaa` → authentication service

---

## 🔑 Authentication & Testing with Postman

### 1. Get Service Key

* In BTP Cockpit → Space → **Instances**
* Find `my-simple-api-uaa`
* Create **Service Key** (e.g. `postman-key`)
* Copy values:

  * `clientid`
  * `clientsecret`
  * `url`

---

### 2. Get Access Token

POST request in Postman:
`<url>/oauth/token`

**Auth:**

* Type: `Basic Auth`
* Username: `clientid`
* Password: `clientsecret`

**Body (x-www-form-urlencoded):**

```
grant_type=client_credentials
```

Response will contain:

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 43199,
  ...
}
```

---

### 3. Call the API

Create GET request in Postman:

```
https://<your-app>.cfapps.<region>.hana.ondemand.com/odata/v4/UserService/Users
```

**Auth:**

* Type: `Bearer Token`
* Token: `<access_token>`

✅ Response:

```json
{
  "@odata.context": "$metadata#Users",
  "value": []
}
```

---

## 🏆 Done!

Your CAP Java API is now:

* **Running locally**
* **Deployed to SAP BTP**
* **Protected by XSUAA**
* **Backed by HANA Cloud**

---

```

---

Would you like me to also **add the exact Postman collection JSON** (so you can just import it and test the API right away), or keep it only in instructions?
```
