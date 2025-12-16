# Multi-UE Emulation using srsRAN + ZMQ + GNU Radio

### Complete Installation & Execution Guide (Single Ubuntu System)

---

## 📌 Project Overview

This project builds a **fully virtual 5G network on a single Ubuntu machine** using software-defined radio concepts.

There is **no RF hardware** involved:

* ❌ No USRP
* ❌ No SIM
* ❌ No antenna

Everything runs **entirely in software**.

The setup validates:

* UE registration
* RRC & NAS procedures
* PRACH contention
* Scheduling of multiple UEs
* Paging capability
* End-to-end 5G signaling

---

## 🔗 Reference Links Used

1. **GitHub Demo (VM readiness, UE build, workflow)**
   [https://github.com/devopsjourney23/my-srsproject-demo?tab=readme-ov-file#02-vm-machine-readiness](https://github.com/devopsjourney23/my-srsproject-demo?tab=readme-ov-file#02-vm-machine-readiness)

2. **Official srsRAN Multi-UE Documentation**
   [https://docs.srsran.com/projects/project/en/latest/tutorials/source/srsUE/source/index.html#multi-ue-emulation](https://docs.srsran.com/projects/project/en/latest/tutorials/source/srsUE/source/index.html#multi-ue-emulation)

> These references are followed exactly, with corrections based on real execution and debugging.

---

## 🧩 Components Implemented

* **5G Core (5GC)** → Open5GS (Docker-based)
* **gNB (5G Base Station)** → srsRAN Project
* **UEs (Phones)** → srsUE (from srsRAN_4G)
* **RF Medium** → ZeroMQ (ZMQ)
* **Multi-UE RF Fabric** → GNU Radio

---

## 🖥️ System Assumptions

| Item | Value                       |
| ---- | --------------------------- |
| OS   | Ubuntu 22.04.1 LTS (64-bit) |
| User | `r-309`                     |
| Home | `/home/r-309`               |
| Disk | ≥ 50 GB                     |
| RAM  | ≥ 8 GB                      |
| CPU  | ≥ 4 cores                   |

---

## 📁 Directory Layout (Do **NOT** Change)

```
/home/r-309/
├── srsRAN_Project/                 → srsRAN Project (5G gNB)
│   ├── build/
│   ├── docker/                     → Open5GS 5GC
│
├── srsRAN_4G/                      → srsRAN 4G (UE source)
│   ├── srsue/
│   ├── build/
│   └── lib/
│
├── srsRAN_config/                  → Runtime configs
│   ├── gnb_zmq.yaml
│   ├── ue1_zmq.conf
│   ├── ue2_zmq.conf
│   ├── ue3_zmq.conf
│   └── multi_ue_scenario.grc
│
└── /usr/bin/
    └── srsue                       → Installed UE binary
```

**Why this separation matters**

* Clean builds
* No config pollution
* Docker and RF stacks stay isolated

---

## 🔧 Phase 1 — VM / System Readiness

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 📦 Phase 2 — Install Required Packages

```bash
sudo apt install -y \
  git cmake build-essential pkg-config \
  libfftw3-dev libmbedtls-dev \
  libboost-program-options-dev \
  libconfig++-dev libsctp-dev \
  libzmq3-dev \
  gnuradio gnuradio-dev \
  python3 python3-pip \
  net-tools iproute2
```

---

## 🐳 Phase 3 — Docker Installation (Official Repository)

### Remove conflicting packages

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

### Add Docker GPG key

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### Add Docker repository

```bash
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

### Install Docker

```bash
sudo apt-get install -y \
  docker-ce docker-ce-cli \
  containerd.io docker-buildx-plugin \
  docker-compose-plugin
```

### Enable Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker r-309
```

⚠️ **Log out and log back in once**

Verify:

```bash
docker run hello-world
```

---

## 🏗️ Phase 4 — Build srsRAN Project (gNB, ZMQ Enabled)

```bash
cd ~
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project
mkdir build && cd build
cmake .. -DENABLE_ZMQ=ON
make -j$(nproc)
sudo make install
sudo ldconfig
```

Verify:

```bash
gnb --help
```

---

## 📱 Phase 4A — UE Installation (srsUE from srsRAN_4G)

### Install UE dependencies

```bash
sudo apt-get install -y \
  build-essential cmake \
  libfftw3-dev libmbedtls-dev \
  libboost-program-options-dev \
  libconfig++-dev libsctp-dev \
  git curl jq
```

### Clone and build UE

```bash
cd ~
git clone https://github.com/srsRAN/srsRAN_4G.git
cd srsRAN_4G
mkdir build
cd build
cmake ../ -DENABLE_EXPORT=ON -DENABLE_ZEROMQ=ON
make -j$(nproc)
```

### Install UE binary

```bash
sudo cp ~/srsRAN_4G/build/srsue/src/srsue /usr/bin/srsue
sudo chmod +x /usr/bin/srsue
```

Verify:

```bash
srsue --help
```

---

## 🌐 Phase 4B — UE Network Namespaces (Optional)

```bash
sudo ip netns add ue1
sudo ip netns add ue2
sudo ip netns add ue3
ip netns list
```

---

## 🔐 Phase 5 — Fix /tmp Permissions

```bash
sudo chmod 1777 /tmp
```

---

## 🧠 Phase 6 — Start 5G Core (Open5GS)

**Terminal-1**

```bash
cd ~/srsRAN_Project/docker
docker compose up --build 5gc
```

Leave this running.

---

## ⚙️ Phase 7 — Configuration Files

Location:

```bash
~/srsRAN_config/
```

Files:

* `gnb_zmq.yaml`
* `ue1_zmq.conf`
* `ue2_zmq.conf`
* `ue3_zmq.conf`
* `multi_ue_scenario.grc`

---

## ❗ Phase 8 — Mandatory gNB Configuration Fix

Edit `gnb_zmq.yaml`:

```yaml
prach:
  prach_config_index: 1
  total_nof_ra_preambles: 64
```

❌ **Do NOT add**

* `nof_ssb_per_ro`
* `nof_cb_preambles_per_ssb`

These cause:

```
Invalid nof. SSB per RACH occasion value
```

---

## ▶️ Phase 9 — Runtime Execution (6 Terminals)

### Terminal Allocation

| Terminal | Component   |
| -------- | ----------- |
| T1       | Open5GS 5GC |
| T2       | gNB         |
| T3       | GNU Radio   |
| T4       | UE-1        |
| T5       | UE-2        |
| T6       | UE-3        |

### Terminal-2 — gNB

```bash
cd ~/srsRAN_config
sudo pkill -9 gnb
sudo gnb -c gnb_zmq.yaml
```

Expected:

```
N2: Connection to AMF completed
```

---

### Terminal-3 — GNU Radio

```bash
sudo pkill -9 python3
sudo pkill -f multi_ue_scenario
sudo gnuradio-companion ~/srsRAN_config/multi_ue_scenario.grc
```

▶ Click **Run**

---

### Terminal-4 / 5 / 6 — UEs

```bash
sudo pkill -9 srsue
sudo srsue ue1_zmq.conf
sudo srsue ue2_zmq.conf
sudo srsue ue3_zmq.conf
```

Each UE has:

* unique IMSI
* unique ZMQ ports
* independent RACH & RRC

---

## ✅ Phase 10 — Successful UE Attachment

Typical UE output:

```
Random Access Complete. c-rnti=0x4602
RRC Connected
PDU Session Establishment successful. IP: 10.45.1.2
```

Meaning:

* PRACH success
* RRC connected
* Core + UPF working
* NR connection finalized

---

## 🎛️ Phase 11 — GNU Radio Control Panel

### Path-loss Control

* Per-UE path loss sliders
* Simulates distance effects

Enable UE trace logging with:

```
t
```

Observe:

* RSRP variation
* PHY behavior in real time

### Time Slow Down Ratio

* Controls IQ exchange speed
* Lower CPU load
* Higher RTT

Useful for multi-UE scaling.

---

## 🧪 Phase 12 — Cleanup

```bash
sudo pkill -9 gnb
sudo pkill -9 srsue
sudo pkill -9 python3
sudo pkill -f multi_ue_scenario
```

---

## ✅ Final Checklist

✔ gNB running
✔ srsUE installed
✔ GNU Radio active
✔ 3 UEs attached
✔ IPs assigned

---

## 🧠 Final Mental Model

```
Open5GS
   ↑
  N2/N3
   ↑
  gNB
   ↑
  ZMQ
   ↑
GNU Radio
   ↑
UE-1   UE-2   UE-3
