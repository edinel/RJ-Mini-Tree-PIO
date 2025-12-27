# NTP Timer Implementation Guide for RJ-Mini-Tree

## Overview

This guide provides implementation suggestions for adding NTP-based time scheduling to automatically turn the Christmas tree LEDs on/off at specified times each day. The system will use WiFi NTP to sync time and check periodically (every ~1000 loops) to stay accurate even after power loss.

---

## Configuration Settings (Pacific Time)

### 1. Add to Top of main.cpp (after existing #defines, around line 22)

```cpp
// === NTP TIME SCHEDULING CONFIGURATION ===
#define ENABLE_TIME_SCHEDULING 1  // Set to 0 to disable scheduling

// NTP Server Configuration
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = -8 * 3600;     // PST = UTC-8 hours
const int   daylightOffset_sec = 3600;     // PDT adds 1 hour (auto-handled)

// Default schedule times (24-hour format)
int scheduleOnHour = 17;    // 5:00 PM
int scheduleOnMinute = 0;
int scheduleOffHour = 23;   // 11:00 PM
int scheduleOffMinute = 0;

// NTP resync counter
#define NTP_RESYNC_INTERVAL 1000  // Resync every 1000 loop iterations
unsigned long loopCounter = 0;
```

---

## Implementation Steps

### STEP 1: Add Preferences Include and Object

**File:** `main.cpp` (top section, after includes around line 9)

```cpp
#include <Preferences.h>  // Built-in ESP32 library - no platformio.ini changes needed

Preferences preferences;
```

**Note:** The `Preferences` library is built into the ESP32 Arduino framework. You don't need to add anything to `platformio.ini` - it's already available.

### STEP 2: Initialize NTP Time in setup()

**File:** `main.cpp` - Modify `setup()` function

Add after `connectToWifi();` call (around line 440):

```cpp
// SET UP WIFI
connectToWifi();  // Like it says
if (debug) { printWifiStatus(); }

// === INITIALIZE NTP TIME ===
#if ENABLE_TIME_SCHEDULING
  Serial.println("Initializing NTP time...");
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);

  // Wait for time to be set (max 10 seconds)
  int ntpTimeout = 0;
  while (time(nullptr) < 100000 && ntpTimeout < 20) {
    Serial.print(".");
    delay(500);
    ntpTimeout++;
  }

  if (time(nullptr) > 100000) {
    Serial.println("\nNTP time synchronized!");
    printLocalTime();  // Function defined below
  } else {
    Serial.println("\nNTP sync timeout - scheduling may not work correctly");
  }

  // Load saved schedule from flash storage
  preferences.begin("tree-schedule", false);
  scheduleOnHour = preferences.getInt("onHour", 17);
  scheduleOnMinute = preferences.getInt("onMin", 0);
  scheduleOffHour = preferences.getInt("offHour", 23);
  scheduleOffMinute = preferences.getInt("offMin", 0);
  preferences.end();

  Serial.printf("Schedule: ON at %02d:%02d, OFF at %02d:%02d\n",
                scheduleOnHour, scheduleOnMinute,
                scheduleOffHour, scheduleOffMinute);
#endif

sleep(3);
gWiFiServer.begin();
```

### STEP 3: Add Helper Functions

**File:** `main.cpp` - Add before `setup()` function (around line 407)

