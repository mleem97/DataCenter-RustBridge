# Data Center Modloader

A Rust-based modloader for the Unity game **Data Center** by Waseku.

Built on **MelonLoader** with a **Rust FFI bridge** — write mods in Rust with safe wrappers!

## Quick Start

### 1. Install MelonLoader

Download the MelonLoader installer from [GitHub](https://github.com/LavaGang/MelonLoader/releases/tag/v0.7.2) and install it for `Data Center.exe`.

### 2. Build the Modloader

```
cargo build --release
dotnet build -c Release csharp/DataCenterModLoader/
```

### 3. Deploy

```powershell
.\tools\install.ps1
```

### 4. Launch the Game

Start Data Center through Steam. You should see the MelonLoader console with the modloader banner.

## Writing a Rust Mod

Create a new Rust library crate with `crate-type = ["cdylib"]` and depend on `dc_api`:

```rust
use dc_api::*;

static mut API: Option<Api> = None;

#[no_mangle]
pub extern "C" fn mod_info() -> ModInfo {
    ModInfo::new("my_mod", "My Mod", "1.0.0", "YourName", "Does cool stuff")
}

#[no_mangle]
pub extern "C" fn mod_init(api: &'static GameAPI) -> bool {
    let api = unsafe { Api::from_raw(api) };
    api.log_info("Hello from Rust!");
    unsafe { API = Some(api); }
    true
}

#[no_mangle]
pub extern "C" fn mod_update(dt: f32) {
    let api = unsafe { API.as_ref().unwrap() };
    // Your per-frame logic here
}
```

Build it with `cargo build --release` and copy the `.dll` to the game's `Mods/native/` folder.

### Handling Game Events

Mods can optionally export `mod_on_event` to react to game events dispatched via Harmony hooks:

```rust
use dc_api::events::{self, Event};

#[no_mangle]
pub extern "C" fn mod_on_event(event_id: u32, event_data: *const u8, data_size: u32) {
    let Some(event) = events::decode(event_id, event_data, data_size) else { return };
    let api = unsafe { API.as_ref().unwrap() };

    match event {
        Event::MoneyChanged { old_value, new_value, delta } => {
            api.log_info(&format!("Money: ${:.2} -> ${:.2}", old_value, new_value));
        }
        Event::ServerBroken => {
            api.log_warning("A server broke down!");
        }
        Event::DayEnded { day } => {
            api.log_info(&format!("Day {} started", day));
        }
        _ => {}
    }
}
```

#### Available Events

| ID  | Event               | Data                          | Trigger                              |
|-----|---------------------|-------------------------------|--------------------------------------|
| 100 | `MoneyChanged`      | `old_value, new_value, delta` | `Player.UpdateCoin`                  |
| 101 | `XPChanged`         | `old_value, new_value, delta` | `Player.UpdateXP`                    |
| 102 | `ReputationChanged` | `old_value, new_value, delta` | `Player.UpdateReputation`            |
| 200 | `ServerPowered`     | `powered_on: bool`            | `Server.PowerButton`                 |
| 201 | `ServerBroken`      | —                             | `Server.ItIsBroken`                  |
| 202 | `ServerRepaired`    | —                             | `Server.RepairDevice`                |
| 203 | `ServerInstalled`   | —                             | `Server.ServerInsertedInRack`        |
| 300 | `DayEnded`          | `day: u32`                    | Day counter change in TimeController |
| 400 | `CustomerAccepted`  | `customer_id: i32`            | `MainGameManager.ButtonCustomerChosen` |
| 500 | `ShopCheckout`      | —                             | `ComputerShop.ButtonCheckOut`        |
| 600 | `EmployeeHired`     | —                             | `HRSystem.ButtonConfirmHire`         |
| 601 | `EmployeeFired`     | —                             | `HRSystem.ButtonConfirmFireEmployee` |
| 700 | `GameSaved`         | —                             | `SaveSystem.SaveGame`                |
| 701 | `GameLoaded`        | —                             | `SaveSystem.Load`                    |

## Mod Config System

The modloader includes a **runtime configuration system** that lets mods register settings (bools, ints, floats) editable through an in-game IMGUI panel.

- Press **F8** to toggle the Mod Settings panel
- Or click **Settings** in the main menu and choose **"Mod Settings"**
- All values are persisted to `UserData/ModConfigs/<modId>.json`

### For C# MelonLoader Mods

```csharp
using DataCenterModLoader;

// Set display info (shown in the panel header)
ModConfigSystem.SetModInfo("my_mod", "YourName", "1.0.0");

// Register options
ModConfigSystem.RegisterBoolOption("my_mod", "god_mode", "God Mode", false, "Prevents all damage");
ModConfigSystem.RegisterIntOption("my_mod", "speed", "Move Speed", 5, 1, 20, "Speed multiplier");
ModConfigSystem.RegisterFloatOption("my_mod", "gravity", "Gravity", 1.0f, 0.0f, 3.0f);

// Read values at any time
bool godMode = ModConfigSystem.GetBoolValue("my_mod", "god_mode");
int speed    = ModConfigSystem.GetIntValue("my_mod", "speed", 5);
```

### For Rust Native Mods

Rust mods register config entries through the FFI bridge automatically via the `dc_api` config functions. See `crates/dc_api` for details.

> **Full API reference:** [docs/mod_config_system.md](docs/mod_config_system.md)

## Architecture

```
Game (Data Center — Unity 6, IL2CPP)
  └── MelonLoader v0.7.2
        └── RustBridge.dll (C# MelonMod)
              ├── Harmony patches on game methods → Event System
              ├── GameAPI function pointer table  → Polling API (v1/v2)
              └── FFI Bridge
                    └── Mods/native/*.dll (Rust cdylib mods)
                          ├── mod_init(api)       ← receives GameAPI table
                          ├── mod_update(dt)      ← called every frame
                          ├── mod_on_event(id,..) ← receives Harmony events
                          └── mod_shutdown()      ← called on quit
```

## Project Structure

```
├── csharp/DataCenterModLoader/
│   ├── Core.cs                   MelonMod entry point + Harmony registration
│   ├── FFIBridge.cs              Loads Rust DLLs, resolves exports, dispatches
│   ├── GameAPI.cs                Function pointer table (polling API v1/v2)
│   ├── GameHooks.cs              Safe Il2Cpp accessor wrappers
│   ├── EventSystem.cs            Event IDs, data structs, EventDispatcher
│   └── HarmonyPatches.cs         Harmony patches → fires events to Rust mods
├── crates/dc_api/
│   ├── src/lib.rs                GameAPI struct, Api wrapper, FFI type aliases
│   └── src/events.rs             Event types, data structs, decode() function
├── crates/dc_example_mod/        Example Rust mod (infinite money + events)
├── tools/dump/                   Cpp2IL type dump
├── tools/install.ps1             Build & deploy to game directory
├── tools/dump_types.ps1          Run Cpp2IL type dumper
└── tools/find_type.ps1           Search Il2Cpp types by name
```

## Game Info

| Property | Value |
|----------|-------|
| Engine | Unity 6000.3.12f1 (IL2CPP) |
| Developer | Waseku |
| Install Path | `C:\Program Files (x86)\Steam\steamapps\common\Data Center` |

## Documentation

| Document | Description |
|----------|-------------|
| [Mod Config System](docs/mod_config_system.md) | Developer guide for registering and using mod settings from C# mods |
| [Game API Reference](docs/game_api_reference.md) | Il2Cpp API reference for game classes (Server, Technician, PlayerManager, etc.) |
| [Multiplayer Roadmap](docs/multiplayer_roadmap.md) | Design doc and roadmap for the multiplayer mod |
| [Security & Stability Report](docs/security_stability_report.md) | Security and stability analysis of the C#/Rust bridge and runtime behavior |

## License

MIT
