# Monitoring a home network with LibreNMS

These are my notes on setting up [LibreNMS](https://www.librenms.org/) to monitor my home network over SNMP and send alerts to Telegram. I wrote them as a step-by-step guide, so you can follow along and do the same on your own network.

![Dashboard](./images/dashboard.png)

## What I'm monitoring

LibreNMS runs in an LXC container on a Proxmox mini PC. It polls four devices on the same network (`192.168.10.0/24`) using SNMP v2c:

| Name in LibreNMS | Device | IP | Notes |
|---|---|---|---|
| `Fortigate 40f` | FortiGate 40F, FortiOS 7.2.11 | `192.168.10.1` | Firewall. WAN (1 Gbps) and a 2 Gbps LACP link to the switch |
| `Switch Cisco` | Cisco SG300-10PP | `192.168.10.8` | 10-port PoE+ switch. The LACP group is `Po1` |
| `Proxmox SNMP Agent (Mini PC)` | Proxmox VE host | `192.168.10.105` | Runs LibreNMS. SNMP comes from `snmpd` |
| `Netgear AP` | WAX214 access point | `192.168.10.108` | LibreNMS detects it as an EnGenius access point |

LibreNMS itself is at `192.168.10.111`.

```mermaid
flowchart TB
    NET([Internet]) -->|"WAN 1 Gbps"| FGT["FortiGate 40F<br/>192.168.10.1"]
    FGT ==>|"LACP Po1 · 2 Gbps"| SW["Cisco SG300-10PP<br/>192.168.10.8"]
    SW --- AP["Wireless AP<br/>192.168.10.108"]
    subgraph PVE["Proxmox VE host · 192.168.10.105 · snmpd"]
        NMS["LibreNMS (LXC 102)<br/>192.168.10.111"]
    end
    SW --- PVE
```

Everything is on the same LAN, so LibreNMS talks to each device directly on UDP port 161.

**What you need:** a Proxmox host (or any Linux machine or VM) to run LibreNMS, devices that support SNMP, and a Telegram account for the alerts.

The steps:

1. Install LibreNMS
2. Set up SNMP on the Proxmox host
3. Enable SNMP on the other devices and add them
4. Optional: look at the SNMP traffic
5. Cut the noise
6. Set up Telegram alerts
7. Test the alerts
8. Build a dashboard

---

## 1. Install LibreNMS

I run LibreNMS in its own LXC container. That keeps it separate from the Proxmox host, I can snapshot it before an upgrade, and it has a fixed IP address.

This is what I gave the container:

| Setting | Value |
|---|---|
| CT ID / hostname | `102` / `NMS` |
| Cores | 2 |
| Memory / swap | 4 GB / 512 MiB |
| Root disk | 64 GB (`local-lvm`) |
| Address | `192.168.10.111/24`, gateway `192.168.10.1` |

![LXC resources in Proxmox](./images/proxmox-lxc-resources.png)

Before installing anything, check that the container has an address, a default route and Internet access:

```bash
ip address
ip route
ping 1.1.1.1
```

![Network check inside the container](./images/lxc-network-check.png)

Install the dependencies:

```bash
apt update && apt install -y \
  acl ca-certificates curl fping git graphviz imagemagick \
  mariadb-server mariadb-client mtr-tiny nginx nmap \
  php-fpm php-cli php-curl php-gd php-gmp php-mbstring \
  php-mysql php-snmp php-xml php-zip php-bcmath php-intl \
  php-pear python3-dotenv python3-pymysql python3-redis \
  python3-setuptools python3-systemd python3-urllib3 \
  rrdtool snmp snmpd unzip whois
```

The rest of the installation follows the [official guide](https://docs.librenms.org/Installation/Install-LibreNMS/), so I won't copy every command here. These are the steps and what each one is for:

1. Create a `librenms` user and clone the repository into `/opt/librenms`. LibreNMS should not run as root.
2. Install the PHP dependencies as that user with `./scripts/composer_wrapper.php install --no-dev`.
3. Create the database. LibreNMS needs two MariaDB options (`innodb_file_per_table=1` and `lower_case_table_names=0`), then:

   ```sql
   CREATE DATABASE librenms CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'librenms'@'localhost' IDENTIFIED BY '<DB_PASSWORD>';
   GRANT ALL PRIVILEGES ON librenms.* TO 'librenms'@'localhost';
   FLUSH PRIVILEGES;
   ```

4. Set the PHP timezone and configure php-fpm and nginx so the web interface works.
5. Install the cron and logrotate files. The cron job is what runs the polling.
6. Open `http://192.168.10.111/install` and go through the web installer.

On the database page of the installer, use the values from step 3:

![Web installer, database page](./images/librenms-installer-database.png)

| Field | Value |
|---|---|
| Host / port | `localhost` / `3306` |
| User / database | `librenms` / `librenms` |
| Password | the one you chose |

**One setting I changed.** LibreNMS shows a device as *warning* if its uptime is below a threshold. By default that is 24 hours (`86400` s), so after any reboot the device stays yellow for a day. I set it to `300` s. You find it under the gear icon → `Global Settings` → `Web UI` → `Device Settings`, in *Show Device as warning if Uptime below (seconds)*.

![uptime_warning setting](./images/uptime-warning.png)

---

## 2. Set up SNMP on the Proxmox host

LibreNMS can only read what an SNMP agent answers, and Proxmox doesn't come with one set up. I installed `snmpd` on the host:

```bash
apt install snmpd
```

Then edit `/etc/snmp/snmpd.conf`:

```conf
# Listen on all interfaces, UDP 161 (the default only listens on localhost)
agentaddress udp:161

# Read-only community, answered only to the LibreNMS server
rocommunity <COMMUNITY> 192.168.10.111

# Shown in the device overview in LibreNMS
sysLocation Homelab
sysContact  admin@example.com
```

- `agentaddress` makes `snmpd` listen on the network. Without it LibreNMS can't reach it.
- `rocommunity` is the password for read-only access. The IP at the end limits it to the LibreNMS server, so nobody else on the LAN can use it.
- Comment out the default `rocommunity public ...` lines that come with the package.

![snmpd agentaddress](./images/snmpd-agentaddress.png)

Restart `snmpd` and make it start on boot:

```bash
systemctl restart snmpd
systemctl enable snmpd
```

![Active snmpd.conf lines and service status](./images/snmpd-conf.png)

To check it works, run this from the LibreNMS container. You should get the system description back:

```bash
snmpwalk -v2c -c <COMMUNITY> 192.168.10.105 sysDescr
```

---

## 3. Enable SNMP on the other devices and add them

Each device needs three things: SNMP turned on, a read-only community, and permission for `192.168.10.111` to query it. SNMP communities are case-sensitive, so the string you enter in LibreNMS has to match exactly what you set on the device.

| Device | Where | What to set |
|---|---|---|
| FortiGate 40F | `Network → Interfaces` | On the interface LibreNMS reaches (`lan`), tick **SNMP** under Administrative Access |
| | `System → SNMP` | Enable the SNMP agent and create a v1/v2c community. I enabled v2c queries and disabled v1, and allowed host `192.168.10.111/32` |
| Cisco SG300 | `SNMP → Communities` | Add a community: type *Basic*, access mode *Read Only*, management station `192.168.10.111` |
| Wireless AP | SNMP settings page | Enable SNMP and set the read-only community |

![FortiGate interface with SNMP access](./images/fortigate-interface-snmp.png)
![FortiGate SNMP community](./images/fortigate-snmp-community.png)
![Cisco SG300 communities](./images/sg300-snmp.png)
![Access point SNMP settings](./images/ap-snmp-settings.png)

Now add the devices in LibreNMS with `Devices → Add Device`, once per device:

| Field | Value |
|---|---|
| Hostname or IP | the device IP |
| SNMP | ON |
| SNMP Version | `v2c` |
| Transport | `udp` (leave the port empty) |
| Community | the one you set on that device |

![Add Device form](./images/add-device.png)

You can also do it from the command line with `lnms device:add <hostname or ip>` (see `lnms device:add --help` for the SNMP options).

After adding each one, open `Device → Edit` and set a **Display Name**. That name is what you will see in the dashboard and in the alerts, instead of an IP.

![Device list](./images/devices-list.png)

---

## 4. Optional: look at the SNMP traffic

I wanted to see what the polling looks like on the wire, so I mirrored a switch port and captured it with Wireshark. You don't need this for the setup.

On the SG300 I used `Port and VLAN Mirroring` to copy the traffic of `GE3` (the port of the mini PC that runs Proxmox, LibreNMS and my other services) to `GE4`, where my laptop was connected. I mirrored both directions (Tx and Rx).

![Port mirroring on the SG300](./images/snmp-capture-mirror.png)

In Wireshark, filter by `snmp`. You see LibreNMS (`192.168.10.111`) sending `get-request` packets to the other devices and each one answering with a `get-response`.

![SNMP polling in Wireshark](./images/snmp-capture-overview.png)

Opening one request and its answer, both to the FortiGate (packets 6216 and 6217): it's SNMP v2c on UDP port 161, the answer has `error-status: noError`, and the community is in the packet in clear text (I blurred it).

![SNMP request](./images/snmp-capture-request.png)
![SNMP response](./images/snmp-capture-response.png)

You won't see the Proxmox host (`192.168.10.105`) in this capture. LibreNMS runs on that same host, so that traffic never leaves it.

---

## 5. Cut the noise

Out of the box, LibreNMS alerts about anything that is down, including things that are down on purpose. If you leave that alone you end up ignoring the alerts. I did two things.

**Unused ports.** The *Port status up/down* rule fires for every port that is down. Ports with no cable and VLAN interfaces I don't route are down all the time, so I marked them as **Ignore** in `Device → Edit → Ports`. LibreNMS keeps polling them, it just stops alerting.

| Device | Ignored ports |
|---|---|
| Fortigate 40f | `lan1`, `a`, `lan`, `fortilink` |
| Switch Cisco | `gi4` to `gi8`, and VLAN interfaces `10`, `20`, `30`, `99`, `999` |

![Ignored ports on the FortiGate](./images/port-ignore-fortigate.png)
![Ignored ports on the Cisco switch](./images/port-ignore-cisco.png)

**License sensors.** FortiOS reports one *Days left for ...* sensor per subscription (30 of them). I don't have an active Fortinet contract, so they are all at `-295` days, and the *Sensor under limit* rule kept firing for them. I turned **Alerts** off for each of those sensors in `Device → Edit → Health`.

![License sensors with alerts turned off](./images/health-license-sensors.png)

---

## 6. Set up Telegram alerts

In LibreNMS an alert goes through four pieces. A **rule** decides when something is wrong. An **operation** decides who gets notified and when. A **transport** is the channel (Telegram here). A **template** is the text of the message. A rule with no operation raises the alert but sends nothing.

### Create the bot

1. In Telegram, talk to `@BotFather`, send `/newbot` and follow the questions. Keep the token it gives you.
2. Send `/start` to your new bot, so it is allowed to write to you.
3. Get your chat ID by opening `https://api.telegram.org/bot<BOT_TOKEN>/getUpdates` in a browser. It's the `id` inside `"chat"`.

![BotFather](./images/botfather.png)

### Create the transport

`Alerts → Alert Transports → Create alert transport`:

| Field | Value |
|---|---|
| Transport name | `Bot` |
| Transport type | Telegram |
| Default Alert | ON |
| Chat ID | your chat ID |
| Thread ID | empty |
| Token | your bot token |
| Format | HTML |
| Send PNG Graph as | photo |

![Telegram transport](./images/telegram-transport.png)

Press the check button (*Test transport*) to send a test message:

![Test transport button](./images/transport-test-button.png)
![Test message in Telegram](./images/telegram-test-alert.png)

### Create the operation

`Alerts → Alert Operations → Create operation`. Mine is called `Send to telegram`, with a default step duration of `300`, one segment (steps `1` to `1`, start `1` s, step duration `300` s) and the `Telegram: Bot` transport.

![Alert operation](./images/alert-operation.png)

### Link the rules

Open each rule in `Alerts → Alert Rules` and pick the operation in its **Operation** field.

![Choosing the operation in a rule](./images/alert-rule-operation.png)

These are the rules I have enabled:

| Rule | Fires when |
|---|---|
| CPU usage above 90% | CPU usage is 90% or more |
| RAM usage above 90% | RAM usage is 90% or more |
| Storage usage above 80% | disk usage is 80% or more |
| Device Down (SNMP unreachable) | the device doesn't answer SNMP |
| Device Down! Due to no ICMP response. | the device doesn't answer ping |
| Device rebooted | uptime is below 300 s |
| Ping Latency | ping latency is above 60 ms |
| Port status up/down | a port is down |
| Port utilisation over threshold | a port is at 80% or more of its speed |
| Sensor over limit / Sensor under limit | a sensor goes above its high limit or below its low limit |
| Service up/down | a monitored service is down |
| State Sensor Critical | a state sensor reports critical |
| Wireless Sensor over limit / under limit | the same, for wireless sensors |

![Alert rules list](./images/alert-rules-list.png)

One thing that tripped me up: on threshold rules (CPU, RAM, storage) choose **greater or equal**, not **equal**. With *equal* the rule only fires when the value is exactly 90, so it never fires in practice.

### Replace the message template

The default alert message is long and prints raw SNMP fields (`sysObjectID`, `snmpEngineID` and so on):

![Default alert message](./images/telegram-default.png)

I replaced it in `Alerts → Alert Templates → Default Alert Template` with a short one. It shows the state, rule name, host and time, plus one line for each port, sensor, disk, RAM or CPU that caused the alert:

```html
@if ($alert->state == 0)
🟢 <b>RESOLVED: {{ $alert->name }}</b>
@elseif ($alert->state == 1)
🔴 <b>ALERT: {{ $alert->name }}</b>
@elseif ($alert->state == 2)
🟡 <b>ACK: {{ $alert->name }}</b>
@endif
<b>Host:</b> {{ $alert->display }} (<code>{{ $alert->hostname }}</code>)
<b>Time:</b> {{ $alert->timestamp }} ({{ $alert->elapsed }})
@if ($alert->faults)

@foreach ($alert->faults as $key => $value)
@if ($value['ifDescr'])
• <b>Port:</b> <code>{{ $value['ifDescr'] }}</code> @if ($value['ifAlias'] && $value['ifAlias'] != $value['ifDescr'])({{ $value['ifAlias'] }})@endif — {{ $alert->state == 0 ? 'up' : $value['ifOperStatus'] }}
@elseif ($value['sensor_descr'])
• <b>Sensor:</b> {{ $value['sensor_descr'] }} (<code>{{ $value['sensor_current'] }}</code>)
@elseif ($value['storage_descr'])
• <b>Disk:</b> {{ $value['storage_descr'] }} (<code>{{ $value['storage_perc'] }}%</code>)
@elseif ($value['mempool_descr'])
• <b>RAM:</b> {{ $value['mempool_descr'] }} (<code>{{ $value['mempool_perc'] }}%</code>)
@elseif ($value['processor_descr'])
• <b>CPU:</b> <code>{{ $value['processor_usage'] }}%</code>
@endif
@endforeach
@endif
```

The part at the end of the `Port` line is there because with a plain `{{ $value['ifOperStatus'] }}` the recovery message still said `down` next to the port. Now it prints `up` when the alert is resolved (state 0) and the real status otherwise.

![Template editor](./images/alert-template-editor.png)

This is what an alert looks like with it:

![Alert with the new template](./images/telegram-minimal.png)

---

## 7. Test the alerts

The easy way to know it all works is to break something. I unplugged one cable of the LACP link between the FortiGate and the switch (`lan2` on the FortiGate, `gigabitethernet9` on the switch), waited, and plugged it back in.

LibreNMS only notices on its next polling cycle (every 5 minutes by default), so give it a few minutes. The ports show up on the Faults page (this capture is from an earlier run of the same test):

![Faults page](./images/lacp-test-faults.png)

And the alerts arrive in Telegram, one per device. In my run they came at 21:20:

![Alerts in Telegram](./images/lacp-test-alert.png)

Plug the cable back in and, after the next poll, you get the recovery messages (21:25 in my run):

![Recovery messages in Telegram](./images/lacp-test-resolved.png)

If you don't want to touch cables, you can also run these from `/opt/librenms` in the container:

```bash
# Check the installation
./validate.php

# Send a real alert through the transport
./scripts/test-alert.php -r <rule_id> -h <hostname_or_device_id> -d

# Show how a template renders for a device and a rule
./scripts/test-template.php -t <template_id> -d -h <hostname> -r <rule_id>
```

`test-template.php` only shows the alert message, not the recovery one. For that you need a real fault, like the cable.

If an alert shows up in LibreNMS but nothing reaches Telegram, check that the rule has an operation, and press *Test transport* again.

---

## 8. Build a dashboard

I wanted one page that shows everything at a glance. On the Overview page, use the **+** next to the dashboard selector to create one (mine is called *Homelab NOC*), then *Edit* to add widgets:

| Widget | What it shows |
|---|---|
| Availability Map | which devices are up, warning or down |
| Alerts | active alerts (empty when all is fine) |
| Graph: *WAN interface* | traffic on the FortiGate `wan` port (1 Gbps) |
| Graph: *LACP FORTIGATE (LAN)* | traffic on the FortiGate LACP link (2 Gbps) |
| Top Devices | CPU load per device |
| Top Interfaces | the busiest interfaces in the last 15 minutes |
| Device Summary Horizontal | totals of devices, ports and sensors |

![Dashboard summary widgets](./images/dashboard-summary.png)

Two more pages are useful to check that data is coming in: `Health → Processor` for CPU on every device, and the overview page of each device.

![CPU per device](./images/health-processors.png)
![Overview of the switch](./images/switch-overview.png)

---

## Security notes

- Use read-only communities and limit them to the LibreNMS IP (the IP at the end of `rocommunity`, and the allowed host on the FortiGate and the SG300).
- Don't leave the default strings (`public`, `private`). My access point also has a read-write community field: leave it empty or change it.
- SNMP v2c sends the community in clear text, as the capture in step 4 shows. Keep UDP 161 inside your LAN. SNMPv3 is the safer option if you can use it.
- Hide communities, bot tokens and chat IDs before you share screenshots.
