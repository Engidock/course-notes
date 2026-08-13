# 🐳 Docker Project Mastery

### The VendorHub Project — What We're Building

**VendorHub** is a multi-vendor e-commerce platform. Sellers list products. Buyers browse and purchase. Real-time inventory, payments, order tracking, seller dashboard.

**The Architecture:** Node.js API backend, React frontend, MySQL database, Redis cache, Nginx reverse proxy — all containerized. Single developer can run the entire stack locally with `docker compose up`. Engineers deploy to production servers in minutes.

**Why Docker?** Without containers: "Install Node 18, MySQL 8, Redis 7, npm packages, set environment variables..." on 10 servers. Someone misconfigures one server. Bugs appear only in production. Database schema drifts. With Docker: build once, deploy everywhere. Same container on your laptop, staging, production.

**📍 Scene: VendorHub's First Deployment**

> **Karthik (Backend Lead)**
> 
> "We're ready to deploy. I've built the Node.js API. Srishti has the React frontend. Neeraj set up MySQL. But we have 5 production servers in different data centers, 10 developers on different laptops (some Mac, some Linux), and a staging environment. How do we ensure they all run exactly the same code?"

> **Priya (DevOps Lead)**
> 
> "Docker. We containerize everything. The Node.js API gets a Dockerfile. The React frontend too. MySQL, Redis — we pull official images. We write a docker-compose.yml that defines the entire stack. One file, one command: `docker compose up`, and all 5 services run identically everywhere."

> **Karthik**
> 
> "But what if the database needs to be running before the API starts? What about volumes for data persistence? What if we need to scale to 100 servers?"

> **Priya**
> 
> "That's where we start. Docker Compose for local dev and staging. When we hit scale limits, we move to Kubernetes. But first, let's master Docker."

### 1. VendorHub Project Structure — Everything Containerized

#### Directory Layout

```
vendorhub/
├── api/ # Node.js backend
│   ├── Dockerfile
│   ├── package.json
│   ├── server.js
│   ├── routes/
│   ├── models/
│   └── .dockerignore
├── frontend/ # React app
│   ├── Dockerfile
│   ├── package.json
│   ├── public/
│   ├── src/
│   └── .dockerignore
├── nginx/ # Reverse proxy
│   ├── Dockerfile
│   └── nginx.conf
├── docker-compose.yml # Entire stack definition
├── .env # Secrets (git-ignored)
└── README.md
```

