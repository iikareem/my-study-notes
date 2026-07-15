---
tags:
  - containers
  - devops
  - docker
---

### 🟩 Difference Between a **Dockerfile** and a **Docker Image**:

| Aspect             | **Dockerfile**                                                                        | **Docker Image**                                                                               |
| ------------------ | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| 🔧 **What is it?** | A **text file** with instructions on how to build a Docker image.                     | A **snapshot or blueprint** of a container, created by running `docker build` on a Dockerfile. |
| 🏗️ **Purpose**    | To **define how to build** a containerized environment (like OS, packages, app code). | To **store the built environment**, used to run containers.                                    |
| 📄 **File Type**   | Plain text file (usually named `Dockerfile`)                                          | Binary artifact (stored in Docker engine or registries)                                        |
| 🔁 **Usage**       | You write/edit it to define or change the image.                                      | You run containers from it (`docker run image-name`).                                          |
| 🧱 **Example**     | Instructions like `FROM node`, `RUN npm install`                                      | An image like `node:22`, `my-app:latest`                                                       |
| 🔄 **Modifiable?** | Yes — edit it any time                                                                | No — immutable. You must **rebuild** the image to change it.                                   |

---

### 🧠 Analogy:
- **Dockerfile** is like a **recipe** 📄 for baking a cake.
- **Docker Image** is the **baked cake** 🎂 — a finished product made by following the recipe.
- **Container** is when you **serve the cake** 🍽️ — actually running the image.

---

### 🔧 How it works:
```bash
# Step 1: You write a Dockerfile
FROM node:22
WORKDIR /app
COPY . .
RUN npm install
CMD ["npm", "start"]

# Step 2: You build an image from it
docker build -t my-app .

# Step 3: You run a container from the image
docker run my-app
```

Let me know if you want to see the difference visually or with a real-world example!

Ports
Great question — understanding **ports** is _crucial_ in Docker and networking, especially as a backend engineer. Let me explain clearly with **simple examples** and **why you see ports in Docker and `docker-compose.yml`**.

---

## 🚪 What Are Ports?

Think of a **port** as a **door** through which your app can talk to the outside world.

- A server (your machine or container) has many ports: `0` to `65535`
    
- A **service (e.g., RabbitMQ, PostgreSQL, Node.js)** listens on a specific port
    
- You send or receive data through that port — just like walking through a specific door
    

---

## 💡 Example Analogy:

- Your laptop is a **building**
    
- Inside it, you have rooms (apps)
    
- Each room has a **door (port number)**
    
- To talk to **RabbitMQ**, go to **room 5672**
    
- To visit the **RabbitMQ Web UI**, use **room 15672**
    

---

## 🐳 In Docker Compose — `ports:` Means:

```yaml
ports:
  - "5672:5672"
```

This means:

> **Host machine port 5672** will forward to the **container's port 5672**

So when your app on your laptop tries to connect to `localhost:5672`, Docker forwards the request into the container, directly to RabbitMQ.

---

## 📦 Example: RabbitMQ Service in `docker-compose.yml`

```yaml
services:
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"     # Messaging port (used by your Node.js services)
      - "15672:15672"   # Web UI port (used by your browser to access RabbitMQ dashboard)
```

|Port|Purpose|Exposed To|
|---|---|---|
|`5672`|AMQP protocol for services to send/receive messages|Node.js services|
|`15672`|RabbitMQ web UI for debugging|You (via browser)|

---

## 🧠 In Simple Terms:

- The **left port** (`5672`) = port **on your computer**
    
- The **right port** (`5672`) = port **inside the container**
    

So:

```yaml
- "5672:5672"
```

means:

> When I visit `localhost:5672`, Docker will forward traffic to **RabbitMQ inside the container**.

---

## 👨‍🔧 What Happens If You Skip `ports:`?

If you **don’t expose ports**, you:

- **Can’t access RabbitMQ from your host machine**
    
- Can only access it _internally_ (from other containers)
    

So `ports:` is how you _open doors_ from **your machine** → **Docker containers**

---

## 🧪 You Can Try:

Visit:

