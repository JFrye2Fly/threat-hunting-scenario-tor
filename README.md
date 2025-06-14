# threat-hunting-scenario-tor

<img width="400" src="https://github.com/user-attachments/assets/44bac428-01bb-4fe9-9d85-96cba7698bee" alt="Tor Logo with the onion and a crosshair on it"/>

# Threat Hunt Report: Unauthorized TOR Usage
- [Scenario Creation](https://github.com/joshmadakor0/threat-hunting-scenario-tor/blob/main/threat-hunting-scenario-tor-event-creation.md)

## Platforms and Languages Leveraged
- Windows 10 Virtual Machines (Microsoft Azure)
- EDR Platform: Microsoft Defender for Endpoint
- Kusto Query Language (KQL)
- Tor Browser

##  Scenario

Management suspects that some employees may be using TOR browsers to bypass network security controls because recent network logs show unusual encrypted traffic patterns and connections to known TOR entry nodes. Additionally, there have been anonymous reports of employees discussing ways to access restricted sites during work hours. The goal is to detect any TOR usage and analyze related security incidents to mitigate potential risks. If any use of TOR is found, notify management.

### High-Level TOR-Related IoC Discovery Plan

- **Check `DeviceFileEvents`** for any `tor(.exe)` or `firefox(.exe)` file events.
- **Check `DeviceProcessEvents`** for any signs of installation or usage.
- **Check `DeviceNetworkEvents`** for any signs of outgoing connections over known TOR ports.

---

## Steps Taken

### 1. Searched the `DeviceFileEvents` Table

Searched for any file that had the string "tor" in it and discovered what looks like the user "employee" downloaded a TOR installer, did something that resulted in many TOR-related files being copied to the desktop, and the creation of a file called `tor-shopping-list.txt` on the desktop at `2025-06-04T14:11:37.0305131Z`. These events began at `2025-06-04T14:11:37.0305131Z`.

**Query used to locate events:**

```kql
DeviceFileEvents
| where DeviceName == "jeffreywindows1"
| where FileName contains "tor" 
| where InitiatingProcessAccountName == "fryecyber12345!"
| where Timestamp >= datetime('2025-06-04T14:11:37.0305131Z')
| order by TimeGenerated desc 
| project Timestamp, DeviceName, ActionType, FileName, FolderPath, SHA256, Account = InitiatingProcessAccountName
```
<img width="1261" alt="Screen Shot 2025-06-06 at 10 30 35 PM" src="https://github.com/user-attachments/assets/da6ca243-9e67-4333-9e1e-a1a060936e9a" />


---

### 2. Searched the `DeviceProcessEvents` Table

Searched for any `ProcessCommandLine` that contained the string "tor-browser-windows-x86_64-portable-14.0.1.exe". Based on the logs returned, at `2025-06-04T14:14:46.1007736Z`, an employee on the "jeffreywindows1" device ran the file `tor-browser-windows-x86_64-portable-14.0.1.exe` from their Downloads folder, using a command that triggered a silent installation.

**Query used to locate event:**

```kql

DeviceProcessEvents
| where DeviceName == "jeffreywindows1"
| where FileName contains "tor-browser-windows-x86_64-portable-14.5.3.exe"
| project Timestamp, DeviceName, ActionType, FolderPath, SHA256, ProcessCommandLine
```
<img width="1280" alt="Screen Shot 2025-06-06 at 10 33 36 PM" src="https://github.com/user-attachments/assets/5dfcd014-b92b-4812-99a7-95cd4a5edeaf" />

---

### 3. Searched the `DeviceProcessEvents` Table for TOR Browser Execution

Searched for any indication that user "employee" actually opened the TOR browser. There was evidence that they did open it at `2025-06-04T14:15:36.4032172Z`. There were several other instances of `firefox.exe` (TOR) as well as `tor.exe` spawned afterwards.

**Query used to locate events:**

```kql
DeviceProcessEvents
| where DeviceName == "jeffreywindows1"
| where FileName has_any ("tor.exe", "firefox.exe", "tor-browser.exe")
| order by TimeGenerated desc
| project Timestamp, DeviceName, ActionType, FolderPath, SHA256, ProcessCommandLine
```
<img width="1266" alt="Screen Shot 2025-06-06 at 10 36 39 PM" src="https://github.com/user-attachments/assets/bca6a34a-4a91-4575-a1f7-5001eb63fd22" />

---

### 4. Searched the `DeviceNetworkEvents` Table for TOR Network Connections

Searched for any indication the TOR browser was used to establish a connection using any of the known TOR ports. At `June 4, 2025, at 2:17 PM (UTC)`, an employee on the "jeffreywindows1" device successfully established a connection to the remote IP address `77.174.164.37` on port `9001`. The connection was initiated by the process `tor.exe`, located in the folder `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`. There were a couple of other connections to sites over port `443`.

**Query used to locate events:**

```kql
DeviceNetworkEvents
| where DeviceName == "jeffreywindows1"
| where RemotePort in ("9001", "9030", "9040", "9050", "9051", "9150")
| project Timestamp, DeviceName, InitiatingProcessFileName, InitiatingProcessFolderPath, ActionType, RemoteIP, RemotePort
| order by Timestamp desc
```
<img width="1275" alt="Screen Shot 2025-06-06 at 10 38 43 PM" src="https://github.com/user-attachments/assets/5a2d325d-0ac0-4db8-8cf2-c780ea2a2449" />

---

## Chronological Event Timeline 

### 1. File Download - TOR Installer

- **Timestamp:** `2025-06-04T14:11:37.0305131Z`
- **Event:** The user "fryecyber12345!" downloaded a file named `tor-browser-windows-x86_64-portable-14.0.1.exe` to the Downloads folder.
- **Action:** File download detected.
- **File Path:** `C:\Users\employee\Downloads\tor-browser-windows-x86_64-portable-14.0.1.exe`

### 2. Process Execution - TOR Browser Installation

- **Timestamp:** `2025-06-04T14:14:46.1007736Z`
- **Event:** The user "fryecyber12345!" executed the file `tor-browser-windows-x86_64-portable-14.0.1.exe` in silent mode, initiating a background installation of the TOR Browser.
- **Action:** Process creation detected.
- **Command:** `tor-browser-windows-x86_64-portable-14.0.1.exe /S`
- **File Path:** `C:\Users\employee\Downloads\tor-browser-windows-x86_64-portable-14.0.1.exe`

### 3. Process Execution - TOR Browser Launch

- **Timestamp:** `2025-06-04T14:15:36.4032172Z`
- **Event:** User "fryecyber12345!" opened the TOR browser. Subsequent processes associated with TOR browser, such as `firefox.exe` and `tor.exe`, were also created, indicating that the browser launched successfully.
- **Action:** Process creation of TOR browser-related executables detected.
- **File Path:** `C:\Users\employee\Desktop\Tor Browser\Browser\TorBrowser\Tor\tor.exe`

### 4. Network Connection - TOR Network

- **Timestamp:** `June 4, 2025, at 2:17 PM (UTC)`
- **Event:** A network connection to IP `77.174.164.37` on port `9001` by user "employee" was established using `tor.exe`, confirming TOR browser network activity.
- **Action:** Connection success.
- **Process:** `tor.exe`
- **File Path:** `c:\users\employee\desktop\tor browser\browser\torbrowser\tor\tor.exe`

### 5. Additional Network Connections - TOR Browser Activity

- **Timestamps:**
  - `2025-06-04T14:18:26.5463381Z` - Connected to `77.174.164.37` on port `443`.
  - `2025-06-05T11:37:04.7710084Z` - Local connection to `127.0.0.1` on port `9150`.
- **Event:** Additional TOR network connections were established, indicating ongoing activity by user "employee" through the TOR browser.
- **Action:** Multiple successful connections detected.

### 6. File Creation - TOR Shopping List

- **Timestamp:** `2025-06-05T11:37:04.7710084Z`
- **Event:** The user "fryecyber12345!" created a file named `tor_shopping_list.txt` on the desktop, potentially indicating a list or notes related to their TOR browser activities.
- **Action:** File creation detected.
- **File Path:** `C:\Users\employee\Desktop\tor_shopping_list.txt`

---

## Summary

The user "fryecyber12345!" on the "jeffreywindows1" device initiated and completed the installation of the TOR browser. They proceeded to launch the browser, establish connections within the TOR network, and created various files related to TOR on their desktop, including a file named `tor_shopping_list.txt`. This sequence of activities indicates that the user actively installed, configured, and used the TOR browser, likely for anonymous browsing purposes, with possible documentation in the form of the "shopping list" file.

---

## Response Taken

TOR usage was confirmed on the endpoint `jeffreywindows1` by the user `fryecyber12345!`. The device was isolated, and the user's direct manager was notified.

---