> **api/** — Node.js service. Has its own Dockerfile. Defines dependencies, entry point, exposed port.
**frontend/** — React service. Builds a static bundle, served by Nginx or embedded in Node.js.
**nginx/** — reverse proxy. Routes /api/* to Node.js, everything else to React. Handles SSL (later), compression, caching.
**docker-compose.yml** — defines all services, networks, volumes, environment variables. One source of truth.
**.env** — secrets (database passwords, API keys). Never committed to Git. Environment-specific.

```
     VendorHub Architecture (Containerized)

        ┌────────────────────────────────────────────────────┐
        │                  Developer Laptop                  │
        │  $ docker compose up                               │
        │                                                    │
        │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐│
        │  │   Nginx      │  │   Node.js    │  │  React   ││
        │  │   :80, :443  │→→│   API        │→→│  Frontend││
        │  │   Container  │  │   :3000      │  │ :3000    ││
        │  └──────────────┘  └──────────────┘  └──────────┘│
        │                            ↓                       │
        │                    ┌──────────────┐                │
        │                    │   MySQL 8    │                │
        │                    │   :3306      │                │
        │                    │   Container  │                │
        │                    └──────────────┘                │
        │                            ↓                       │
        │                    ┌──────────────┐                │
        │                    │   Redis      │                │
        │                    │   :6379      │                │
        │                    │   Container  │                │
        │                    └──────────────┘                │
        │                                                    │
        │  All containers on same docker network,            │
        │  resolve each other by service name.               │
        └────────────────────────────────────────────────────┘
        
```

### 2. Building the API Container — Node.js Dockerfile

#### api/Dockerfile — Production-Ready

```dockerfile
FROM node:18-alpine
RUN addgroup -g 1001 appuser && \
    adduser -D -u 1001 -G appuser appuser
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --chown=appuser:appuser . .
USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (r) => {if (r.statusCode !== 200) throw new Error(r.statusCode)})"
CMD ["node", "server.js"]
```

> **FROM node:18-alpine** — minimal base. Alpine = 5 MB. Full Ubuntu image = 100+ MB. Fast to build, fast to deploy.
**adduser appuser** — creates non-root user. Running as root is a security vulnerability. If container is compromised, attacker gets root privileges.
**npm ci --only=production** — "clean install" respects package-lock.json exactly. Production flag skips devDependencies (testing libraries, linters). Smaller image.
**--chown=appuser:appuser** — files are owned by appuser, not root. Prevents permission issues later.
**HEALTHCHECK** — Docker monitors /health endpoint. If 3 consecutive checks fail (90 seconds), container is marked unhealthy. Orchestrators can restart it.

#### api/.dockerignore — Keep Build Context Small

```bash
node_modules/
.git/
.env
.env.local
npm-debug.log
coverage/
tests/
.DS_Store
.vscode/
*.md
```

> **node_modules/** — don't copy your local node_modules. RUN npm ci installs fresh, matching exact versions in lock file.
**.git/** — exclude version control. Saves 50+ MB. Git info is not needed in the running container.
**.env** — never include secrets in the image. Passed at runtime via environment variables or Docker secrets.
Smaller build context = faster `docker build`. Docker sends the context to the daemon; huge contexts are slow.

#### Building & Running the API Container

```
$ cd api/
$ docker build -t vendorhub-api:1.0 .

$ docker run -d --name api \
  -p 3000:3000 \
  -e DB_HOST=localhost \
  -e DB_USER=root \
  -e DB_PASS=rootpass \
  -e REDIS_URL=redis://localhost:6379 \
  vendorhub-api:1.0
```

> **docker build -t vendorhub-api:1.0 .** — builds the image using api/Dockerfile. Tags it as "vendorhub-api" version "1.0".
**-d (detach)** — runs in background. You get your terminal back. Use `docker logs api` to see output.
**-p 3000:3000** — maps host port 3000 to container port 3000. Access API at http://localhost:3000.
**-e DB_HOST=localhost** — sets environment variable inside container. Code reads process.env.DB_HOST.
Note: Using localhost won't work if database is in another container. We'll fix this with docker-compose and networking.

### 3. Building the Frontend Container — Multi-Stage React Build

#### frontend/Dockerfile — Multi-Stage for Optimization

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
COPY --from=builder /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> **FROM node:18-alpine AS builder** — first stage: build the React app. node:18-alpine is ~150 MB.
**npm run build** — runs webpack, creates optimized static files in ./build. This process needs Node/npm.
**FROM nginx:alpine** — second stage: production image. nginx:alpine is ~40 MB. Much smaller than node.
**--from=builder /app/build** — copies only the static output from stage 1. No Node.js, no source code, no npm.
**Final image size:** ~50 MB (nginx + React build). Without multi-stage: ~300 MB (Node + build artifacts + source).

### 4. Docker Compose — The Complete Stack Definition

#### docker-compose.yml — VendorHub Full Application

```
version: '3.9'
services:
  mysql:
    image: mysql:8.0
container_name: vendorhub-db
environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
MYSQL_DATABASE: vendorhub
MYSQL_USER: ${DB_USER}
MYSQL_PASSWORD: ${DB_PASSWORD}
volumes:
      - db-data:/var/lib/mysql
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
ports:
      - "3306:3306"
networks:
      - vendorhub-net
healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
interval: 10s
timeout: 5s
retries: 5
redis:
    image: redis:7-alpine
container_name: vendorhub-cache
volumes:
      - cache-data:/data
ports:
      - "6379:6379"
networks:
      - vendorhub-net
api:
    build:
      context: ./api
dockerfile: Dockerfile
container_name: vendorhub-api
ports:
      - "3000:3000"
environment:
      NODE_ENV: production
DB_HOST: mysql
DB_PORT: 3306
DB_NAME: vendorhub
DB_USER: ${DB_USER}
DB_PASSWORD: ${DB_PASSWORD}
REDIS_URL: redis://redis:6379
JWT_SECRET: ${JWT_SECRET}
depends_on:
      mysql:
        condition: service_healthy
redis:
        condition: service_started
networks:
      - vendorhub-net
restart: unless-stopped
frontend:
    build:
      context: ./frontend
dockerfile: Dockerfile
container_name: vendorhub-frontend
ports:
      - "80:80"
depends_on:
      - api
networks:
      - vendorhub-net
restart: unless-stopped
volumes:
  db-data:
  cache-data:

networks:
  vendorhub-net:
    driver: bridge
```

> **version: '3.9'** — Compose file format. Compatible with Docker 20+.
**${DB_ROOT_PASSWORD}** — reads from .env file. Never hardcode secrets.
**mysql.volumes: db-data:/var/lib/mysql** — named volume persists database data. Survives container restart.
**./init.sql:/docker-entrypoint-initdb.d/** — mounts SQL file. MySQL runs it on first startup. Sets up schema.
**healthcheck** — MySQL runs mysqladmin ping. If fails 5 times, container marked unhealthy. depends_on waits for this.
**api.depends_on.mysql.condition: service_healthy** — API starts only after MySQL is actually ready (not just started). Critical.
**DB_HOST: mysql** — API connects to service name "mysql", which Docker DNS resolves to MySQL container's IP.
**REDIS_URL: redis://redis:6379** — Redis service name. API connects via this URL.
**restart: unless-stopped** — if container crashes, Docker restarts it automatically. unless-stopped = don't restart if explicitly stopped.
**vendorhub-net** — user-defined bridge. All services on this network can resolve each other by name.

### 5. Running the Full VendorHub Stack

#### Starting Everything — One Command

```
$ cat > .env << EOF
DB_ROOT_PASSWORD=rootsecret123
DB_USER=vendoruser
DB_PASSWORD=userpass456
JWT_SECRET=your-jwt-secret-key-here
EOF

$ docker compose up -d

$ docker compose logs -f api
```

> **.env file** — stores secrets. Never commit to Git. Add .env to .gitignore.
**docker compose up -d** — starts all services in background. Builds api and frontend images (first time), creates network, starts containers in dependency order.
**-f (follow)** — streams logs in real-time. Watch startup, see any errors.

#### Checking Stack Status

```
$ docker compose ps
NAME                COMMAND              STATUS
vendorhub-db        "docker-entrypoint"  Up 15s (healthy)
vendorhub-cache     "redis-server"       Up 12s
vendorhub-api       "node server.js"     Up 8s (healthy)
vendorhub-frontend  "nginx -g daemon"    Up 5s
$ curl http://localhost:3000/health
{"status":"ok","database":"connected","cache":"connected"}
$ curl http://localhost/
200 OK — React frontend served
```

> **docker compose ps** — shows all services defined in docker-compose.yml. NAME, COMMAND, STATUS.
**(healthy)** — healthcheck is passing. Container is deemed healthy by Docker.
**curl http://localhost:3000/health** — API health endpoint. Returns JSON. Frontend can't reach it directly (blocked by nginx).
**curl http://localhost/** — hits nginx on port 80. Nginx serves React frontend.

### 6. How Containers Talk — Networking & Service Discovery

#### Inspecting the Network

```
$ docker network ls
NETWORK ID     NAME                    DRIVER
a1b2c3d4e5f6   vendorhub_vendorhub-net bridge
$ docker network inspect vendorhub_vendorhub-net

{
  "Name": "vendorhub_vendorhub-net",
  "Containers": {
    "abc123": {
      "Name": "vendorhub-api",
      "IPv4Address": "172.24.0.3/16"
    },
    "def456": {
      "Name": "vendorhub-db",
      "IPv4Address": "172.24.0.2/16"
    }
  },
  "Scope": "local"
}
```

> **vendorhub_vendorhub-net** — named bridge network created by docker compose. All services attached.
**Containers object** — lists all attached containers with their IPs.
**Embedded DNS** — Docker's DNS server resolves service names to IPs. When API code does `mysql.connect()`, DNS resolves "mysql" to 172.24.0.2.

#### Inside the API Container — How Connection Strings Work

```
$ docker compose exec api bash

app# echo $DB_HOST
mysql
app# ping mysql
PING mysql (172.24.0.2): 56 data bytes
64 bytes from 172.24.0.2: icmp_seq=0 ttl=64 time=0.432 ms
app# mysql -h $DB_HOST -u $DB_USER -p$DB_PASSWORD -e "SELECT VERSION();"
+---------------------------+
| VERSION()                 |
+---------------------------+
| 8.0.35-0ubuntu0.22.04.1   |
+---------------------------+
```

> **docker compose exec api bash** — opens a shell inside the running api container.
**echo $DB_HOST** — confirms environment variable is "mysql" (service name, not IP).
**ping mysql** — DNS resolves "mysql" to 172.24.0.2. Network connectivity works.
**mysql -h $DB_HOST** — successfully connects to MySQL using service name. No hardcoded IPs needed.

### 7. Data Persistence — Volumes and Backups

#### Understanding Named Volumes

```
$ docker volume ls
DRIVER    VOLUME NAME
local     vendorhub_db-data
local     vendorhub_cache-data
$ docker volume inspect vendorhub_db-data
[
  {
    "Name": "vendorhub_db-data",
    "Mountpoint": "/var/lib/docker/volumes/vendorhub_db-data/_data",
    "Driver": "local"
  }
]
$ docker compose down           # Stop all services
$ docker volume ls              # Volumes still exist!
$ docker compose up -d          # Restart stack
$ mysql -h mysql -u user -p -e "SELECT COUNT(*) FROM products;"
Count: 45623                   (data is still there!)
```

> **docker volume ls** — lists all named volumes on the system.
**Mountpoint: /var/lib/docker/volumes/...** — actual location on host filesystem where data is stored.
**docker compose down** — stops and removes containers. Does NOT remove named volumes (unless -v flag).
**Data persists** — when you restart the stack, new MySQL container mounts the same volume. All data is still there.

#### Backing Up Database Data

```
$ docker compose exec mysql mysqldump \
  -u $DB_USER -p$DB_PASSWORD vendorhub \
  > vendorhub-backup-$(date +%Y%m%d-%H%M%S).sql

$ ls -lh vendorhub-backup-*.sql
-rw-r--r-- 1 user user 2.3M vendorhub-backup-20240115-143025.sql
# Restore from backup
$ docker compose exec -T mysql mysql \
  -u $DB_USER -p$DB_PASSWORD vendorhub \
  < vendorhub-backup-20240115-143025.sql
```

> **docker compose exec mysql mysqldump** — runs mysqldump inside the MySQL container. Outputs SQL dump to your host.
**$(date +%Y%m%d-%H%M%S)** — includes timestamp in filename. Prevents overwriting old backups.
**docker compose exec -T mysql** — the -T flag disables pseudo-TTY. Needed when piping stdin (< filename).
Backup is a standard SQL file. Can restore to any MySQL 8.0 database, Docker or not.

```
     Data Persistence with Volumes

        Host Machine:
        /var/lib/docker/volumes/vendorhub_db-data/_data/
        ├── mysql/
        │   ├── ibdata1
        │   ├── mysql.ibd
        │   └── vendorhub/
        │       ├── products.ibd
        │       ├── orders.ibd
        │       └── users.ibd
        
        Inside Container:
        /var/lib/mysql/                    (mount point)
        ├── ibdata1
        ├── mysql.ibd
        └── vendorhub/
        
        Same data, two views:
        - Container reads/writes to /var/lib/mysql/
        - Host sees actual files at /var/lib/docker/volumes/...
        
        Container stops → Data stays on host filesystem.
        New container starts → Mounts same volume → Sees same data.
        
```

### 8. Deploying to Production — Building & Pushing Images

#### Building Production Images

```
$ docker build -t vendorhub-api:1.2.3 ./api
$ docker build -t vendorhub-frontend:1.2.3 ./frontend

$ docker images | grep vendorhub
vendorhub-api        1.2.3    a1b2c3d4e5f6   2 minutes ago   185 MB
vendorhub-frontend   1.2.3    f6e5d4c3b2a1   1 minute ago    52 MB
```

> **Tag format: repo:version** — use semantic versioning (1.2.3), git commit SHA, or both.
**API image: 185 MB** — Node.js runtime + dependencies. Alpine base helps.
**Frontend image: 52 MB** — nginx + React static files. Multi-stage build removes build-time dependencies.

#### Pushing to Docker Registry (AWS ECR)

```bash
$ aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com
$ docker tag vendorhub-api:1.2.3 \
  123456789.dkr.ecr.us-east-1.amazonaws.com/vendorhub-api:1.2.3
$ docker push \
  123456789.dkr.ecr.us-east-1.amazonaws.com/vendorhub-api:1.2.3
```

> **aws ecr get-login-password** — generates temporary auth token (valid 12 hours). More secure than static password.
**docker login** — authenticates with private ECR registry.
**docker tag** — creates alias pointing to the image. Now it has an ECR URI.
**docker push** — uploads image layers to ECR. Only new/changed layers are sent (efficient).

#### Deploying on Production Server

```bash
# On production server (e.g., EC2 instance)
$ aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com
$ docker pull \
  123456789.dkr.ecr.us-east-1.amazonaws.com/vendorhub-api:1.2.3
$ docker stop vendorhub-api
$ docker run -d --name vendorhub-api \
  -p 3000:3000 \
  -e DB_HOST=db.internal \
  -e DB_USER=$DB_USER \
  -e DB_PASSWORD=$DB_PASSWORD \
  -v api-logs:/app/logs \
  123456789.dkr.ecr.us-east-1.amazonaws.com/vendorhub-api:1.2.3
```

> **aws ecr get-login-password** — EC2 instance (with IAM role) authenticates with ECR. No hardcoded credentials.
**docker pull** — downloads the image from ECR. Only changed layers are downloaded (incremental).
**docker stop vendorhub-api** — gracefully stops old container (10-second timeout).
**docker run** — starts new container with new image version. Old container data (volumes) persists if mounted.
**-v api-logs:/app/logs** — mounts volume for logs. Survives container restart. Logs don't disappear.

### 9. Production Troubleshooting — Monitoring & Debugging

#### Checking Container Health

```
$ docker ps -a
STATUS: Up 2 hours (healthy)
$ docker inspect vendorhub-api | grep -A 10 '"Health"'
"Health": {
  "Status": "healthy",
  "FailingStreak": 0,
  "Log": [
    {
      "Start": "2024-01-15T14:30:45.123456Z",
      "Exit": 0,
      "Output": "..."
    }
  ]
}
$ docker stats vendorhub-api
CONTAINER ID   MEM USAGE / LIMIT   CPU %    NET I/O
abc12345       145M / 512M        0.8%     1.2MB / 890KB
```

> **docker ps -a** — STATUS shows (healthy), (unhealthy), or (starting). Healthy = healthcheck passing.
**docker inspect | grep Health** — detailed healthcheck history. Shows last 5 checks, exit codes, timestamps.
**FailingStreak: 0** — healthcheck passed. If it was 3, container would be marked unhealthy.
**docker stats** — real-time resource usage. If MEM USAGE is near LIMIT, container may be getting OOMKilled.

#### Viewing Logs & Debugging Crashes

```
$ docker logs vendorhub-api --tail 100

$ docker logs vendorhub-api --follow        # Real-time
$ docker logs vendorhub-api --since 10m     # Last 10 minutes
$ docker compose logs api | grep ERROR
```

> **docker logs** — prints stdout/stderr from the container. This is your application's console output.
**--tail 100** — shows last 100 lines. Useful for large logs.
**--follow** — like tail -f. Watch logs in real-time as they're written.
**--since 10m** — shows logs from the last 10 minutes. Filter by time to find recent errors.
**If container is dead:** `docker ps -a` shows it with status Exited(1). Check logs to see the error.

**📍 Scene: VendorHub Goes Live**

> **Priya**
> 
> "Launch day. We've containerized the entire stack. Built images, pushed to ECR. Started all 5 services on the production server with one docker-compose up. MySQL is running, Redis is caching, API is responding, frontend is being served."

> **Karthik**
> 
> "Within the first hour: 2,000 concurrent users. The healthcheck monitors everything. API stays up. Database stays up. No crashes. Logs are clean. This is night and day from our previous deployments where we'd manually SSH into servers, check logs, restart services."

> **Neeraj (QA)**
> 
> "And a bug was found in production. Instead of a 4-hour hotfix cycle, Karthik pushed a fix to Git, the CI pipeline built a new image (1.2.4), pushed it to ECR, and we deployed the new container. 15 minutes total. The old container logs are saved. The new one runs the fixed code."

> **Docker Project — Core Takeaways for Freshers**

> - **Docker is the deployment unit.** Not a VM. Not a server. A lightweight, immutable container. Same container from dev to production.
> - **Dockerfile is infrastructure-as-code.** Every dependency, version, permission is version-controlled. No "it works on my machine" mysteries.
> - **Multi-stage builds shrink images.** Build in a heavy image (compiler, build tools), copy only the artifact to a slim final image. 300 MB → 50 MB.
> - **docker-compose orchestrates locally.** Define API, database, cache, frontend in YAML. One command starts the entire stack. Perfect for development and testing.
> - **Named volumes persist data.** Containers are ephemeral. Data stored in volumes survives container restart. Database must use volumes.
> - **Service discovery by name.** API code doesn't hardcode database IP. It uses service name "mysql". Docker DNS resolves it automatically.
> - **Healthchecks enable auto-recovery.** If the app crashes, healthcheck detects it. Orchestrators (Docker Swarm, Kubernetes) restart the container automatically.
> - **Registries are your artifact repository.** Build once, push to ECR, pull on any server. No "recompile on each server" nonsense.

##### Docker Standards — VendorHub Production Rules

- Use semantic versioning for image tags. Never use "latest" in production. Tag with 1.2.3, or commit SHA like abc1234de. Ambiguity causes deployment disasters.
- Always run containers as non-root. If container is compromised, attacker gets unprivileged user, not root. Add a RUN line creating an appuser.
- Include HEALTHCHECK in every Dockerfile. Orchestrators use it to restart failed containers. Without it, a dead app looks alive to the orchestrator.
- Use named volumes for persistent data. Anonymous volumes are cleaned up with `docker rm -v`. Named volumes survive. Use for databases, caches, logs.
- Use depends_on with condition: service_healthy. Default behavior (just service_started) starts containers in order but doesn't wait for readiness. MySQL might still be starting when API tries to connect.
- Never hardcode secrets in Dockerfiles or images. Use .env files for development, AWS Secrets Manager or Kubernetes Secrets for production.
- Keep images under 200 MB if possible. Use Alpine base images, multi-stage builds, .dockerignore to exclude build artifacts. Large images = slow deploys.
- Logs are your window into production. Ensure the app writes to stdout/stderr, not files. `docker logs` captures them. Use ELK stack or CloudWatch for aggregation.

##### 🏋️ Hands-On Exercises — Build VendorHub End-to-End

1. **Set up the project structure:** Create directories vendorhub/{api,frontend,nginx}. Inside api/, create a simple Express.js app in server.js that listens on port 3000 and has /health endpoint. Create package.json with express. Create a Dockerfile using node:18-alpine, USER appuser, HEALTHCHECK. Run `docker build -t my-api:1.0 api/`. Run it locally: `docker run -p 3000:3000 my-api:1.0`. Test with curl http://localhost:3000/health.
2. **Add a database with docker-compose:** Create docker-compose.yml with two services: api (your Node.js app) and db (mysql:8.0). Set MYSQL_ROOT_PASSWORD in environment. Add volumes: db-data for /var/lib/mysql. Add healthcheck to MySQL. Run `docker compose up -d`. Wait for MySQL to be healthy (check docker compose ps). Exec into api container, use mysql CLI to connect to db service by hostname. Run SELECT 1; to verify connection.
3. **Build a React frontend:** Create frontend/ with Create React App. Add a simple page that calls http://localhost:3000/api/products. Create Dockerfile using multi-stage: build React, copy to nginx. Run `docker compose up`. Frontend is served on port 80, API on 3000. Nginx routes /api/* to the API service. Test both from browser.
4. **Set up data persistence:** Use docker-compose volumes to mount db-data. Insert sample data: docker compose exec db mysql -u root -ppassword vendorhub < sample.sql. Run docker compose down. Run docker compose up -d. Verify data is still there with docker compose exec db mysql -u root -ppassword vendorhub -e "SELECT COUNT(*) FROM products;". Data persisted!
5. **Deploy to "production":** Build final images: docker build -t vendorhub-api:1.0.0 api/. Tag for a registry (if you have AWS account): docker tag vendorhub-api:1.0.0 123456789.dkr.ecr.us-east-1.amazonaws.com/vendorhub-api:1.0.0. Push: docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/vendorhub-api:1.0.0. On another machine (or simulated with `docker pull` from your personal Docker account), pull the image, run the stack, verify it works identically.

### Docker Project Complete 🎉

You have built VendorHub — a complete multi-container e-commerce platform. You containerized a Node.js API, a React frontend, MySQL database, Redis cache, and Nginx reverse proxy. Wrote Dockerfiles with production best practices (Alpine base, non-root user, healthchecks, multi-stage builds). Orchestrated them with docker-compose for local development. Pushed images to a registry. Deployed to production servers. Monitored and debugged running containers. This is exactly how modern DevOps engineers build and deploy applications.

> **Priya**
> 
> "Before Docker: development environment, staging environment, production environment — all different. Someone would spend days trying to reproduce a bug because of environment mismatch. Now: one docker-compose.yml, runs identically on 50 developer laptops and 100 production servers. That's the power of containerization. You've just learned the foundation of modern DevOps."

> **Karthik**
> 
> "And now that you understand Docker, the next step is Kubernetes. Docker runs a single container or a few containers on one machine. Kubernetes orchestrates thousands of containers across hundreds of machines. Auto-scaling, self-healing, rolling updates. But Kubernetes is built on Docker. Master Docker first, then Kubernetes will make sense."

> **Next: Advanced Container Orchestration & CI/CD**

> - Kubernetes — orchestrate containers at scale. ReplicaSets for auto-scaling, Services for networking, Ingress for routing, PersistentVolumes for storage.
> - AWS ECS — Docker on AWS. Task definitions are like docker-compose files. Scale to hundreds of tasks across an EC2 cluster or use Fargate (serverless).
> - CI/CD Integration — GitHub Actions, GitLab CI, or Jenkins. Push code → automatically build Docker image → run tests → push to registry → deploy to production.
> - Container Security — scan images for vulnerabilities (Trivy), enforce non-root users, use read-only filesystems, network policies (Kubernetes).
> - Logging & Monitoring — Docker logs to file/syslog, aggregate with ELK stack or CloudWatch, monitor metrics with Prometheus and Grafana.
> - Service Mesh — Istio or Linkerd. Manage service-to-service communication, load balancing, retries, circuit breaking, observability.
