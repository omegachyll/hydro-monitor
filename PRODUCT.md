# hydro-monitor

## Problem

The owner runs a personal hydroponic setup at home: a planter kit with water channels, a Raspberry Pi they already own, and Atlas Scientific sensors. Today nothing monitors the nutrient solution automatically. Checking pH, nutrient strength (EC) and water temperature means manual spot checks, and there is no warning when something drifts or the reservoir runs low.

The owner already runs Home Assistant and wants the hydroponic readings to live there, next to the rest of their home, instead of in a separate app.

## Users

- **The owner**, a single home grower. They are comfortable with a Raspberry Pi and Home Assistant. They want to glance at a dashboard, look at history, get a phone alert when something needs attention, and calibrate probes without typing commands into a terminal.
- Because the repository is public, other hobbyists with similar Atlas Scientific hardware may reuse the code and the Home Assistant alert blueprint. The project is not designed or supported for them specifically.

## v1 scope

1. **Continuous readings.** About once a minute the Raspberry Pi reads pH, conductivity (EC), water temperature and the reservoir float switch.
2. **Automatic appearance in Home Assistant.** After the service is installed on the Pi, the four readings show up in Home Assistant on their own, grouped as one device. No manual Home Assistant configuration is needed to see them.
3. **Dashboard and history in Home Assistant.** Home Assistant provides the dashboard and keeps the history. The Pi has no web page of its own.
4. **Calibration from Home Assistant.** Home Assistant shows buttons for each calibration step:
   - pH: mid (7.00), low (4.00), high (10.00), and clear calibration.
   - EC: dry, low, high, and clear calibration.
   - A "Last calibrated" value for each probe so the owner knows when calibration was last done.
5. **Alerts.** The repository includes a ready-made Home Assistant automation blueprint. The owner chooses acceptable ranges for pH, EC and temperature, and a notification target such as the Home Assistant phone app. They get a notification when a reading goes out of range or the reservoir water is low.
6. **Runs unattended.** The service starts by itself when the Pi boots and recovers by itself if the Pi restarts or the MQTT broker goes away for a while.

## Non-goals

- **No automatic control.** No dosing pumps, relays or pump switching in v1. The owner keeps adjusting the solution by hand. The design should leave room to add dosing later.
- **No standalone dashboard on the Pi.** Home Assistant is the only interface.
- **One reservoir, one set of probes.** No support for multiple systems.
- **No user accounts on the Pi.** Access control is whatever Home Assistant and the MQTT broker already provide.
- **No long-term storage on the Pi.** History lives in Home Assistant.

## Success criteria

v1 is done when:

1. After installing the service, the pH, EC, temperature and water-level sensors appear in Home Assistant automatically, update about every minute, and come back on their own after a Pi reboot or an MQTT broker outage.
2. pH (mid, low, high) and EC (dry, low, high) can be calibrated from Home Assistant buttons, and the "Last calibrated" value updates afterwards.
3. The blueprint sends a phone notification when a reading goes out of range or the water level is low.
4. The system runs unattended for about a week with no missed readings and no crashes.

## Open questions

- **Temperature compensation.** Should the water temperature reading be used to correct the pH and EC readings? This is common practice, but the owner hasn't decided. v1 does not include it unless confirmed.
- **Float switch wiring.** Which Raspberry Pi pin the float switch connects to, and whether "closed" means water is low or water is OK. v1 makes both configurable.
- **MQTT security.** Whether the broker requires a username and password, or TLS.
- **Read interval and smoothing.** Is about 60 seconds right? Should readings be averaged or smoothed before they're sent?
- **Flow Connector Board.** How it fits alongside the i4 InterLink shield in the final wiring.
