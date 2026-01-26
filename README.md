# n8n-practical-guide
Step-by-step guide to understanding n8n, self-hosting it locally or on Oracle Cloud VM, and building practical automation workflows


## table of contents
- [Introduction](#introduction)
- [How n8n works](#how-n8n-works)
- [Running n8n in a Virtual Machine (Oracle Cloud)](#running-n8n-in-a-virtual-machine-oracle-cloud)
- [Running n8n Locally](#running-n8n-locally)
- [Projects ](Projects/projects.md)


## Introduction
**n8n** (short for “node-to-node”) is an open-source workflow automation tool that allows you to connect different systems, services, and APIs using a visual interface.

At its core, n8n helps you automate tasks such as:
- Triggering actions when an event happens
- Moving data between different systems
- Integrating APIs without writing full applications
- Building reliable background workflows

Instead of writing long scripts or glue code, you build workflows composed of nodes, where each node performs a single task.

## How n8n works
n8n is a workflow automation engine that connects different systems using visual workflows.

A workflow is made of nodes connected together, where each node performs a single task such as receiving data, calling an API, transforming information, or sending a notification.

Every workflow starts with a trigger (for example a webhook, schedule, or manual trigger). When the trigger fires, n8n creates an execution, runs each node in order, and passes data between nodes as structured JSON.

Nodes receive input data, process it, and produce output data that is used by the next node. This data flow allows workflows to branch, loop, and handle multiple items in a single execution.

n8n stores workflows, credentials, and execution logs locally, allowing full visibility, debugging, and replay of failed runs. Credentials are encrypted and stored separately from workflow logic.

Overall, n8n acts as an automation and orchestration layer, helping systems communicate with each other without building custom glue code.

## Running n8n in a Virtual Machine (Oracle Cloud)

In this example, we will run **n8n inside an Oracle Cloud VM**.

Before starting, make sure your VM is correctly set up.  
You can refer to my other repository: **oracle-cloud-vm-setup**.

---

### Prerequisites

When creating the VM, ensure the following:

- The VM has a **public IP address**
- The VM is inside a **VCN with an Internet Gateway**
- **Ingress traffic is allowed on port 5678** (n8n default port)

If all of the above are satisfied, we can proceed.

---

## 1. Connect to the VM

SSH into your VM from your local machine:

```bash
ssh opc@<VM_PUBLIC_IP>
```
Update the system
```bash
sudo dnf update -y
```

## 2. Install Docker
```bash
sudo dnf install docker -y
sudo systemctl enable --now docker
```
Verify Installation
```bash
docker --version
```
> On Oracle Linux, Docker commands may be executed using Podman internally.
This is expected and does not affect functionality.

## 3. n8n Docker image 
n8n provides an official Docker image:
```bash
docker.io/n8nio/n8n
```

## 4. Test run (temporary container)

Before configuring persistence, we first run a simple test container to verify that everything works.

```bash
docker run -it --name n8n_test -p 5678:5678 docker.io/n8nio/n8n
```
If n8n starts correctly, you should see logs ending with something similar to:

```bash
Editor is now accessible via:
http://localhost:5678
```

### Acess n8n from local browser
Open your browser and navigate to:

```bash
http://<VM_PUBLIC_IP>:5678
```

If networking and firewall rules are correct, the n8n UI should load.

You may see a cookie / HTTPS warning at this stage, this is expected for HTTP access and will be fixed later.

> if you arrived at this point, congratulation, it means your VM


## 5. Run n8n with persistent data

Now we start n8n properly, with data persistence enabled.

Create the data directory (required on Oracle Linux / Podman):

```bash
mkdir -p $HOME/.n8n
```
Run n8n in detached mode:

```bash
docker run -d \
  --name n8n_container \
  -p 5678:5678 \
  -v $HOME/.n8n:/home/node/.n8n:Z \
  -e N8N_SECURE_COOKIE=false \
  docker.io/n8nio/n8n:latest
```
What this does

```bash
-d → runs the container in the background

-p 5678:5678 → exposes the n8n web UI

-v ~/.n8n:/home/node/.n8n → persists workflows and credentials

N8N_SECURE_COOKIE=false → allows HTTP access (temporary)
```

## 6. Access the n8n UI
Open your browser:
```bash
http://<VM_PUBLIC_IP>:5678
```

You should now be able to:

- Create an n8n user account
- Log ins
- Create workflows
- Restart the container without losing data