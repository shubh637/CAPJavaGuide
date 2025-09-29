# 📘 CAP Java Backend with SAP BTP Deployment

## Table of Contents
1. [Introduction](#introduction)
2. [Setting Up Development Environment](#setting-up-development-environment)
3. [Project Structure](#project-structure)
4. [Data Model](#data-model)
5. [Service Definition](#service-definition)
6. [Running Locally](#running-locally)
7. [Testing with Postman](#testing-with-postman)
8. [MTA Configuration](#mta-configuration)
9. [Building MTA Archive](#building-mta-archive)
10. [Deploying to BTP](#deploying-to-btp)
11. [Accessing the Service](#accessing-the-service)
12. [Conclusion](#conclusion)

---

## Introduction

This project demonstrates how to build and deploy a **CAP (Cloud Application Programming Model) Java application** to **SAP BTP (Cloud Foundry runtime)**.  

It includes:
- 📚 A simple data model (`Books`)  
- 🛠 An OData V4 service (`CatalogService`)  
- 🚀 Deployment to BTP using `mta.yaml` and `mbt`  

---

## Setting Up Development Environment

### Install Tools
```bash
# Install MBT (Multi-Target Application Builder)
npm install -g mbt

# Install CDS Development Kit
npm install -g @sap/cds-dk

# Login to SAP BTP
cf login
