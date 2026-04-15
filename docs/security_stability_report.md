# Sicherheits- und Stabilitätsbericht (C#/Rust Bridge)

## ZUSAMMENFASSUNG DER GEFAHREN (Risk-Level: **Mittel**)

Gesamtbewertung: **Mittel**.

Hauptgründe:
- Keine klaren Malware-Indikatoren wie Shell-Exec (`Process.Start` / `std::process::Command`) im Runtime-Code.
- Netzwerkverkehr ist vorhanden (Relay via WebSocket), aber sichtbar und funktional begründet.
- Relevantes Persistenz-/Integritätsrisiko durch Multiplayer-Save-Handling (temporäres Überschreiben lokaler Saves mit Backup-Mechanik).
- FFI und Threading sind grundsätzlich defensiv umgesetzt, bleiben aber naturgemäß crash-sensitiv bei fehlerhaften nativen Mod-DLLs.

## BEFUND: PERSISTENZ & DATEISYSTEM

### Beobachtungen
- Modloader-Logging in Game-Root:
  - `csharp/DataCenterModLoader/Core.cs` schreibt `dc_modloader_debug.log` in `MelonEnvironment.GameRootDirectory`.
- Mod-Konfiguration und Mod-Assets in UserData:
  - `csharp/DataCenterModLoader/ModConfigSystem.cs` nutzt `MelonEnvironment.UserDataDirectory/ModConfigs`.
  - `csharp/DataCenterModLoader/CustomEmployeeManager.cs` nutzt `MelonEnvironment.UserDataDirectory` für Status/Assets.
- Savegame-Zugriffe im Multiplayer:
  - `csharp/DataCenterModLoader/MultiplayerBridge.cs` nutzt `SaveSystem.saveDirPath` und als Fallback `Application.persistentDataPath`.
  - Host-Join-Sync kann bestehende Save-Datei überschreiben (`WriteSaveToDisk`) und erstellt `.mp_backup`, `_mp_sync`.
  - Cleanup versucht Rücksicherung (`CleanupMpSaveFiles`).
- Rust-Mod `dc_netwatch` schreibt Portrait nach `.../UserData/ModAssets` relativ zu `current_exe`:
  - `crates/dc_netwatch/src/lib.rs`.

### Bewertung
- **Kein Hinweis auf Modifikation von Spiel-Binaries** (`.assets/.dll` des Spiels) im Runtime-Code.
- **Persistenzrisiko vorhanden**: Save-Overwrite-Strategie kann bei Crash/Abbruch zwischen Write und Cleanup zu inkonsistentem lokalen Stand führen.
- `Application.persistentDataPath` kann auf `%AppData%` zeigen (außerhalb des Installationsordners), ist für Unity-Saves aber üblich.

## BEFUND: ENGINE-LOGIK & MEMORY (C#/Rust)

### Harmony / Spiel-Logik
- Viele Patches sind `Postfix`-basiert und event-orientiert (`HarmonyPatches.cs`), was das Risiko harter Logiküberschreibung reduziert.
- Kritischer `Update`-Hook vorhanden:
  - `[HarmonyPatch(typeof(TimeController), "Update")]` als `Postfix`.
- Zwei `Prefix`-Patches können Originalmethoden gezielt unterdrücken (`return false`):
  - `HRSystem.ButtonConfirmHire`
  - `HRSystem.ButtonConfirmFireEmployee`
  - Unterdrückung ist an `CustomEmployeeManager`-Bedingung gebunden.

### FFI / Memory
- C#-Bridge lädt native DLLs via `LoadLibrary/GetProcAddress` (`FFIBridge.cs`).
- String-Marshalling (`Marshal.StringToHGlobalAnsi`) wird freigegeben (`FreeHGlobal`) — korrekt.
- Rust-FFI in `crates/dc_multiplayer/src/ffi/save.rs`:
  - Null-/Längenchecks vorhanden.
  - `copy_nonoverlapping` mit begrenzter Länge (`min(data.len(), max_len)`) reduziert Overflow-Risiko.
- Generell bleibt: Absturzpotenzial besteht bei fehlerhaften externen nativen Mod-DLLs (typisch für FFI-Ökosysteme).

### Threading / Race Conditions
- Rust-Relay nutzt separaten I/O-Thread (`thread::spawn`) + Kanal-Kommunikation (`mpsc`) in `crates/dc_multiplayer/src/net.rs`.
- Shared Multiplayer-State ist über globalen `Mutex` gekapselt (`crates/dc_multiplayer/src/state.rs`).
- Unity-API-Aufrufe finden überwiegend auf C#-Mainthread statt; Rust-I/O-Thread verarbeitet primär Netzwerk.
- **Rest-Risiko**: Potenzielle Timing-/Lock-Contention-Szenarien bei hoher Last, aber keine offensichtliche direkte Unity-API-Nutzung aus Rust-Thread.

## BEFUND: SICHERHEIT (Malware-Check)

### Dateioperationen außerhalb Spielverzeichnis
- Runtime nutzt primär Game-Root/UserData.
- `Application.persistentDataPath` kann systemweit (z. B. `%AppData%`) liegen; entspricht üblicher Unity-Speicherpraxis.
- Keine Hinweise auf Zugriffe auf sensible Systempfade wie `System32`.

### Netzwerk / Exfiltration
- Sichtbarer Relay-Verkehr via WebSocket in `crates/dc_multiplayer/src/net.rs`.
- Default-URL ist hart verdrahtet (`ws://192.99.16.77:9943`) in `crates/dc_multiplayer/src/state.rs`.
- Kein zusätzlicher versteckter Telemetrie-/Webhook-Code in C#/Rust-Runtime gefunden.

### Shell-/Prozessausführung
- Keine Runtime-Treffer für `System.Diagnostics.Process.Start` oder `std::process::Command`.
- Vorhandene Download-/Installlogik in `tools/install.ps1` ist ein separates Setup-Skript, nicht Teil der Ingame-Runtime.

### Sicherheitsbewertung
- Kein direkter Malware-Befund.
- Relevanter Security-Hinweis: Relay nutzt unverschlüsseltes `ws://` statt `wss://` (Man-in-the-Middle-/Manipulationsrisiko im Transit).

## HANDLUNGSEMPFEHLUNG

1. **Save-Sync härten**
   - Atomic Write-Strategie und robustes Rollback für Multiplayer-Save-Overwrite ergänzen.
   - Crash-resistente Marker/Recovery beim nächsten Start einbauen.

2. **Transport absichern**
   - Relay auf **`wss://`** umstellen, Zertifikatsvalidierung sauber erzwingen.
   - Optional Integritätsschutz auf Payload-Ebene (Signatur/MAC).

3. **FFI-Robustheit erhöhen**
   - Klare FFI-Vertragsgrenzen dokumentieren (max Buffergrößen, Lebenszeiten).
   - Defensive Guards/Telemetry für fehlerhafte Drittmods (Rate-Limits, Circuit-Breaker pro Mod).

4. **Threading-Monitoring**
   - Lock-Haltezeiten und Event-Queue-Latenzen messen (Debug-Metriken), um Race/Contention früh zu erkennen.

5. **Persistenz minimieren**
   - Alle mod-spezifischen Artefakte weiterhin strikt in UserData halten.
   - Savegame-Formatkompatibilität ohne Mod removal-safe halten (keine irreversiblen Fremdfelder in Kernsave ohne Fallback).