```cpp
// === NTP TIME HELPER FUNCTIONS ===

#if ENABLE_TIME_SCHEDULING

void printLocalTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    Serial.println("Failed to obtain time");
    return;
  }
  Serial.println(&timeinfo, "Current time: %A, %B %d %Y %H:%M:%S");
}

void resyncNTPTime() {
  Serial.println("Resyncing NTP time...");
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  delay(100);  // Brief delay to allow sync to start
  printLocalTime();
}

bool isWithinScheduledTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    Serial.println("Failed to get time - defaulting to OFF");
    return false;  // Default to off if we can't get time
  }

  int currentMinutes = timeinfo.tm_hour * 60 + timeinfo.tm_min;
  int onMinutes = scheduleOnHour * 60 + scheduleOnMinute;
  int offMinutes = scheduleOffHour * 60 + scheduleOffMinute;

  // Handle schedules that cross midnight
  if (offMinutes < onMinutes) {
    // e.g., ON at 20:00 (1200 min), OFF at 02:00 (120 min)
    return (currentMinutes >= onMinutes || currentMinutes < offMinutes);
  } else {
    // Normal case: e.g., ON at 17:00, OFF at 23:00
    return (currentMinutes >= onMinutes && currentMinutes < offMinutes);
  }
}

void enforceSchedule() {
  if (!isWithinScheduledTime()) {
    // Outside schedule window - force off
    if (gButtonClicked != "off") {
      gButtonClicked = "off";
      Serial.println("Auto-OFF: Outside scheduled time");
    }
  }
  // Note: We don't auto-turn ON, only auto-turn OFF
  // User can still manually turn on via web UI
}

#endif  // ENABLE_TIME_SCHEDULING
```

### STEP 4: Modify loop() to Check Schedule

**File:** `main.cpp` - Modify `loop()` function

Add at the very beginning of `loop()` (right after line 450):

```cpp
void loop() {
  // === PERIODIC NTP RESYNC AND SCHEDULE ENFORCEMENT ===
  #if ENABLE_TIME_SCHEDULING
    loopCounter++;
    if (loopCounter >= NTP_RESYNC_INTERVAL) {
      resyncNTPTime();
      loopCounter = 0;
    }
    enforceSchedule();  // Check if we should be off
  #endif

  // == First, handle incoming HTTP requests ==
  WiFiClient client = gWiFiServer.available();
  // ... rest of existing loop code
```

---

### STEP 5: Add Web UI for Schedule Configuration

### Option A: Simple Time Input Fields

**File:** `include/string_constants.h`

Add this HTML after the existing radio buttons in the `index_html` string (requires modifying the HTML template):

```html
<div class="schedule-config">
  <h3>Daily Schedule (Pacific Time)</h3>
  <form action="/schedule" method="GET">
    <label>Turn ON at:
      <input type="time" name="on" value="17:00">
    </label>
    <label>Turn OFF at:
      <input type="time" name="off" value="23:00">
    </label>
    <button type="submit">Save Schedule</button>
  </form>
  <p>Current: ON at <span id="onTime">17:00</span>, OFF at <span id="offTime">23:00</span></p>
</div>
```

### Option B: URL Parameter Only (No UI changes)

Add a new endpoint handler in `loop()` HTTP processing section (around line 474):

```cpp
// Handle schedule update requests
if (gHeader.indexOf("GET /schedule?") >= 0) {
  // Parse on time
  int onIdx = gHeader.indexOf("on=");
  int offIdx = gHeader.indexOf("off=");

  if (onIdx >= 0 && offIdx >= 0) {
    String onTime = gHeader.substring(onIdx + 3, onIdx + 8);  // "HH:MM"
    String offTime = gHeader.substring(offIdx + 4, offIdx + 9);

    scheduleOnHour = onTime.substring(0, 2).toInt();
    scheduleOnMinute = onTime.substring(3, 5).toInt();
    scheduleOffHour = offTime.substring(0, 2).toInt();
    scheduleOffMinute = offTime.substring(3, 5).toInt();

    // Save to flash
    preferences.begin("tree-schedule", false);
    preferences.putInt("onHour", scheduleOnHour);
    preferences.putInt("onMin", scheduleOnMinute);
    preferences.putInt("offHour", scheduleOffHour);
    preferences.putInt("offMin", scheduleOffMinute);
    preferences.end();

    Serial.printf("Schedule updated: ON %02d:%02d, OFF %02d:%02d\n",
                  scheduleOnHour, scheduleOnMinute,
                  scheduleOffHour, scheduleOffMinute);

    client.println("HTTP/1.1 200 OK");
    client.println("Content-type:text/html");
    client.println();
    client.println("<html><body><h2>Schedule Updated!</h2>");
    client.printf("<p>ON: %02d:%02d, OFF: %02d:%02d</p>",
                  scheduleOnHour, scheduleOnMinute,
                  scheduleOffHour, scheduleOffMinute);
    client.println("<a href='/'>Back to main page</a></body></html>");
    break;
  }
}
```

