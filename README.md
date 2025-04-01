# TP-Link IoT Security Analysis

## Overview
This repository documents an analysis of TP-Link IoT security infrastructure, focusing on API endpoints, authentication mechanisms, and potential vulnerabilities.

---

## UART

### CPU Information

![Alt text](images/PXL_20250401_151445861.MP.jpg)

**CPU Model:** `Qstar AL32760B` (ARM Architecture)

Since initial attempts to find a potential UART interface were unsuccessful, I examined the CPU pin layout to locate UART pins. The CPU pin scheme revealed three potential access points for UART.

**Pin Layout (Photo Reference)**

After testing, the working `TX` and `RX` pins were identified.

To interface with the device, I connected pogo pins to the UART points and determined that the baud rate is **115200**.

Once the camera booted up, the terminal prompted for a password, which restricted access to a bash shell.

### Bypassing the Password
To bypass the password and gain consistent shell access, I modified the firmware. The first step was to dump the firmware.

The firmware is a **SquashFS binary** containing a **Linux OS**. Looking at the file `/etc/inittab`:

```sh
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K shutdown
ttyS0::askfirst:/bin/login.sh
```

The `login.sh` script is responsible for handling the login request. To bypass authentication, I modified `inittab` as follows:

```sh
::sysinit:/etc/init.d/rcS S boot
::shutdown:/etc/init.d/rcS K shutdown
ttyS0::askfirst:/bin/sh
```

This allowed direct access to the shell.

---

## Open Ports
The camera has several open ports:

- **2020** - SOAP (handles camera features like motion detection)
- **8800** - HTTP server
- **554** - RTSP

---

## ELF Files
While analyzing running processes, three significant ELF binaries stood out:

- **`Cloud-iot`** – Manages communication between the mobile app and the camera.
- **`Cloud-Service`** – Handles connections between the camera and TP-Link's cloud servers.
- **`cer`** – Processes all network packets entering and leaving the camera.

Using **Ghidra** to reverse engineer `cer`, I initially searched for **string manipulation vulnerabilities** or **buffer overflows**, but found nothing exploitable. However, I discovered an interesting behavior: `cer` logs data to `/tmp/cer.log`, but the file did not exist in `/tmp` while the process was running.

### Debugging `cer`
To investigate further, I manually created `/tmp/cer.log`. Once the log file was available, valuable information surfaced, including:

1. How the camera communicates with the mobile app.
2. The app sends a **local broadcast** to nearby devices.
3. The camera responds and establishes a **TLS connection**.
4. Each authentication attempt generates a **new digest request**.
5. Once authenticated, the **token remains static** until the camera reboots.

#### Example Authentication Request
```http
POST /stream HTTP/1.1
Host: 192.168.10.80:8800
User-Agent: Dalvik/2.1.0 (Linux; U; Android 15; Pixel 6a Build/AP4A.250105.002)
Authorization: Digest username="admin", realm="TP-Link IP-Camera", uri="/stream", algorithm=MD5, nonce="c14f82dba49979e91292ccca99b4b98000029e1e12ee9b92", nc=00000001, cnonce="a9h5b7i3j2y8c0a6", qop=auth, response="e5f3a8c2d8b9e1f0d9c3b2a5c7f8e4d6", opaque="64943214654649846565646421"
Content-Type: multipart/mixed; boundary=--client-stream-boundary--
Connection: keep-alive
Content-Length: 520

----client-stream-boundary--
Content-Type: application/json
Content-Length: 74

{"type":"response", "seq":1, "params":{"error_code":0, "session_id":"10"}}
```

Although the camera recognized and trusted my request, I encountered issues with the **stream boundaries**, suggesting that I needed to capture the full authentication request from the app.

---

## Discovered API Endpoints
During the research, several key API endpoints used by TP-Link IoT devices were identified:

- `https://n-euw1-device-api.tplinkcloud.com`
- `https://euw1-event-gw.iot.i.tplinknbu.com/v1/things/events`
- `https://euw1-device-telemetry-gw.iot.i.tplinknbu.com/v1/things/history`
- `https://euw1-app-server.iot.i.tplinknbu.com`
- `https://euw1-event-gw.iot.i.tplinknbu.com/v1/things/shadows`

---

## JWT Generation
To generate a **JWT token**, the device sends an authentication request to:

```bash
curl -k -X POST "https://security.iot.i.tplinknbu.com/v1/auth/device" \
  -H "Content-Type: application/json" \
  -d '{
        "alias": "Tapo_C210_F029",
        "deviceId": "802176EB0BFCEB8D69943768599845B02280E74B",
        "deviceMac": "242FD0E7F029",
        "deviceName": "C210",
        "deviceType": "SMART.IPCAMERA",
        "fwId": "089209928F984D73D3108E7AFFBA2C1A",
        "fwVer": "1.3.11 Build 240427 Rel.61418n(4555)",
        "hwId": "5FAAA6EC6A1A74A8B50B0C3302FB26FD",
        "hwVer": "2.0",
        "isSupportTCSP": false,
        "model": "C210",
        "oemId": "A324FA71283989909AAB5B8646626D4C"
    }'
```

#### Server Response
```json
{
    "jwt": "jwt|eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiIsImtpZCI6InNLS3RoMFIxIn0...",
    "jwtExpiresIn": 2592000,
    "deviceToken": "Cc79927de8cfc406f84ee0926d507181",
    "deviceTokenExpiresIn": 86400
}
```

Using the **JWT**, we can access other API endpoints:

```bash
curl -k -X POST "https://euw1-event-gw.iot.i.tplinknbu.com/v1/things/events" \
     -H "Authorization: Bearer Cfaadc94284474c2cba9831dd36e2ed2" \
     -H "Content-Type: application/json" \
     -d '{
           "eventType": "motion_detected",
           "deviceId": "802176EB0BFCEB8D69943768599845B02280E74B",
           "thingName": "Camera_TPLink",
           "region": "euw1",
           "timestamp": 1741710000
         }'
```

However, responses returned `invalid jwt`, preventing further progress.

---

## Fuzzing
I attempted fuzzing additional API endpoints without success:

- `https://n-euw1-device-api.tplinkcloud.com`
- `https://euw1-app-server.iot.i.tplinknbu.com`

Further investigation is required to identify additional vulnerabilities.

---

### **End of Analysis**
