# Volume Driver Plugins in Docker

  - Take me to [Lecture](https://kodekloud.com/topic/volume-driver-plugins-in-docker-4/)

In this section, we will take a look at **Volume Driver Plugins in Docker**

- We discussed about Storage drivers. Storage drivers help to manage storage on images and containers.
- We have already seen that if you want to persist storage, you must create volumes. Volumes are not handled by the storage drivers. Volumes are handled by volume driver plugins. The default volume driver plugin is local.
- The local volume plugin helps to create a volume on Docker host and store its data under the `/var/lib/docker/volumes/` directory.
- There are many other volume driver plugins that allow you to create a volume on third-party solutions like Azure file storage, DigitalOcean Block Storage, Portworx, Google Compute Persistent Disks etc.


![class-9](../../images/class9.PNG)


- When you run a Docker container, you can choose to use a specific volume driver, such as the RexRay EBS to provision a volume from the Amazon EBS. This will create a container and attach a volume from the AWS cloud. When the container exits, your data is safe in the cloud.

```
$ docker run -it --name mysql --volume-driver rexray/ebs --mount src=ebs-vol,target=/var/lib/mysql mysql
```


![class-10](../../images/class10.PNG)

# 🚀 Docker Storage Driver vs Volume Driver (Detailed Notes with Examples)

---

## 🧠 1. Storage Driver

### 📌 Definition

Storage driver manages the **container filesystem (image layers + writable layer)**.

---

### 🔍 What it handles

* Image layers (read-only)
* Container writable layer
* Copy-on-write (CoW)
* File system stacking (union filesystem)

---

### 🧩 How it works (step-by-step)

1. Docker image is made of layers
2. When container starts:

   * These layers are mounted as read-only
   * A writable layer is added on top
3. Any changes (file write/delete/update) go into writable layer

---

### 🧪 Example 1: File creation inside container

```bash
docker run -it ubuntu bash
touch /tmp/file1
```

👉 `/tmp/file1` is stored in:

* Container writable layer (managed by storage driver)

👉 If container is deleted:

```bash
docker rm <container>
```

❌ File is lost

---

### 🧪 Example 2: Image layering

```bash
docker build -t myapp .
```

Dockerfile:

```Dockerfile
FROM ubuntu
RUN apt update
RUN apt install nginx
```

👉 Each instruction creates a layer
👉 Storage driver manages merging:

```text
Layer1: ubuntu
Layer2: apt update
Layer3: nginx install
```

---

### 🛠 Common Storage Drivers

* overlay2 (default & recommended)
* aufs (deprecated)
* devicemapper
* btrfs

---

### ⚠️ Key Limitation

* Not meant for persistent data
* Data deleted with container

---

## 🧠 2. Volume Driver

### 📌 Definition

Volume driver manages **persistent storage outside container lifecycle**.

---

### 🔍 What it handles

* Persistent data
* Shared storage between containers
* External storage integrations

---

### 🧩 How it works (step-by-step)

1. Volume is created
2. Volume is mounted inside container
3. Data written to mount path is stored outside container

---

### 🧪 Example 1: Named volume

```bash
docker volume create mydata
docker run -v mydata:/data nginx
```

👉 Writing file:

```bash
echo "hello" > /data/file.txt
```

👉 Actual storage location:

```text
/var/lib/docker/volumes/mydata/_data
```

👉 Container deleted → data remains ✅

---

### 🧪 Example 2: Bind mount

```bash
docker run -v /home/user/data:/data nginx
```

👉 Container writes to `/data`
👉 Actually stored on host:

```text
/home/user/data
```

---

### 🧪 Example 3: Shared volume between containers

```bash
docker run -v mydata:/data busybox
docker run -v mydata:/data nginx
```

👉 Both containers share same data

---

### 🧪 Example 4: External storage (NFS)

```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=10.0.0.10,rw \
  --opt device=:/data \
  mynfsvolume
```

👉 Data stored on remote server

---

### 🛠 Common Volume Drivers

* local (default)
* nfs
* aws-ebs
* azure-disk
* custom plugins

---

### ✅ Advantages

* Data persistence
* Backup possible
* Shareable
* External storage support

---

## 🔥 Key Differences

| Feature          | Storage Driver        | Volume Driver            |
| ---------------- | --------------------- | ------------------------ |
| Purpose          | Container filesystem  | Persistent storage       |
| Scope            | Internal layers       | External storage         |
| Lifecycle        | Tied to container     | Independent              |
| Data Persistence | ❌ No                  | ✅ Yes                    |
| Performance      | Slower (CoW overhead) | Faster for heavy I/O     |
| Usage            | OS + app runtime      | Databases, logs, uploads |

---

## 🧪 Combined Example (Real-world)

### Scenario: MySQL container

```bash
docker run -d -v mysql-data:/var/lib/mysql mysql
```

👉 Inside container:

* `/var/lib/mysql` → volume (persistent)
* `/etc/mysql` → storage driver (temporary)

---

### What happens:

| Path               | Managed by     |
| ------------------ | -------------- |
| /var/lib/mysql     | Volume driver  |
| rest of filesystem | Storage driver |

---

## 🧠 Real-World Analogy

* **Storage Driver** → Temporary workspace (like RAM/temp files)
* **Volume Driver** → Permanent storage (like SSD/HDD)

---

## 🎯 Final Summary

* Storage driver = manages container filesystem (temporary)
* Volume driver = manages persistent storage (permanent)

---

## ⚡ One-line Answer

👉 Storage drivers handle container layers, while volume drivers handle persistent data storage outside containers.

---







#### Docker Reference Docs

- https://docs.docker.com/engine/extend/legacy_plugins/
- https://github.com/rexray/rexray

