# Cloud Control & Climate Entity — Architektur-Konzept

## Hintergrund

Die BauerGroup Hargassner Integration deckt die **lokale Telemetrie** ab (228+ Sensoren via Telnet). Die Steuerung (Heizmodus, Temperaturen, Heizkurve) ist nur über die **Hargassner Connect Cloud API** möglich, die von Ronald Knirzinger reverse-engineered wurde ([hargassner-ha](https://github.com/knirzinger/hargassner-ha)).

Dieses Dokument beschreibt die Architektur für eine eigenständige Cloud-Control-Integration mit nativer Home Assistant `ClimateEntity`, die als **Companion** zur bestehenden BauerGroup-Integration arbeitet.

## Entscheidung: Separate Integration

| Variante | Pro | Contra |
| --- | --- | --- |
| **Merge in eine Integration** | Ein Config Flow, ein Device | Völlig verschiedene APIs (Telnet vs. HTTPS), erhöhte Komplexität, Internet-Abhängigkeit für lokale Sensoren bei Fehlern |
| **Separate Companion-Integration** | Klare Trennung (Read vs. Write), unabhängig deploybar, unabhängig wartbar, Cloud-Ausfall betrifft nicht lokale Sensoren | Zwei Integrationen installieren, kein gemeinsames Device |

**Empfehlung:** Separate Integration. Die APIs haben nichts gemeinsam — lokales Telnet-Parsing und Cloud-REST-OAuth sind völlig verschiedene Domänen.

## Cloud API — Reverse-Engineered Endpunkte

Basis: `https://web.hargassner.at`

### Authentifizierung

OAuth 2.0 Resource Owner Password Credentials (ROPC):

1. `GET /js/app.js` — OAuth `client_id` und `client_secret` per Regex aus dem Portal-JavaScript extrahieren
2. `POST /oauth/token` — Bearer Token anfordern (TTL: 30 Minuten)
3. Automatische Token-Erneuerung und 401-Retry mit Secret-Re-Extraction

> **Hinweis:** Die OAuth-Credentials sind nicht statisch — sie werden bei jedem Start dynamisch aus dem JavaScript-Bundle extrahiert. Das macht die Integration self-healing wenn Hargassner Credentials rotiert, aber fragil wenn das JS-Bundle-Format sich ändert.

### REST-Endpunkte

| Methode | Endpunkt | Funktion |
| --- | --- | --- |
| GET | `/api/installations/{id}/widgets` | Alle Einstellungen lesen (hierarchisch) |
| PATCH | `/api/installations/{id}/widgets/heating-circuits/{hc}/parameters/{param}` | Heizkreis-Parameter setzen |
| PATCH | `/api/installations/{id}/widgets/heater/parameters/fuel-stock` | Pelletvorrat aktualisieren |
| PATCH | `/api/installations/{id}/widgets/buffer/default/parameters/solar-mode-active` | Solar-Modus umschalten |
| POST | `/api/installations/{id}/widgets/boilers/{b}/actions/force-charging` | Warmwasser-Sofortladung |

### Widget-Struktur (GET Response)

```
installations[]
  └─ widgets
       ├─ heating-circuits[]
       │    ├─ parameters: room-temp-heating, room-temp-reduction,
       │    │              room-temp-correction, heating-curve-steepness,
       │    │              heating-off-temp, day-setback-off-temp,
       │    │              night-setback-off-temp
       │    └─ mode: {automatic, heating, reduction, off}
       ├─ boilers[]
       │    └─ actions: force-charging
       ├─ buffer
       │    └─ parameters: solar-mode-active
       ├─ heater
       │    └─ parameters: fuel-stock
       └─ hot-water
            └─ parameters: bathroom-heating-active
```

## ClimateEntity — Mapping

Die Hargassner Cloud-API lässt sich sauber auf eine HA `ClimateEntity` abbilden:

```python
class HargassnerClimate(ClimateEntity):
    """Hargassner Heizkreis als Climate Entity."""

    # --- Temperatur ---
    @property
    def current_temperature(self) -> float:
        # Option A: Aus Cloud-Widget (weniger aktuell, 15min Polling)
        # Option B: Aus BauerGroup-Integration (Echtzeit, lokal)
        #           → Erfordert Cross-Integration State-Lookup
        ...

    @property
    def target_temperature(self) -> float:
        # Cloud: heating-circuits/{hc}/parameters/room-temp-heating
        # bzw. room-temp-reduction je nach Modus
        ...

    # --- HVAC Mode ---
    @property
    def hvac_mode(self) -> HVACMode:
        # Cloud: heating-circuits/{hc}/mode
        # Mapping:
        #   "automatic" → HVACMode.AUTO
        #   "heating"   → HVACMode.HEAT
        #   "reduction" → HVACMode.IDLE  (oder PRESET)
        #   "off"       → HVACMode.OFF
        ...

    # --- Preset Mode ---
    @property
    def preset_mode(self) -> str:
        # "party", "holiday", "bathroom" etc.
        ...

    # --- Heizkurve (als Extra-Attribut) ---
    @property
    def extra_state_attributes(self) -> dict:
        return {
            "heating_curve_steepness": ...,
            "heating_off_temperature": ...,
        }
```

### Zusätzliche Entities (neben ClimateEntity)

| Entity | Typ | API-Parameter |
| --- | --- | --- |
| Heizkurven-Steilheit | `NumberEntity` | `heating-curve-steepness` (0.2–3.5) |
| Heizung-Aus-Temperatur | `NumberEntity` | `heating-off-temp` (-10 bis +30°C) |
| Solar-Modus | `SwitchEntity` | `solar-mode-active` |
| Badezimmer-Heizung | `SwitchEntity` | `bathroom-heating-active` |
| Pelletvorrat | `NumberEntity` | `fuel-stock` (0–5000 kg) |
| Warmwasser Sofortladung | `ButtonEntity` | `force-charging` |

## Architektur — Companion-Modell

```
┌─────────────────────────────────┐     ┌─────────────────────────────────┐
│  BauerGroup Integration         │     │  Cloud Control Integration      │
│  (bestehend)                    │     │  (neu)                          │
│                                 │     │                                 │
│  Telnet Client ──► 228 Sensors  │     │  OAuth Client ──► ClimateEntity │
│       ▲                         │     │       ▲           NumberEntity  │
│       │                         │     │       │           SwitchEntity  │
│   Lokales LAN                   │     │   HTTPS/Cloud     ButtonEntity  │
│   (kein Internet nötig)         │     │   (Internet nötig)              │
│                                 │     │                                 │
│  Kessel ◄────── Telnet ─────►   │     │  web.hargassner.at ◄── REST ►  │
└─────────────────────────────────┘     └─────────────────────────────────┘
              │                                       │
              └──────── Gleicher Kessel ──────────────┘
                   (gleiche Daten, verschiedene Wege)
```

### Cross-Integration: Ist-Temperatur

Für `current_temperature` der ClimateEntity gibt es zwei Optionen:

**Option A — Eigenständig (empfohlen):**
Die Cloud-API liefert über `/api/installations/{id}/widgets` bereits Temperaturdaten. Diese sind weniger aktuell (15min Polling) aber erfordern keine Abhängigkeit zur BauerGroup-Integration.

**Option B — State-Lookup:**
```python
# In der Cloud-Integration:
local_temp = self.hass.states.get("sensor.hg_pk32_kesseltemperatur")
if local_temp and local_temp.state not in ("unknown", "unavailable"):
    return float(local_temp.state)
# Fallback auf Cloud-Wert
```
Vorteil: Echtzeit-Temperatur. Nachteil: Lose Kopplung, bricht wenn BauerGroup-Integration nicht installiert.

## Config Flow — Cloud-Integration

```
Schritt 1: Hargassner Connect Login
  ├─ E-Mail-Adresse
  └─ Passwort

Schritt 2: Installation auswählen (automatisch erkannt)
  └─ Dropdown falls mehrere Anlagen im Account

Schritt 3: Optionen
  ├─ Polling-Intervall (Standard: 15 Minuten)
  └─ BauerGroup-Integration Entity-Prefix (optional, für Cross-Lookup)
```

## Risiken & Mitigations

| Risiko | Wahrscheinlichkeit | Mitigation |
| --- | --- | --- |
| Hargassner ändert JS-Bundle-Format | Mittel | Regex-Pattern anpassbar, Monitoring auf Fehler |
| API-Endpunkte ändern sich | Niedrig | Versionierung der API-Pfade, Error-Logging |
| OAuth-Secrets werden rotiert | Niedrig | Self-Healing durch dynamische Extraktion |
| Internet-Ausfall | Sicher (temporär) | Letzten bekannten State cachen, graceful degradation |
| Hargassner blockiert API-Zugang | Niedrig | Rate-Limiting einbauen, User-Agent setzen |

## Roadmap

1. **Phase 1:** Eigenständige Cloud-Integration mit Number/Select/Button (wie Knirzinger aktuell)
2. **Phase 2:** ClimateEntity mit HVAC-Mode-Mapping und Temperatursteuerung
3. **Phase 3:** Optional Cross-Integration State-Lookup für Echtzeit-Temperaturen
4. **Phase 4:** Multi-Heizkreis-Support (falls API mehrere Heating Circuits unterstützt)

## Referenzen

- Knirzinger Repo: https://github.com/knirzinger/hargassner-ha
- HA Climate Entity Docs: https://developers.home-assistant.io/docs/core/entity/climate
- BauerGroup Integration: https://github.com/bauer-group/IP-HargassnerIntegration
