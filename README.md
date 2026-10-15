# WayPlay

**Play Android games on Linux like a pro.**

Developer: **GREEN TERMINAL** · License: **GPLv3** · Status: pre-alpha (spec stage)

WayPlay is an open-source gaming layer for [Waydroid](https://waydro.id). It adds the features BlueStacks users expect (keymapping, controller support, per-game profiles, performance control) as a **standalone companion app** that runs next to an **unmodified Waydroid**.

---

## 1. Architecture Decision (read first)

**WayPlay is a standalone app, not a fork of Waydroid.**

| | Standalone companion (chosen) | Fork of Waydroid |
|---|---|---|
| Waydroid updates | Free, just install upstream | You must merge upstream forever |
| Maintenance | Small codebase | Large codebase you don't own |
| Install | Works on any machine with Waydroid | Users must replace Waydroid |
| Risk | Limited by Waydroid's public interfaces | Full control |

Fork only if a feature is impossible from outside (most likely: native multi-instance). Until then, talk to Waydroid only through its CLI (`waydroid ...`), `waydroid shell`, properties, cgroups, and Linux input/Wayland interfaces.

```
┌───────────────────────────────────────────┐
│             WayPlay GUI (PySide6)         │
│  Library · Overlay editor · Perf · Settings│
├───────────────────────────────────────────┤
│              WayPlay Core (daemon)        │
│ Input engine · Profiles · Monitor · Macros│
├───────────────────────────────────────────┤
│        Backend abstraction (Backend ABC)  │
│    WaydroidBackend   │   RedroidBackend   │
└───────────────────────────────────────────┘
```

All Android-specific calls go through `Backend`. Never call `waydroid` from outside `backends/`.

---

## 2. Tech Stack

- Python 3.11+, PySide6 (Qt 6), `python-evdev`, `psutil`, `pydantic` (profile validation)
- Packaging: `pyproject.toml`, later AppImage/Flatpak
- Tests: `pytest`; lint: `ruff`; types: `mypy`
- Performance-critical paths may move to Rust later. Do not start there.

## 3. Repo Structure

```
wayplay/
├── pyproject.toml
├── README.md
├── AGENTS.md                 # rules for AI coding agents (section 4)
├── src/wayplay/
│   ├── core/
│   │   ├── daemon.py         # main loop, IPC
│   │   ├── input_engine.py   # evdev read -> action translation
│   │   ├── touch_device.py   # uinput virtual touchscreen
│   │   ├── profiles.py       # load/save/validate JSON
│   │   ├── perf.py           # cgroups, governor, props
│   │   └── monitor.py        # CPU/RAM/FPS/temps
│   ├── backends/
│   │   ├── base.py           # Backend ABC
│   │   ├── waydroid.py
│   │   └── redroid.py        # v2.0
│   ├── gui/
│   │   ├── main_window.py
│   │   ├── overlay.py        # transparent keymap editor
│   │   └── perf_panel.py
│   └── cli.py                # `wayplay launch <pkg>`
├── profiles/                 # bundled game profiles (*.json)
├── tests/
└── docs/
```

---

## 4. Rules for AI Coding Agents (copy into AGENTS.md)

1. Work on **one milestone task at a time**. Do not implement features from later milestones.
2. Every task has **acceptance criteria**. Do not mark done until each is met and a test or manual check exists.
3. Keep modules small and typed. Public functions get docstrings and type hints.
4. All Waydroid/Android interaction goes through `backends/base.py`. No direct `subprocess` calls to `waydroid` elsewhere.
5. Never hardcode screen resolution. Read it from the backend.
6. Anything touching `/dev/uinput` or `/dev/input` must fail gracefully with a clear error message about permissions (udev rule / `input` group).
7. Do not add dependencies without stating why in the PR/commit message.
8. If an assumption about Waydroid behavior is unverified, write a small **spike script** in `tools/spikes/` and report the result instead of guessing.
9. Do not claim anti-cheat bypass, ban evasion, or integrity-check spoofing. Document limits honestly.

### Prompting pattern that works

> "Read README.md and AGENTS.md. Implement **Task M1.2** only. Show the file list you'll change first. Write tests. Stop when acceptance criteria are met."

---

## 5. Milestones and Tasks

### M0: Feasibility spikes (week 1)

- **M0.1** Install Waydroid, run one test game. *Done when:* game is playable with mouse-as-touch.
- **M0.2 (critical spike)** Verify how to inject touch into Waydroid. Test, in order:
  1. Host `uinput` virtual touchscreen, seen by the Wayland compositor and delivered to the Waydroid window.
  2. Injection inside the container (`waydroid shell` + `/dev/uinput` or `sendevent`).
  3. `adb shell input` (known to be too slow; baseline only).

  *Done when:* a script taps a known screen coordinate and you have written down which method works, its latency, and compositor behavior (GNOME/KDE/Sway). **The whole input design depends on this result.**
- **M0.3** Init repo, GPLv3 license, `pyproject.toml`, CI with ruff + pytest.

### M1: Input engine (weeks 2-3)

- **M1.1** `touch_device.py`: virtual multitouch device (starter code below). *Done when:* `down/move/up` for 2+ simultaneous slots works.
- **M1.2** `input_engine.py`: grab keyboard/mouse via evdev, translate to actions. *Done when:* a key press produces a tap at configured coordinates.
- **M1.3** Virtual joystick: WASD maps to a stick region with smooth center-to-edge movement. *Done when:* holding W+D moves diagonally.
- **M1.4** Mouse-look: relative motion becomes camera-drag on a region, with pointer lock hotkey. *Done when:* camera rotates smoothly with sensitivity setting.
- **M1.5** Latency measurement tool. *Done when:* reports input-to-injection time; target under 20 ms.

### M2: Profiles (week 4)

- **M2.1** Pydantic model for profile JSON (schema in section 6).
- **M2.2** Load/save from `~/.config/wayplay/profiles/`.
- **M2.3** Auto-switch by foreground package name from the backend.

### M3: Overlay editor (weeks 5-6)

- **M3.1** Transparent always-on-top Qt window aligned to the Waydroid window.
- **M3.2** Draggable widgets: tap, joystick, swipe, aim. "Press a key to bind" flow.
- **M3.3** Show/hide hotkey. Saves directly into the active profile.
- **Note:** overlay and pointer-lock behavior differs per Wayland compositor. Test on GNOME and KDE before polishing.

### M4: Controllers (week 7)

- **M4.1** Detect gamepads via evdev, hotplug aware.
- **M4.2** Passthrough mode (native Android gamepad support).
- **M4.3** Mapping mode: stick to joystick region, buttons to tap points, dead zones.

### M5: Performance panel (weeks 8-9)

- **M5.1** Resolution/DPI through backend properties.
- **M5.2** CPU/RAM limits via cgroups on the Waydroid container.
- **M5.3** CPU governor and GPU selection (`DRI_PRIME`); optional `gamemoderun`.
- **M5.4** Presets: Performance / Balanced / Battery.
- **M5.5** Live monitor: CPU, RAM, GPU, temps, FPS where available.

### M6: Release v0.1 (weeks 10-12)

- Setup wizard (Waydroid, GApps, ARM translation via `waydroid_script`)
- Packaging, docs, sample profiles, test on Ubuntu / Arch / Fedora and Intel / AMD / NVIDIA

### Later (post v0.1)

Game library, screenshot/recording, macros, clipboard/file sync, community profiles, plugins, multi-instance via `RedroidBackend`, gyro emulation, phone-as-controller.

---

## 6. Profile Format (draft)

```json
{
  "version": 1,
  "package": "com.example.game",
  "name": "Example Game",
  "screen": { "width": 1920, "height": 1080 },
  "bindings": [
    { "type": "tap", "key": "KEY_SPACE", "x": 1700, "y": 800 },
    { "type": "joystick", "keys": ["KEY_W", "KEY_A", "KEY_S", "KEY_D"],
      "cx": 300, "cy": 800, "radius": 150 },
    { "type": "aim", "region": { "x": 960, "y": 0, "w": 960, "h": 1080 },
      "sensitivity": 1.2, "toggle_key": "KEY_LEFTALT" }
  ],
  "performance": { "preset": "performance" }
}
```

Coordinates are stored in the profile's reference resolution and scaled to the live screen at runtime.

---

## 7. Starter Code: Virtual Touch Device (M1.1)

`src/wayplay/core/touch_device.py`

```python
"""Virtual multitouch screen built on uinput."""
from evdev import UInput, AbsInfo, ecodes as e


class TouchDevice:
    MAX_SLOTS = 10

    def __init__(self, width: int, height: int) -> None:
        self.width, self.height = width, height
        caps = {
            e.EV_KEY: [e.BTN_TOUCH],
            e.EV_ABS: [
                (e.ABS_MT_SLOT, AbsInfo(0, 0, self.MAX_SLOTS - 1, 0, 0, 0)),
                (e.ABS_MT_TRACKING_ID, AbsInfo(0, 0, 65535, 0, 0, 0)),
                (e.ABS_MT_POSITION_X, AbsInfo(0, 0, width, 0, 0, 0)),
                (e.ABS_MT_POSITION_Y, AbsInfo(0, 0, height, 0, 0, 0)),
                (e.ABS_X, AbsInfo(0, 0, width, 0, 0, 0)),
                (e.ABS_Y, AbsInfo(0, 0, height, 0, 0, 0)),
            ],
        }
        try:
            self.ui = UInput(caps, name="wayplay-touch",
                             input_props=[e.INPUT_PROP_DIRECT])
        except PermissionError as exc:
            raise SystemExit(
                "Cannot open /dev/uinput. Add your user to the 'input' group "
                "or install the WayPlay udev rule, then re-login."
            ) from exc
        self._active: set[int] = set()
        self._next_id = 1

    def down(self, slot: int, x: int, y: int) -> None:
        self.ui.write(e.EV_ABS, e.ABS_MT_SLOT, slot)
        self.ui.write(e.EV_ABS, e.ABS_MT_TRACKING_ID, self._next_id)
        self._next_id = (self._next_id % 65534) + 1
        self.ui.write(e.EV_ABS, e.ABS_MT_POSITION_X, x)
        self.ui.write(e.EV_ABS, e.ABS_MT_POSITION_Y, y)
        if not self._active:
            self.ui.write(e.EV_KEY, e.BTN_TOUCH, 1)
        self._active.add(slot)
        self.ui.syn()

    def move(self, slot: int, x: int, y: int) -> None:
        if slot not in self._active:
            return
        self.ui.write(e.EV_ABS, e.ABS_MT_SLOT, slot)
        self.ui.write(e.EV_ABS, e.ABS_MT_POSITION_X, x)
        self.ui.write(e.EV_ABS, e.ABS_MT_POSITION_Y, y)
        self.ui.syn()

    def up(self, slot: int) -> None:
        if slot not in self._active:
            return
        self.ui.write(e.EV_ABS, e.ABS_MT_SLOT, slot)
        self.ui.write(e.EV_ABS, e.ABS_MT_TRACKING_ID, -1)
        self._active.discard(slot)
        if not self._active:
            self.ui.write(e.EV_KEY, e.BTN_TOUCH, 0)
        self.ui.syn()

    def close(self) -> None:
        self.ui.close()
```

## 8. Starter Code: Minimal Keymapper (M1.2 + M1.3)

`tools/spikes/keymap_demo.py`

```python
"""Demo: SPACE taps a point, WASD drives a virtual joystick."""
import sys
from evdev import InputDevice, ecodes as e, list_devices
from wayplay.core.touch_device import TouchDevice

W, H = 1920, 1080
TAP = {e.KEY_SPACE: (1700, 800)}
JOY_CENTER, JOY_RADIUS = (300, 800), 150
JOY_KEYS = {e.KEY_W: (0, -1), e.KEY_S: (0, 1), e.KEY_A: (-1, 0), e.KEY_D: (1, 0)}
SLOT_JOY, SLOT_TAP = 0, 1


def find_keyboard() -> InputDevice:
    for path in list_devices():
        dev = InputDevice(path)
        if e.EV_KEY in dev.capabilities() and e.KEY_A in dev.capabilities()[e.EV_KEY]:
            return dev
    sys.exit("No keyboard found")


def main() -> None:
    touch = TouchDevice(W, H)
    kb = find_keyboard()
    kb.grab()  # exclusive: keys no longer reach the desktop. Add a hotkey to release!
    held: set[int] = set()
    joy_on = False
    try:
        for ev in kb.read_loop():
            if ev.type != e.EV_KEY or ev.value == 2:  # ignore autorepeat
                continue
            if ev.code in TAP:
                x, y = TAP[ev.code]
                touch.down(SLOT_TAP, x, y) if ev.value else touch.up(SLOT_TAP)
            elif ev.code in JOY_KEYS:
                held.add(ev.code) if ev.value else held.discard(ev.code)
                dx = sum(JOY_KEYS[k][0] for k in held)
                dy = sum(JOY_KEYS[k][1] for k in held)
                if dx or dy:
                    norm = max(abs(dx), abs(dy)) and (dx * dx + dy * dy) ** 0.5
                    x = int(JOY_CENTER[0] + dx / norm * JOY_RADIUS)
                    y = int(JOY_CENTER[1] + dy / norm * JOY_RADIUS)
                    if not joy_on:
                        touch.down(SLOT_JOY, *JOY_CENTER)
                        joy_on = True
                    touch.move(SLOT_JOY, x, y)
                elif joy_on:
                    touch.up(SLOT_JOY)
                    joy_on = False
            elif ev.code == e.KEY_F12 and ev.value:  # panic exit
                break
    finally:
        kb.ungrab()
        touch.close()


if __name__ == "__main__":
    main()
```

Run with: `sudo python tools/spikes/keymap_demo.py` (until the udev rule exists). **Result of spike M0.2 decides whether this host-side approach is final.**

---

## 9. Backend Interface (stub)

```python
# src/wayplay/backends/base.py
from abc import ABC, abstractmethod


class Backend(ABC):
    @abstractmethod
    def is_running(self) -> bool: ...
    @abstractmethod
    def start(self) -> None: ...
    @abstractmethod
    def stop(self) -> None: ...
    @abstractmethod
    def screen_size(self) -> tuple[int, int]: ...
    @abstractmethod
    def foreground_package(self) -> str | None: ...
    @abstractmethod
    def launch(self, package: str) -> None: ...
    @abstractmethod
    def set_prop(self, key: str, value: str) -> None: ...
    @abstractmethod
    def set_density(self, dpi: int) -> None: ...
```

---

## 10. Known Risks

- **Anti-cheat:** many competitive games detect emulators and may block or ban. WayPlay does not bypass this. Keep an honest compatibility list.
- **Wayland differences:** overlay positioning and pointer lock vary across GNOME, KDE, Sway, Hyprland.
- **NVIDIA:** Waydroid support is weaker; expect extra work.
- **Permissions:** uinput/evdev need a udev rule; ship it in the installer.
- **EULA:** do not reverse-engineer or copy BlueStacks code. WayPlay is built from open components only.

## 11. Contributing

PRs welcome after v0.1. Follow `AGENTS.md`, include tests, and keep Waydroid access inside `backends/`.

## 12. License

GPLv3. See `LICENSE`.
