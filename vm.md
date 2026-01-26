# VM Installation Guide - Focusboard Application Stack

This guide describes how to install and configure the complete Focusboard application stack (Frontend, Focus Service, Auth Service) on a bare Linux VM without Docker.

## Prerequisites

- Debian based Linux distribution (other distros might work but are untested)
- Root or sudo access

## System Architecture

The application consists of three main services:
- **Frontend**: Next.js application (port 3000)
- **Auth Service**: Node.js/Express with gRPC (ports 4000 HTTP, 50051 gRPC)
- **Focus Service**: Quarkus/Java application (port 8080)
- **PostgreSQL**: Two separate databases (auth_db, focus_db)
- **Nginx**: Reverse proxy for HTTPS/routing

---

## Step 1: Required Software

1. Node.js
2. nvm
3. Java
4. Maven
5. PostgresSQL
6. Nginx

---

## Step 2: Setup DB

Authentication for DB

---

## Step 3: Setup Applications

1. clone git repositories
2. install packages using npm / maven
3. create env files
4. compile code / build using existing targets

---

# Step 4: DB init

1. Run drizzle script
2. flyway auto runs

---

## Step 5: Configure Nginx reverse proxy

Configure ports, routing & headers

---

## Step 6: TLS (optional)

Add certificates for Nginx