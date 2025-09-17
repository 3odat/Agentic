Here’s the tight, no-fluff summary you asked for—what each file is, its API/commands, and how they plug together.

# Files at a glance

## 1) `internal_latest.py` — **Sensors Service**

* **Purpose:** Pulls PX4 telemetry via MAVSDK and exposes a live JSON snapshot for other modules.
* **Run:**
  `python internal_latest.py --url udp://:14540 --hz 1.0 --host 0.0.0.0 --port 8001 --json mavsdk_sensor_snapshot.json`
* **API (HTTP, port 8001):**

  * `GET /sensors` → current drone state:

    * `timestamp`
    * `battery {voltage_v, remaining, …}`
    * `gps {num_satellites, fix_type}`
    * `position {lat_deg, lon_deg, abs_alt_m, rel_alt_m}`
    * `velocity_ned {north_m_s, east_m_s, down_m_s}`
    * `attitude {quaternion, euler_deg{yaw_deg,…}}`
    * `status {armed, in_air, flight_mode}`
    * `heading_deg`, `wind`
* **Meaning:** This is the **single source of truth** for pose/attitude/health that everything else consumes.

---

## 2) `soso.py` — **Vision Service (Detections + Geo-projection)**

* **Purpose:** Detect objects from the camera, keep a short rolling window, and **geo-locate** detections by fusing with `/sensors` (GPS + yaw).
* **Run:**
  `python soso.py --port 8088`  *(expects the Sensors Service to be up on :8001)*
* **API (HTTP, port 8088):**

  * `GET /scene` → **most recent \~5s** snapshot:

    * `global {timestamp, gps{lat,lon,alt}, yaw_deg, window_sec}`
    * `count`, `summary_by_object`
    * `detections[]` with:

      * `Object Name`
      * `Center {x_m, y_m, z_m}` *(camera frame: x right, y up, z forward)*
      * `Confidence`
      * `estimated_global {lat, lon, alt}` *(computed using yaw + GPS + Center)*
  * `GET /history` → previous \~5s slice (making a \~10s rolling window).
  * `GET /take_photo` → saves the latest frame; returns `{status, file}`.
* **Meaning:** Turns raw detections into **actionable world GPS targets** for the planner/controller, plus offers quick evidence capture.

---

## 3) `Agentic_Controller_v4.py` — **MAVSDK Controller (REPL)**

* **Purpose:** High-level flight skills with clear logs (connect, takeoff, goto, orbit, look, land…).
* **Run:**
  `python Agentic_Controller_v4.py`
* **Commands (typed in the REPL):**

  * `status`, `arm`, `takeoff [alt_m]`, `land`, `rtl`, `stop`
  * Body-frame moves: `forward/backward/left/right/up/down <meters>`
  * Yaw/gimbal: `yaw_left/right <deg>`, `look_down <deg>`, `look_forward`
  * **Nav:** `goto <lat> <lon> [abs_alt_m]`
  * **Inspection:** `orbit <radius_m> [cw|ccw] [speed_mps] [return]` *(true 360°, snap-back to start perimeter, optional return to center)*
* **Meaning:** Deterministic motion primitives your agent can chain after choosing a target from the Vision Service.

---

# How they connect (data flow)

```
PX4 ──MAVLink──> MAVSDK
   └────────────> internal_latest.py (Sensors Service, :8001)
                      │   exposes GET /sensors (pose/health/yaw)
                      ▼
                 soso.py (Vision, :8088)
                      ├─ polls /sensors for GPS + yaw
                      ├─ detects objects, projects Center{x,y,z} -> estimated_global{lat,lon,alt}
                      └─ serves /scene, /history, /take_photo
                      ▼
               Planner / You
                      ├─ read /scene or /history
                      ├─ pick target’s estimated_global
                      └─ drive Agentic_Controller_v4.py
                              ├─ goto <lat lon alt>
                              └─ orbit / look / land / photo
```

# In one line each

* **Sensors:** “What’s my drone state right now?” → `/sensors`.
* **Vision:** “What objects are around me, and where are they on the map?” → `/scene` & `/history` (with `estimated_global`).
* **Controller:** “Go there, circle it, take a look, and come back.” → `goto`, `orbit`, `look_*`, `rtl`.

If you want this even shorter (for a repo header), say:
**Sensors (8001) → Vision (8088) → Controller (REPL).**
Vision depends on Sensors; Controller uses Vision outputs to decide where to fly.