- `http://localhost:15672` → RabbitMQ Web UI
    
- Login with: `guest / guest`
    

If that works, ports are forwarded correctly.

---

Docker and Docker Compose — concise study notes.

---

## 🐳 Docker & Docker Compose — Microservices Setup Notes

### 🔹 1. Why Use Docker for Microservices?

- Each service runs in **its own isolated environment** (container).
    
- Simplifies deployment and local development.
    
- Ensures consistency across all machines (works the same everywhere).
    

---

### 🔹 2. Dockerizing a Microservice

- Each microservice has:
    
    - A **Dockerfile** that defines how to build it.
        
    - A `CMD` or `ENTRYPOINT` to run the app (e.g. Node.js server).
        
    - Port exposed (typically 3000 inside the container).
        

---

### 🔹 3. Docker Compose

Use `docker-compose.yml` to:

- Define multiple services (Node apps, DBs, RabbitMQ, etc.)
    
- Set **environment variables**, ports, volumes, networks
    
- Start everything together with one command:
    
    ```bash
    docker-compose up --build
    ```
    

---

### 🔹 4. Ports and Communication

|Concept|Description|
|---|---|
|`EXPOSE`|Declares the internal port the app listens to (e.g. 3000). It’s **not published** outside unless you map it.|
|`ports:`|Maps internal container ports to your host machine. Example: `3001:3000` → host:3001 to container:3000|
|Docker DNS|Services in Docker Compose can talk using **service names** as hostnames (e.g. `orders`, `orders_db`).|

---

### 🔹 5. Making Services Talk (Same Port, No Problem)

- All services can listen on **port 3000 inside the container**.
    
- They’re isolated; external communication happens via:
    
    ```yaml
    ports:
      - "3001:3000" # orders
      - "3002:3000" # payments
    ```
    
- Inside the Docker network, you don’t need ports. Just use:
    
    ```js
    fetch("http://payments:3000/api/...")
    ```
    

---

### 🔹 6. Databases per Microservice

- Each service should have **its own database** to ensure loose coupling.
    
- Use Docker Compose to spin up one DB per service:
    
    ```yaml
    services:
      orders_db:
        image: postgres
        environment:
          POSTGRES_USER: user
          POSTGRES_PASSWORD: pass
          POSTGRES_DB: orders_db
    ```
    

---

### 🔹 7. RabbitMQ (Message Broker)

- Add RabbitMQ container:
    
    ```yaml
    rabbitmq:
      image: rabbitmq:3-management
      ports:
        - "5672:5672"     # Internal service comms
        - "15672:15672"   # Web UI for admin
    ```
    
- Use it for async messaging between services (event-based choreography).
    

---

### 🔹 8. `depends_on`

Tells Docker to start a dependency (e.g. DB) **before** the service:

```yaml
depends_on:
  - orders_db
```

⚠️ `depends_on` doesn’t wait for the DB to be _ready_, just that it's started. Use retries in your service if needed.

---

### 🔹 9. `restart: always`

Ensures containers **restart automatically** on failure or reboot.

|Policy|Behavior|
|---|---|
|`no`|Default. Don’t restart.|
|`always`|Restart on crash or system boot.|
|`on-failure`|Restart only if exit code ≠ 0.|
|`unless-stopped`|Restart unless manually stopped.|

✅ Best for: Databases, RabbitMQ, backend services.

---

### 🔹 10. Example Environment Variable in Compose

```yaml
environment:
  DATABASE_URL: postgres://user:pass@orders_db:5432/orders_db
```

- You access the DB using Docker DNS: `orders_db` is the host.
    
- Same applies to calling other services.
    

---

### ✅ Final Tips

- Each service should be **small and self-contained**.
    
- All services can share the same RabbitMQ.
    
- Always test communication with container names, not `localhost`.
    
- Optional: Expose DB ports (like 5433, 5434) to debug with external tools like pgAdmin.
    

---

Would you like a ready-to-copy **Notion table of contents** or link block for this note too?

## See also

- [[Docker Volumes Mounts and Networking]] · [[VMs vs Containers]]
