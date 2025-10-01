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



# Now getting the data in the React 
That's a comprehensive setup\! You've successfully integrated XSUAA authentication into a React Native frontend and implemented the necessary CORS/Security fixes on your CAP Java backend.

Here is the complete GitHub README file, including the file structure, setup instructions, and the code snippets for both the frontend and backend.

-----

# 🚀 Full-Stack CAP Java & React Native Integration

This project demonstrates a secure communication flow between a **React Native** (Frontend) client running on `localhost` and a **CAP Java Service** (Backend) deployed on SAP BTP Cloud Foundry, secured by **XSUAA** using the Client Credentials Flow.

## 📁 Project Structure

The solution involves changes in two main areas: the React Native project (for authentication and UI) and the CAP Java service (for security and CORS).

```
.
├── capJavaBackend/
│   ├── srv/
│   │   ├── src/
│   │   │   ├── main/
│   │   │   │   ├── java/
│   │   │   │   │   └── customer/my_api/
│   │   │   │   │       ├── WebConfig.java       <-- 1. CORS Policy
│   │   │   │   │       └── SecurityConfig.java  <-- 2. Security Bypass (Fixes 401)
│   │   │   │   └── resources/
│   │   │   │       └── application.yaml       <-- No security configs here
│   │   └── pom.xml                            <-- Spring Security dependencies added here
│   └── mta.yaml
└── reactNativeFrontend/
    └── myApp/
        ├── app/
        │   ├── index.js                     <-- 3. Main UI and API calls
        │   └── authService.js               <-- 4. XSUAA Token Retrieval (Client Credentials)
        ├── package.json                     <-- Dependencies: react-native, base-64
        └── xsuaaConfig.js                   <-- 5. Hardcoded Client Secrets (DEV ONLY)
```

## 1\. Backend Setup (CAP Java Service)

The Java service must be updated to resolve the Cross-Origin Resource Sharing (CORS) preflight $\mathbf{401}$ error encountered when connecting from `localhost`.

### A. Dependencies (`srv/pom.xml`)

Ensure you have the required Spring Security and CAP Security starters:

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>com.sap.cds</groupId>
        <artifactId>cds-starter-spring-security</artifactId>
    </dependency>
    </dependencies>
```

### B. Security Bypass (`srv/src/main/java/customer/my_api/SecurityConfig.java`)(don't do it if don't want to bypass the security).

This is the **CRITICAL FIX**. It allows the unauthenticated $\text{OPTIONS}$ request to pass the XSUAA filter.

```java
package customer.my_api;

import org.springframework.context.annotation.Configuration;
// ... other imports for SecurityFilterChain, HttpMethod, etc.

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Use requestMatchers for Spring Security 6+ syntax
                .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll() 
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2.jwt())
            .cors() // Integrate the WebConfig bean
            .and().csrf().disable();
            
        return http.build();
    }
}
```

### C. CORS Policy (`srv/src/main/java/customer/my_api/WebConfig.java`)

This defines which origins (`localhost:8081`) are allowed access.

```java
package customer.my_api;

import org.springframework.context.annotation.Configuration;
// ... imports

@Configuration
public class WebConfig {
 
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(Arrays.asList(
            "http://localhost:8081",
            "http://localhost:19006",
            // Replace with your exact deployed service URL
            "https://jps-hpwyvuj0-space1-my-api-srv.cfapps.eu12.hana.ondemand.com"
        ));
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "HEAD", "OPTIONS"));
        configuration.setAllowedHeaders(Arrays.asList("*")); 
        configuration.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

### D. Deployment

1.  Build the project: `mvn clean package -DskipTests=true`
2.  Redeploy the MTA: `mbt build` then `cf deploy <archive>.mtar`

-----

## 2\. Frontend Setup (React Native App)

The frontend requires two files for authentication and the main UI logic.

### A. Configuration (`myApp/xsuaaConfig.js`)

**⚠️ Development Only:** Never use this in production. Get these values from your XSUAA Service Key.

```javascript
// myApp/xsuaaConfig.js
const XSUAA_CONFIG = {
  tokenUrl: "https://<your-xsuaa-host>/oauth/token", 
  clientId: "<your-clientid>", 
  clientSecret: "<your-clientsecret>"
};
export default XSUAA_CONFIG;
```

### B. Authentication Service (`myApp/authService.js`)

This file fetches the XSUAA token using the Client Credentials Grant Flow.

```javascript
// myApp/authService.js
import base64 from 'base-64'; 
import XSUAA_CONFIG from './xsuaaConfig'; 

const { tokenUrl, clientId, clientSecret } = XSUAA_CONFIG;

export const getAccessToken = async () => {
  try {
    // 1. Prepare Basic Auth header
    const credentials = `${clientId}:${clientSecret}`;
    // Use base64.encode() if you imported the library, or btoa() if available globally
    const encodedCredentials = base64.encode(credentials); 

    const response = await fetch(tokenUrl, {
      method: 'POST',
      headers: {
        // 2. Set Basic Auth and Content Type
        'Authorization': `Basic ${encodedCredentials}`,
        'Content-Type': 'application/x-www-form-urlencoded',
      },
      // 3. Set grant type for Client Credentials Flow
      body: 'grant_type=client_credentials',
    });

    const json = await response.json();

    // Check for HTTP errors before trying to read the token
    if (!response.ok) {
        // This captures errors like 401 Bad Credentials, 400 Bad Request, etc.
        throw new Error(`HTTP Error: ${response.status} - ${json.error_description || json.error || 'Unknown error'}`);
    }
 
    if (json.access_token) {
      console.log('Successfully fetched new access token!');
      return json.access_token;
    } else {
      // This is the fallback if response.ok is true but token is missing
      throw new Error('Failed to fetch access token: Token field missing in response.');
    }

  } catch (error) {
    console.error("Error fetching access token:", error.message || error);
    // Returning null will cause the subsequent API call to fail, which is correct behavior
    return null; 
  }
};
```

### C. Main UI (`myApp/app/index.js`)

This is your main React Native component, which initializes the token and performs the secured $\text{GET}$ and $\text{POST}$ requests.

```javascript
// myApp/app/index.js

import React, { useState, useEffect } from 'react';
// ... other imports

import { getAccessToken } from '../authService'; 
 
const API_URL = 'https://jps-hpwyvuj0-space1-my-api-srv.cfapps.eu12.hana.ondemand.com/odata/v4/UserService/Users';

export default function HomeScreen() {
  const [token, setToken] = useState(null);
  const [loading, setLoading] = useState(true);
  // ... rest of state variables

  useEffect(() => {
    // Initialization logic to fetch token
    const initialize = async () => {
      const fetchedToken = await getAccessToken();
      if (fetchedToken) {
        setToken(fetchedToken);
      } else {
        Alert.alert('Authentication Failed', 'Could not retrieve access token.');
        setLoading(false);
      }
    };
    initialize();
  }, []);
 
  useEffect(() => {
    // Fetch users after token is successfully set
    if (token) {
      fetchUsers();
    }
  }, [token]);
 
  const fetchUsers = async () => {
    // ... (Your fetchUsers logic using Authorization: Bearer ${token})
  };
 
  const handleSaveUser = async () => {
    // ... (Your handleSaveUser logic using Authorization: Bearer ${token})
  };
 
  return (
    // ... (Your JSX component rendering form and list)
  );
}
// ... (Your StyleSheet.create)
```