---

## Usage Examples

### Setting Schedule via URL (Option B)

Navigate to:

```text
http://[tree-ip]/schedule?on=17:30&off=22:45
```

This sets:

- Turn ON at 5:30 PM Pacific
- Turn OFF at 10:45 PM Pacific

### Testing Time Function

Add to Serial monitor output or create a `/time` endpoint:

```cpp
if (gHeader.indexOf("GET /time") >= 0) {
  client.println("HTTP/1.1 200 OK");
  client.println("Content-type:text/html");
  client.println();

  struct tm timeinfo;
  getLocalTime(&timeinfo);

  client.println("<html><body><h2>Current Time</h2>");
  client.printf("<p>Time: %02d:%02d:%02d</p>",
                timeinfo.tm_hour, timeinfo.tm_min, timeinfo.tm_sec);
  client.printf("<p>Schedule: ON %02d:%02d, OFF %02d:%02d</p>",
                scheduleOnHour, scheduleOnMinute,
                scheduleOffHour, scheduleOffMinute);
  client.printf("<p>Within schedule: %s</p>",
                isWithinScheduledTime() ? "YES" : "NO");
  client.println("<a href='/'>Back</a></body></html>");
  break;
}
```

---

## Key Design Decisions

1. **Auto-OFF Only**: The system automatically turns OFF outside the schedule window, but doesn't auto-turn ON. Users can still manually activate via web UI anytime.

2. **Persistent Storage**: Schedule times are saved in ESP32 flash using Preferences library, so they survive reboots.

3. **Periodic Resync**: Every 1000 loops (~8 seconds at 120 FPS), NTP resyncs to handle:
   - Power loss recovery
   - Clock drift
   - Daylight saving time changes

4. **Graceful Degradation**: If NTP fails, system defaults to OFF state for safety.

5. **Midnight Crossing Support**: Schedule logic handles cases like "ON at 10 PM, OFF at 2 AM".

6. **Compile-Time Toggle**: `#define ENABLE_TIME_SCHEDULING 0` completely disables scheduling without code changes.

---

## Testing Checklist

- [ ] NTP sync succeeds on WiFi connection
- [ ] `/time` endpoint shows correct Pacific time
- [ ] Schedule persists after reboot
- [ ] LEDs auto-OFF outside window
- [ ] Manual override still works during scheduled time
- [ ] Midnight-crossing schedules work correctly
- [ ] NTP resync happens every ~1000 loops (check Serial output)
- [ ] Power loss recovery syncs time within 10 seconds

---

## Troubleshooting

**NTP Sync Fails:**

- Check WiFi connection
- Try alternate NTP servers: `"time.google.com"`, `"time.cloudflare.com"`
- Increase timeout in setup (change 20 to 40)

**Wrong Timezone:**

- PST = UTC-8 (winter)
- PDT = UTC-7 (summer)
- ESP32 handles DST automatically with daylightOffset_sec parameter

**Schedule Not Working:**

- Check Serial monitor for "Auto-OFF" messages
- Verify schedule with `/time` endpoint
- Ensure `ENABLE_TIME_SCHEDULING` is set to 1

---

## Memory Impact

Estimated additions:

- Code: ~2KB
- RAM: ~200 bytes (global variables)
- Flash storage: 16 bytes (Preferences)

This should fit easily within ESP32 Feather v2 limits.

---

## Future Enhancements (Not Implemented Here)

- Multiple daily time windows
- Day-of-week schedules
- Gradual brightness fade at transitions
- Web UI with live clock display
- Mobile-friendly responsive design
