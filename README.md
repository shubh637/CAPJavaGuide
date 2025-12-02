# Steps to deploy a role based app in BTP with a React UI

<aside>
📎

This guide is based on capire documentation : https://cap.cloud.sap/docs/guides/deployment/to-cf#cf-cli

</aside>

## Requirements

1. S4 Hana Cloud instance available in your space where you will deploy your application 

![Screenshot 2025-11-16 at 8.07.54 PM.png](https://github.com/shubh637/CAPJavaGuide/blob/main/Screenshot2025-11-16at8.07.54P.jpeg)

## Set Up

Open the folder where you want your project to be set up in VS code and use the following commands

```bash
cds init <app-name> --add java 
cd bookshop
**npm i -g @sap/cds-dk 
npm i -g mbt
cf add-plugin-repo CF-Community https://plugins.cloudfoundry.org
cf install-plugin -f multiapps
cf install-plugin -f html5-plugin**
```

## Prepare for Deployment

```bash
cds add hana
cds add xsuaa
cds add mta
cds add approuter
cds add app-front
```

go to app/router/ and create a resources folder.

update app/ui/frontend/package.json

```bash
  .......
  .......
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build && npm run copy-build",
    "copy-build": "rm -rf ../../router/resources && mkdir -p ../../router/resources && cp -r build/* ../../router/resources/",
    ....
    ....
    ...
    }
```

update package.json in root folder

```bash
"scripts": {
    "stage":"cd app/ui/frontend && npm run build && cd ../../..",
     "deploy": "mbt build && cf deploy mta_archives/*.mtar --retries 1"
  }
```

Now make changes in the xs-security.json in root directory

```jsx
{
  "scopes": [
    {
      "name": "$XSAPPNAME.admin",
      "description": "admin"
    }
  ],
  "attributes": [],
  "role-templates": [
    {
      "name": "admin",
      "description": "generated",
      "scope-references": [
        "$XSAPPNAME.admin"
      ],
      "attribute-references": []
    }
  ]
}

```

go to app/router/xs-app.json and make changes

```jsx
{
  "welcomeFile": "/index.html",
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/odata/v4/(.*)$",
      "target": "/odata/v4/$1",
      "destination": "hpsmt-srv-api",//set this according to the mta.yaml file
      "authenticationType": "xsuaa",
      "csrfProtection":true
    },
    {
      "source": "^/user-api/currentUser$",
      "target": "/user-api/currentUser",
      "service": "sap-approuter-userapi",
      "authenticationType": "xsuaa"
    },
    {
      "source": "^(.*)$",
      "localDir": "resources"
    }    
    
    
    
  ]
}

```

## Modify

We don’t need the complete sample, so we will start by making it simpler 

1. Change db/schema.cds

```bash
namespace my.hpsmt;
using { managed } from '@sap/cds/common';

entity Users:managed{
  key ID : UUID;
  name : String;
  email : String;
  role : String; // 'User' or 'Manager'
  isActive : Boolean;
}

entity Teams:managed{
  key ID : UUID;
  name : String;
  description : String;
  manager : Association to Users;
  
}

entity TeamMembers{
  key ID : UUID;
  team : Association to Teams;
  user : Association to Users;
}
```

1. Change srv/admin-service.cds

```bash
using {my.hpsmt as h} from '../db/schema';
service AdminService @(requires : 'admin'){
    entity Users as projection on h.Users;
    entity Teams as projection on h.Teams;
    entity TeamMembers as projection on h.TeamMembers;
}
```

1. Change srv/core-service.cds

```bash
using {my.hpsmt as h} from '../db/schema';
service CoreService{
    @readonly
    entity Users as projection on h.Users;
    @readonly
    entity Teams as projection on h.Teams;
    @readonly
    entity TeamMembers as projection on h.TeamMembers;
}
```

## Create a Frontend

```bash
mkdir app/ui
cd app/ui
npx create-react-app frontend
```

This will create a frontend starting template.

Create file app/ui/frontend/src/GlobalConstant.js

this file is to create the csrf token and use it globally

```jsx

import { createContext, useContext, useState } from "react";

//creating the context
const AppContext = createContext(null);

//exporting so we can use AppContext.
export const useAppContext = () => useContext(AppContext);

export const AppProvider = ({ children }) => {
  const [token, setToken] = useState(null);

  const ADMIN_BASE = "/odata/v4/AdminService";//base url for the csrf token generation chaneg according to service..
  const fetchCsrfToken = async () => {
    const res = await fetch(`${ADMIN_BASE}/`, {
      method: "GET",
      credentials: "include",
      headers: { "X-CSRF-Token": "Fetch" },
    });

    if (!res.ok) throw new Error("Failed to fetch CSRF token");

    const csrf = res.headers.get("X-CSRF-Token");
    if (!csrf) throw new Error("No CSRF token returned");

    setToken(csrf);
    return csrf;
  };

  return (
    <AppContext.Provider
      value={{ token, setToken, ADMIN_BASE, fetchCsrfToken }}
    >
      {children}
    </AppContext.Provider>
  );
};

```

go to app/ui/frontend/src/index.js

```jsx
import React from "react";
import ReactDOM from "react-dom/client";
import "./index.css";
import App from "./App";
import reportWebVitals from "./reportWebVitals";
import { AppProvider } from "./GlobalConstant";

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <React.StrictMode>
    <AppProvider> //wrap the main App component with the AppProvider to access global variable..
      <App />
    </AppProvider>
  </React.StrictMode>
);

reportWebVitals();
```

Edit the frontend/src/App.js

```jsx
import React, { useState, useEffect } from "react";
import { FontAwesomeIcon } from "@fortawesome/react-fontawesome";
import { faPen, faTrash } from "@fortawesome/free-solid-svg-icons";
import { useAppContext } from "./GlobalConstant";//import the global constants.
import "./App.css";

const App = () => {
  const ADMIN_API = "/odata/v4/AdminService"; 
  const USER_API = "/odata/v4/CoreService";
  const { token, fetchCsrfToken } = useAppContext();//using token from global constant.

  const [userInfo, setUserInfo] = useState(null);//store current user information.
  const [admin, setAdmin] = useState(false);
  const [editingId, setEditingId] = useState(null);

  const [form, setForm] = useState({
    name: "",
    email: "",
    role: "USER",
    isActive: "true",
  });
  
   const [loading, setLoading] = useState(false);
  const [message, setMessage] = useState(null);

  const [users, setUsers] = useState([]);
  const [listLoading, setListLoading] = useState(true);
  const [listError, setListError] = useState(null);

  const ROLES = [
    { value: "USER", label: "User" },
    { value: "MANAGER", label: "Manager" },
   
  ];
  
  //---------------------------CHECK ADMIN--------------------------------------
  // Check if user is admin
  const checkAdmin = async () => {
    try {
      const res = await fetch(`${ADMIN_API}/Users?$top=1`, {
        credentials: "include",
      });
      setAdmin(res.ok);
    } catch {
      setAdmin(false);
    }
  };
  
  //---------------------------FETCH CURRENT USER INFO--------------------------------------
  
  const fetchCurrentUser = async () => {
    try {
      const res = await fetch("/user-api/currentUser", {
        credentials: "include",
      });
      if (res.ok) setUserInfo(await res.json());
    } catch (err) {
      console.error(err);
    }
  };  
  
  //---------------------------FETCH USERS--------------------------------------
  
  const fetchUsers = async () => {
    setListLoading(true);
    try {
      const res = await fetch(`${USER_API}/Users`, { credentials: "include" });
      const data = await res.json();
      setUsers((data.value || []).sort((a, b) => b.ID - a.ID));
    } catch (err) {
      setListError(err.message);
    } finally {
      setListLoading(false);
    }
  };

  useEffect(() => {
    fetchUsers();
    fetchCurrentUser();
    fetchCsrfToken();
    checkAdmin();
  }, []);
  
  
   //---------------------------POST DATA TO USERS--------------------------------------
  
  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm((prev) => ({ ...prev, [name]: value }));
  };

  // Create / Update User
  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setMessage(null);

    const payload = { ...form, isActive: form.isActive === "true" };

    try {
      const method = editingId ? "PATCH" : "POST";
      const url = editingId
        ? `${ADMIN_API}/Users(${editingId})`
        : `${ADMIN_API}/Users`;

      const res = await fetch(url, {
        method,
        headers: {
          "Content-Type": "application/json",
          "X-CSRF-Token": token,
        },
        body: JSON.stringify(payload),
      });

      //if (!res.ok) throw new Error("Failed to save user");
      if (!res.ok) {
        let errorMessage = "Failed to save user";

        try {
          const errorBody = await res.json();
          if (errorBody?.error?.message) {
            errorMessage = errorBody.error.message;
          }
        } catch (e) {
          // ignore JSON parse error
        }

        throw new Error(errorMessage);
      }
      setMessage({
        type: "success",
        text: editingId ? "User Updated" : "User Created",
      });

      fetchUsers();
      setEditingId(null);
      setForm({ name: "", email: "", role: "USER", isActive: "true" });
    } catch (err) {
      setMessage({ type: "error", text: err.message });
    } finally {
      setLoading(false);
    }
  };  
  
  
  
    //--------------------------DELETE USER----------------------------------
  const deleteUser = async (id) => {
    if (!window.confirm("Delete this user?")) return;
    try {
      const res = await fetch(`${ADMIN_API}/Users(${id})`, {
        method: "DELETE",
        headers: { "X-CSRF-Token": token },
      });
      if (!res.ok) throw new Error("Delete failed");
      fetchUsers();
    } catch (err) {
      setMessage({ type: "error", text: err.message });
    }
  };
  
  //---------------------------Edit user (prefill form)-------------------------------
  const startEdit = (u) => {
    setEditingId(u.ID);
    setForm({
      name: u.name,
      email: u.email,
      role: u.role,
      isActive: u.isActive ? "true" : "false",
    });
  };

  // UI: Message
  const MessageDisplay = ({ message }) => {
    if (!message) return null;
    return (
      <div
        className={`msg ${message.type === "success" ? "msg-success" : "msg-error"
          }`}
      >
        {message.text}
      </div>
    );
  };

  //--------------------------------- LIST--------------------------------------
  const UserList = () => {
    if (listLoading) return <div className="info">Loading users...</div>;
    if (listError) return <div className="msg msg-error">{listError}</div>;
    if (users.length === 0)
      return <div className="info">No users available.</div>;

    return (
      <div className="user-list">
        {users.map((u) => (
          <div key={u.ID} className="user-card">
            <div className="user-top">
              <span className="user-name">{u.name}</span>
              <span
                className={`user-status ${u.isActive ? "active" : "inactive"}`}
              >
                {u.isActive ? "Active" : "Inactive"}
              </span>
            </div>

            <p className="email">{u.email}</p>

            <div className="user-footer">
              <span className="role">{u.role}</span>
            </div>

            {admin && (
              <div className="action-row">
                <button className="edit-btn" onClick={() => startEdit(u)}>
                  <FontAwesomeIcon icon={faPen} />
                </button>
                <button className="delete-btn" onClick={() => deleteUser(u.ID)}>
                  <FontAwesomeIcon icon={faTrash} />
                </button>
              </div>
            )}
          </div>
        ))}
      </div>
    );
  };

  return (
    <div className={`page ${!admin ? "not-admin" : ""}`}>
      <div className="container">
        <header className="header">
          <h1>HPSMT</h1>
          <p className="welcome">Hello, {userInfo?.firstname ?? "Guest"}!</p>
        </header>

        <div className="grid">
          {admin && (
            <div className="card">
              <h2>{editingId ? "Edit User" : "Add New User"}</h2>

              <MessageDisplay message={message} />

              <form onSubmit={handleSubmit}>
                <label>Name</label>
                <input name="name" value={form.name} onChange={handleChange} />

                <label>Email</label>
                <input
                  name="email"
                  type="email"
                  value={form.email}
                  onChange={handleChange}
                />

                <label>Role</label>
                <select name="role" value={form.role} onChange={handleChange}>
                  {ROLES.map((r) => (
                    <option value={r.value} key={r.value}>
                      {r.label}
                    </option>
                  ))}
                </select>

                <label>Status</label>
                <select
                  name="isActive"
                  value={form.isActive}
                  onChange={handleChange}
                >
                  <option value="true">Active</option>
                  <option value="false">Inactive</option>
                </select>

                <button disabled={loading}>
                  {loading
                    ? "Saving..."
                    : editingId
                      ? "Update User"
                      : "Create User"}
                </button>
              </form>
            </div>
          )}
          <div className="card">
            <h2>Existing Users ({users.length})</h2>
            <UserList />
          </div>

        </div>

      </div>
    </div>
  );
};

export default App;

```

**NOTE**:  ADD CSS ACCORDING TO YOU

```jsx
/* Page */
.page {
  background: #f3f4f6;
  min-height: 100vh;
  padding: 30px;
}

/* Non-admin layout = single column */
.not-admin .grid {
  grid-template-columns: 1fr !important;
}

.container {
  max-width: 100%;
  width: 100%;
  height: 100%;
  margin: 0;
  padding: 0 20px;
}

/* Header */
.header h1 {
  font-size: 34px;
  font-weight: bold;
  color: #222;
  text-align: center;
}

.welcome {
  text-align: center;
  color: #1e40af;
  font-size: 20px;
  margin-top: 8px;
}

.subtitle {
  text-align: center;
  color: gray;
  margin-top: 4px;
}

.grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 100px;
  width: 80%;             /* grid width */
  margin: 0 auto;         /* centers the grid horizontally */
  height: 80%;
  padding: 20px;
  align-items: stretch;    /* makes cards same height */
}

/* Card */
.card {
  background: white;
  padding: 35px;
  width: 100%;       /* ensure card covers full grid cell */
  height: 100%;      /* optional: make it taller */
  border-radius: 18px;
  box-shadow: 0 4px 10px #0002;
}

/* Form */
label {
  display: block;
  margin-bottom: 6px;
  font-weight: 500;
  color: #444;
}

input,
select {
  width: 100%;
  padding: 11px;
  border: 1px solid #ccc;
  border-radius: 8px;
  margin-bottom: 16px;
}
input{
  width: 97%;
}

button {
  width: 100%;
  padding: 13px;
  background: #2563eb;
  border: none;
  border-radius: 8px;
  color: white;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
}

button:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

/* Msg */
.msg {
  padding: 14px;
  border-left: 4px solid;
  border-radius: 8px;
  margin-bottom: 20px;
  font-weight: 500;
}

.msg-success {
  background: #d1fae5;
  border-color: #059669;
  color: #065f46;
}

.msg-error {
  background: #fee2e2;
  border-color: #dc2626;
  color: #7f1d1d;
}

/* User List */
.user-list {
  max-height: 350px;
  overflow-y: auto;
  padding-right: 10px;
}

.user-card {
  background: #f8fafc;
  position: relative;
  padding-bottom: 50px;
  border: 1px solid #e5e7eb;
  padding: 14px;
  border-radius: 10px;
  margin-bottom: 14px;
}

/* User top row */
.user-top {
  display: flex;
  justify-content: space-between;
  margin-bottom: 6px;
}

.user-name {
  font-weight: 600;
  color: #111;
}

.user-status {
  padding: 4px 10px;
  border-radius: 10px;
  font-size: 12px;
}

.active {
  background: #bbf7d0;
  color: #065f46;
}

.inactive {
  background: #fecaca;
  color: #7f1d1d;
}

/* Footer line inside user card */
.user-footer {
  display: flex;
  justify-content: space-between;
  font-size: 12px;
  border-top: 1px solid #e5e7eb;
  padding-top: 6px;
  margin-top: 8px;
  margin-bottom: 15px;
}

.action-row {
  margin-top: 0;
  position: absolute;
  bottom: 10px;
  right: 10px;
  display: flex;
  gap: 8px;
}

.edit-btn,
.delete-btn {
  background: none;
  border: none;
  padding: 8px;
  font-size: 16px;
  line-height: 1;
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  transition: background-color 0.2s, transform 0.1s;
}
.edit-btn {
  background-color: #2563eb;
}
.edit-btn:hover {
  background-color: #1d4ed8;
}

.delete-btn {
  background-color: #dc2626;
}
.delete-btn:hover {
  background-color: #b91c1c;
}

/* Footer */
.footer {
  text-align: center;
  color: gray;
  font-size: 13px;
  margin-top: 40px;
}

```

Add  root/srv/src/main/java/customer/hpsmt/handlers/UserValidation.java

```jsx
package customer.hpsmt.handlers;

import java.util.List;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import com.sap.cds.Result;
import com.sap.cds.Row;

import com.sap.cds.ql.Select;
import com.sap.cds.ql.cqn.CqnSelect;

import com.sap.cds.services.cds.CqnService;
import com.sap.cds.services.handler.EventHandler;
import com.sap.cds.services.handler.annotations.On;
import com.sap.cds.services.handler.annotations.ServiceName;

import com.sap.cds.services.persistence.PersistenceService;

import cds.gen.adminservice.AdminService_;
import cds.gen.adminservice.Users;
import cds.gen.adminservice.Users_;

@Component
@ServiceName(AdminService_.CDS_NAME)
public class UserValidation implements EventHandler {

    @Autowired
    PersistenceService db;
     
    //---------------VALIDATION ON CREATE--------------------------
    @On(event = CqnService.EVENT_CREATE, entity = Users_.CDS_NAME)
    private void validateCreate(Users user){
        if (user==null) return;
        validateUserInput(user);
    }
    //----------------VALIDATION ON UPDATE-------------------------
    @On(event = CqnService.EVENT_UPDATE, entity = Users_.CDS_NAME)
    private void validateUpdate(Users user){
        if (user==null) return;
        validateUserInput(user);
    }
    //---------------VALIDATION OF USER INPU------------------------
    private void validateUserInput(Users userInput) {
        String name = userInput.getName();
        String email = userInput.getEmail();
        String role = userInput.getRole();
        String id = userInput.getId();
    
        //Name check.
        if (name == null || name.trim().isEmpty()) {
            throw new IllegalArgumentException("Name is required.");
        }
        if (name.length() < 3) {
            throw new IllegalArgumentException("Name must be at least 3 characters.");
        }
        if (!name.matches("^[A-Za-z ]+$")) {
            throw new IllegalArgumentException("Name can contain only alphabets and spaces.");
        }
    
        // Email check
        if (email == null || email.trim().isEmpty()) {
            throw new IllegalArgumentException("Email is required.");
        }
    
        email.trim().toLowerCase();//setting the email to lowercase.
        userInput.setEmail(email);
    
        // Check duplicates (new OR update)
        if (emailExists(email, id)) {
            throw new IllegalArgumentException("Email already exists.");
        }
    
        //Role check
        if (role == null || role.trim().isEmpty()) {
            throw new IllegalArgumentException("Role must be selected.");
        }
    }
    

    //------------------CHECK IF EMAIL ALREAY EXISTS (EXCEPT SELF)----------------------
    private boolean emailExists(String email, String excludeId) {

        CqnSelect select = Select.from(Users_.class)
                .where(u -> u.email().eq(email));
    
        Result result = db.run(select);
        List<Row> rows = result.list();
    
        if (rows.isEmpty()) {
            return false;   // email does not exist
        }
    
        String foundId = rows.get(0).get("ID").toString();
    
        // If updating and it's your own record → OK
        if (excludeId != null && excludeId.equals(foundId)) {
            return false;
        }
    
        // Otherwise it's someone else's email → duplicate
        return true;
    }
    
    
    
}

```

As we made changes in the package.json  we don’t need to move build from ui/frontend to resources manually and we don’t have to run mbt build and cf deploy seperately we can simply run:
```bash
npm run stage
npm run deploy
```
