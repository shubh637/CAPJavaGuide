This guide outlines the process of creating a simple CAP (Cloud Application Programming Model) Java API and deploying it to the SAP Business Technology Platform (BTP) using best practices.

## 🚀 CAP Java API Creation and BTP Deployment Guide

This guide details the steps to create a simple CAP Java service with a HANA database and secure it using XSUAA for deployment on SAP BTP.

-----

## 📋 1. Prerequisites Check

Before starting, ensure all necessary development tools are installed and correctly configured.

| Tool | Command | Expected Output Status |
| :--- | :--- | :--- |
| **Java** (JDK 17 LTS) | `java -version` | **PASS** (SapMachine 17.0.16) |
| **Maven** | `mvn -version` | **PASS** (Apache Maven 3.9.11) |
| **Node.js** | `node -v` | **PASS** (v20.19.5) |
| **npm** | `npm -v` | **PASS** (10.8.2) |
| **CAP CDS SDK** | `cds --version` | **PASS** (@sap/cds: 9.2.1) |
| **Cloud Foundry CLI** | `cf --version` | **PASS** (cf version 8.14.1) |
| **MTA Build Tool** | `mbt --version` | **PASS** (version 1.2.34) |

> **Note:** The Java and Maven Java versions are confirmed to be consistent and from a supported vendor (SAP SE), which is ideal for CAP Java development.

-----

## 🛠️ 2. Create Your CAP Project

Initialize a new CAP project and specify the **`java`** feature to set up the Spring Boot environment.

```bash
# Initialize the project and add the 'java' template
cds init my-simple-api --add java

# Navigate into the new project directory
cd my-simple-api
```

-----

## 📦 3. Define the Data Model and Service

### Data Model (`db/schema.cds`)

Create the data structure for the `Users` entity in the `db/schema.cds` file.

```cds
namespace my.simple.api;

entity Users {
  key ID         : UUID;
  firstName      : String(50);
  lastName       : String(50);
  email          : String(100);
}
```

### Service Definition (`srv/service.cds`)

Expose the `Users` entity via an OData service by creating the `srv/service.cds` file.

```cds
using { my.simple.api as db } from '../db/schema';

service UserService {
    entity Users as projection on db.Users;
}
```

-----

## 🧪 4. Run and Test Locally

Use Maven to compile and run the CAP Java service locally.

```bash
# Compile and run the Spring Boot application
mvn spring-boot:run
```

Once the application starts, open your browser or a tool like Postman and check the OData endpoint:

$$\text{http://localhost:8080/odata/v4/UserService/Users}$$

You should receive a successful $\text{JSON}$ response with an empty array:

```json
{
  "@odata.context": "$metadata#Users",
  "value": []
}
```

Press $\text{Ctrl} + \text{C}$ in the terminal to stop the running server.

-----

## ☁️ 5. Prepare for BTP Deployment

Configure the project for deployment to the SAP BTP Cloud Foundry environment using the **Multi-Target Application (MTA)** format.

> **Prerequisite:** Ensure you have an SAP BTP subaccount with the **SAP HANA Cloud (HANA plan)** subscribed, and a dedicated **space** created for deployment.

```bash
# Add the Multi-Target Application (MTA) file
cds add mta

# Add the HANA deployment configuration
cds add hana

# Add the XSUAA security configuration
cds add xsuaa
```

The `mta.yaml` file now includes configurations to orchestrate the creation of three essential BTP components:

1.  **`my-simple-api-srv`**: Your Java service application.
2.  **`my-simple-api-db-deployer`**: The app responsible for creating tables in the HANA database.
3.  **`my-simple-api-uaa`**: The XSUAA service instance for authentication and authorization.

-----

## 📤 6. Build and Deploy

### Build the MTA Archive

The `mbt build` command compiles the service and bundles all components into a deployable MTA archive (`.mtar`).

```bash
mbt build
```

The resulting file will be located in the `mta_archives` folder (e.g., `mta_archives/my-simple-api_1.0.0-SNAPSHOT.mtar`).

### Login to Cloud Foundry

Log in to your target BTP space using the Cloud Foundry command-line interface.

```bash
cf login
```

### Deploy the MTA

Use the Cloud Foundry CLI to deploy the generated MTA archive.

```bash
cf deploy mta_archives/my-simple-api_1.0.0-SNAPSHOT.mtar
```

This process may take several minutes as it creates the HANA, XSUAA, and deploys your service.

-----

## 🔎 7. Check BTP Space Status

After deployment, check your BTP Subaccount's Space for the deployed components:

| Component | Status | Description |
| :--- | :--- | :--- |
| `my-simple-api-srv` | **Started** (1/1) | Your live API service. |
| `my-simple-api-db-deployer` | **Stopped** (0/1) | **Correct and Expected.** This is a one-time task to create the database tables. Once it finishes its job, it stops automatically. |
| `my-simple-api-auth` | **Created** (Service Instance) | Your XSUAA authentication instance. |

-----

## 🔑 8. Testing (Secure API Access)

Since the service is secured by XSUAA, you must obtain an **Access Token** before calling the API.

### 1\. Get Your Credentials (Service Key) 🔑

1.  Navigate to your **BTP Cockpit** $\rightarrow$ **Space**.
2.  In the left menu, click **Instances**.
3.  Find your authentication instance, which should be named **`my-simple-api-auth`**.
4.  Click the three dots **(...)** and select **Create Service Key**.
5.  Name the key (e.g., `postman-key`) and click **Create**.
6.  View the key's $\text{JSON}$ content and extract these three values:
      * `clientid`
      * `clientsecret`
      * `url` (the authentication URL)

### 2\. Get the Access Token in Postman 🎟️

1.  Open **Postman** and create a new **POST** request.
2.  **URL:** Paste the `url` value from your service key and append $\mathbf{/oauth/token}$.
      * *Example:* `https://<...>.authentication.eu12.hana.ondemand.com/oauth/token`
3.  **Authorization Tab:**
      * Set **Type** to **Basic Auth**.
      * **Username:** Paste your `clientid`.
      * **Password:** Paste your `clientsecret`.
4.  **Body Tab:**
      * Select the **x-www-form-urlencoded** option.
      * Add a single entry: $\mathbf{KEY: \text{grant\_type}}$, $\mathbf{VALUE: \text{client\_credentials}}$.
5.  Click **Send**.
6.  In the response, copy the value of the $\mathbf{access\_token}$.

### 3\. Call Your API 🚀

1.  Create another new request in Postman, a **GET** request.
2.  **URL:** Paste your application route (from your BTP space) and append the service path.
      * *Example:* `https://<my-api-route>.cfapps.eu12.hana.ondemand.com/odata/v4/UserService/Users`
3.  **Authorization Tab:**
      * Set **Type** to **Bearer Token**.
      * In the **Token** field, paste the $\mathbf{access\_token}$ you copied.
4.  Click **Send**.

You should now receive a $\mathbf{200~OK}$ status and the empty $\text{JSON}$ array, confirming your deployed API is live, secure, and accessible with proper authorization.
