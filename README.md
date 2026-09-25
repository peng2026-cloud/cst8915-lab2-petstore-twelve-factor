# CST8915 Lab 2: Twelve-Factor Refactor of the Algonquin Pet Store

| Student Name | Student ID | Course | Semester |
|---|---|---|---|
| Peng Wang | 041107730 | CST8915 Full-stack Cloud-native Development | Fall 2026 |

---

## Demo Video

🎥 [Watch Demo Video](https://youtu.be/VIDEO_ID)

---

## Service Repositories

Each service has its own public repository with two commits: the unchanged Lab 1 service, then the Lab 2 refactor, so the refactor can be read as a single diff.

| Service | Repository | Refactor commit |
|---|---|---|
| Order Service (Node.js) | [peng2026-cloud/order-service](https://github.com/peng2026-cloud/order-service) | [431c53a](https://github.com/peng2026-cloud/order-service/commit/431c53a1a312857df20961d5cd6560a65c219fb1) |
| Product Service (Rust) | [peng2026-cloud/product-service](https://github.com/peng2026-cloud/product-service) | [e8d9687](https://github.com/peng2026-cloud/product-service/commit/e8d96874eb30bcb7f3efdf48baa297a72375256c) |
| Store Front (Vue.js) | [peng2026-cloud/store-front](https://github.com/peng2026-cloud/store-front) | [5addec2](https://github.com/peng2026-cloud/store-front/commit/5addec2bc563622cdc7c526be8fb64d33f48bc7b) |

---

## Deployment

The lab asks for four VMs. My Azure for Students subscription allows only 6 vCPUs and 3 public IP addresses per region, and the smallest VM size it can use has 2 vCPUs, so four VMs do not fit in one region. As the professor allowed for this case, I used two VMs: one for RabbitMQ and one for the three application services. RabbitMQ is still a separate machine that the Order Service reaches through its public IP.

Quota check in West US 2 with the two VMs deployed (Azure CLI). Four VMs would need at least 8 vCPUs and 4 public IPs:

```text
> az vm list-usage --location westus2 --query "[?localName=='Total Regional vCPUs'].{Name:localName, CurrentValue:currentValue, Limit:limit}" -o table
Name                  CurrentValue    Limit
--------------------  --------------  -------
Total Regional vCPUs  4               6

> az network list-usages --location westus2 --query "[?name.value=='PublicIPAddresses'].{Name:name.localizedValue, CurrentValue:currentValue, Limit:limit}" -o table
Name                 CurrentValue    Limit
-------------------  --------------  -------
Public IP Addresses  2               3
```

| VM | Public IP | Runs | Inbound rules (NSG) |
|---|---|---|---|
| `rabbitmq-vm` | 4.154.75.161 | RabbitMQ (backing service) | 22 from my laptop; 5672 only from `app-vm` |
| `app-vm` | 4.154.132.183 | Order Service :3000, Product Service :3030, Store Front :8080 | 22, 3000, 3030 and 8080 from my laptop |

Both VMs are Standard_B2ls_v2 (2 vCPUs, 4 GiB) running Ubuntu Server 24.04 LTS in West US 2, in the resource group `lab2-petstore-rg`.

Configuration used in this deployment (each value lives in the service's `.env`, which Git ignores):

| Service | Variable | Value |
|---|---|---|
| order-service | `RABBITMQ_CONNECTION_STRING` | `amqp://orderapp:****@4.154.75.161:5672/` |
| order-service | `PORT` | `3000` |
| product-service | `PORT` | `3030` |
| store-front | `VUE_APP_ORDER_SERVICE_URL` | `http://4.154.132.183:3000` |
| store-front | `VUE_APP_PRODUCT_SERVICE_URL` | `http://4.154.132.183:3030` |

Verification: `curl -i http://localhost:3030/products` returned HTTP 200 with three products, `POST /orders` returned HTTP 200 with `Order received`, and `sudo rabbitmqctl list_queues name durable messages` showed `order_queue` as durable, with the message count rising by one for each order.

---

## Reflection Questions

### 1. What changes did you make to the order-service and product-service to comply with the Configuration and Backing Services factors?

In the Order Service, `index.js` no longer hard-codes `amqp://localhost` and port 3000. It calls `require('dotenv').config()` and reads `process.env.RABBITMQ_CONNECTION_STRING` and `process.env.PORT`, keeping the old values only as local defaults, and `dotenv` is now declared in `package.json` and locked in `package-lock.json`. In the Product Service, `main.rs` calls `dotenv().ok()` and reads `PORT` with `env::var` (default 3030), and I added `dotenv = "0.15"` to `Cargo.toml` and committed the updated `Cargo.lock`. Each repository ignores `.env` and commits only a `.env.example` with placeholders. For Backing Services, RabbitMQ runs on its own VM, and the Order Service knows it only through the connection string: the broker's public IP and a dedicated `orderapp` account, because the default `guest` user only works from localhost. Pointing the service at a different broker is a one-line change in `.env` plus a restart, with no code change.

### 2. Why is it important to use environment variables instead of hard-coding configurations in your application?

Configuration is what changes between deployments, and code should not. In Lab 1 the Store Front had `localhost` URLs in `OrderForm.vue`, so every new VM IP meant editing the code. Now the same commit runs on my laptop or on any VM, and only `.env` differs. Environment variables also keep secrets out of Git: the RabbitMQ password exists only in the VM's `.env`, which `.gitignore` excludes, so the repositories can stay public. I also learned that the Vue variables work differently from the back ends: Vue CLI copies the `VUE_APP_` values into the JavaScript bundle at build time (the served `app.js` already contains `http://4.154.132.183:3030`), so changing them requires restarting `npm run serve`, and they must never hold secrets because every visitor can read them.

### 3. Why is it important to have separate repositories for each microservice? How does this help maintain independence and scalability of each service?

The three services use different languages, dependency tools and release cycles: Node.js with npm, Rust with Cargo, and Vue. With one repository per service, each has its own history, so a change to the Product Service cannot be shipped together with the Order Service by accident, and each service can be tested, versioned, deployed or rolled back on its own. A VM only needs to clone the service it runs, and a team can own one repository without touching the others. For scalability, a busy service can be scaled out, for example with more Order Service instances behind a load balancer, and can get its own CI/CD pipeline without rebuilding the rest. The cost is coordination: when a shared contract such as the order JSON changes, the repositories have to be updated together.

---

## Challenges and Learnings (Optional)

- **Four VMs did not fit the student subscription.** `az vm list-usage` and `az network list-usages` showed 6 vCPUs and 3 public IPs per region, and the low-cost B-series sizes were available only in West US 2, the smallest with 2 vCPUs. A fourth VM was therefore impossible in one region, so I used the two-VM layout the professor allowed.
- **AllocationFailed.** The first VM, on the AMD size B2ats_v2, failed with "insufficient capacity". Instead of recreating it, I resized the failed VM to an Intel size under Availability + scale → Size, which kept its network interface, NSG and static public IP.
- **VS Code Remote-SSH needs memory.** On a 1 GiB VM, the VS Code server took free memory from about 450 MB to 30 MB (Azure metric *Available Memory Bytes*), and new SSH logins hung even though port 22 was open. Resizing to B2ls_v2 (4 GiB) fixed it.
- **SSH from Windows.** VS Code's *Add New SSH Host* dropped the backslashes from my key path in `~/.ssh/config`, and the key moved out of Downloads kept an extra group in its permissions, which OpenSSH rejects as an unprotected private key. Forward slashes in the config and `icacls <key> /reset` fixed both.
- **Cross-origin requests.** Before each `POST /orders`, the browser sent a CORS preflight (`OPTIONS`, 204), because the page on port 8080 and the API on port 3000 are different origins. The `cors` middleware in the Order Service answers it.

---

## Acknowledgments

- Lab instructions and source code: `ramymohamed10/26F_Lab2_CST8915` and `ramymohamed10/26F_Lab1_CST8915`.
- GenAI declaration: As permitted for labs in this course, I used Claude (Anthropic) to help me understand the code and the Twelve-Factor changes, troubleshoot Azure quota, VM size and SSH problems, and draft and review this README; I typed the refactoring and deployment commands myself, and the testing and demo video are my own work.
