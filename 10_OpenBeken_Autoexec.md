# 10 — OpenBeken autonomous smart-plug automation


Configure OpenBeken smart plugs to synchronize with the local Termux NTP server (`192.168.31.2`) and run an autonomous night-tariff schedule. The relay turns ON at **23:02** and OFF at **06:58**, including when Internet access or Home Assistant is unavailable. The server address must match the Android host LAN address configured in [step 07](07_Chrony_NTP_Server.md); `192.168.31.2` is an example and may differ on your network.

#### 1. Where to Paste the Configuration

Open the web interface of the OpenBeken device by navigating to its local IP address in your browser (e.g., `http://192.168.31.xxx`).

1. Click **Launch Web Application** (or go directly to **LittleFS** / **File System**).
2. Open the **Filesystem** tab.
3. Locate and open `autoexec.bat` (if it does not exist, click **Create file** and name it `autoexec.bat`).
4. Paste the script below, click **Save**, and reboot the device via **Config → Restart** (or the **Restart** button on the main page).

> **Note:** OpenBeken also allows pasting startup commands in **Config → Configure Startup**, which writes directly to `autoexec.bat` in LittleFS. Editing `autoexec.bat` in the LittleFS tab is recommended for multi-line scripts.

---

#### 2. `autoexec.bat` Configuration


```bat
PowerSave 1
startDriver NTP
ntp_setServer 192.168.31.2
time_setTZ +02:00
SetChannel 1 0
SetChannel 5 0
addClockEvent 23:02 0xff 1 SetChannel 1 1
addClockEvent 06:58 0xff 2 SetChannel 1 0
addRepeatingEvent 30 -1 if $NTPOn==1&&$CH5==0&&$hour*60+$minute>=418&&$hour*60+$minute<1382 then backlog SetChannel 1 0; SetChannel 5 1
addRepeatingEvent 30 -1 if $NTPOn==1&&$CH5==0&&$hour*60+$minute<418 then backlog SetChannel 1 1; SetChannel 5 1
addRepeatingEvent 30 -1 if $NTPOn==1&&$CH5==0&&$hour*60+$minute>=1382 then backlog SetChannel 1 1; SetChannel 5 1
```

---

#### 3. How It Works (Command Breakdown)

##### Power & Local NTP Time Synchronization
* `PowerSave 1` — Intended to enable dynamic Wi-Fi power saving to reduce heat and idle power consumption; see the warning above.
* `startDriver NTP` — Starts the built-in NTP client driver.
* `ntp_setServer 192.168.31.2` — Points the device to the local `chrony` NTP server running on the Android host instead of external internet pools. This address is an example and may differ; use the Android host LAN address from step 07.
* `time_setTZ +02:00` — Sets a fixed UTC+2 offset. It does not by itself switch to UTC+3 for Kyiv's summer time. To have the schedule follow Kyiv civil time automatically, configure and test an OpenBeken `time_setDST` rule separately; see the [OpenBeken command reference](https://github.com/openshwprojects/OpenBK7231T_App/blob/main/docs/commands.md) for its syntax. Otherwise, the example remains on UTC+2 year-round.

##### Safe Boot State & Daily Clock Events
* `SetChannel 1 0` — Forces the main relay (`Channel 1`) to **OFF** immediately on boot so the load stays off until accurate time is acquired.
* `SetChannel 5 0` — Resets virtual flag `Channel 5` to `0` (meaning *"initial post-boot time recovery has not run yet"*).
* `addClockEvent 23:02 0xff 1 SetChannel 1 1` — Every day of the week (`0xff`), at **23:02**, executes event ID `1` to turn the relay **ON**.
* `addClockEvent 06:58 0xff 2 SetChannel 1 0` — Every day of the week (`0xff`), at **06:58**, executes event ID `2` to turn the relay **OFF**.

##### Power-Outage Recovery Logic (Boot Catch-Up)
If a power outage occurs and electricity returns at, for example, **01:15** or **14:00**, the scheduled `addClockEvent` triggers (`23:02` or `06:58`) were missed while the plug had no power. The three `addRepeatingEvent` lines solve this:
* Every **30 seconds** (`30`), infinitely (`-1`), the device checks if NTP time is synchronized (`$NTPOn==1`) and whether the initial recovery check has not yet completed (`$CH5==0`).
* Time is evaluated in minutes since midnight (`$hour*60+$minute`):
  * **06:58** $= 6 \times 60 + 58 = \mathbf{418}$ minutes.
  * **23:02** $= 23 \times 60 + 2 = \mathbf{1382}$ minutes.
* **Daytime window (`>= 418` and `< 1382`, i.e., 06:58–23:01):** Ensures `Channel 1` is **OFF** (`0`) and sets `Channel 5` to `1` so the check stops repeating.
* **Nighttime windows (`< 418`, i.e., 00:00–06:57, OR `>= 1382`, i.e., 23:02–23:59):** Ensures `Channel 1` is **ON** (`1`) and sets `Channel 5` to `1`.

---

#### 4. Verification

1. Open the OpenBeken main web UI and confirm the status header displays the accurate local date and time (instead of `1970-01-01`).
2. Navigate to **Tools → Log** (or **Console**) and verify there are no NTP timeout errors and that `Channel 5` switches to `1` within 30 seconds of acquiring NTP time.
3. Optionally run `listEvents` in the OpenBeken console to confirm both clock events (`23:02` and `06:58`) are registered in memory.


