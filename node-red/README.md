# Node-RED: Nächtliche Batterie-Entladung auf 10 %

Zusätzlicher Flow für die bestehende Victron/MultiPlus-II ESS-Steuerung.
Er entlädt den Batteriespeicher **von 00:00 bis 07:00 Uhr auf 10 % SoC** und sorgt
dabei – im Rahmen der Wechselrichter-Leistung – dafür, dass **kein Netzbezug**
entsteht.

## Dateien

| Datei | Inhalt |
|-------|--------|
| `night-discharge-flow.json` | Nur die **neuen** Nodes – zum Importieren in eine bestehende Installation. |
| `flows.json` | Bestehende Flows **inkl.** der neuen Nodes (komplette Konfiguration). |

## Funktionsweise

1. **Start 00:00** (`inject`, cron `00 00 * * *`)
   → setzt ESS-Modus auf **3 (External control)** und initialisiert die Steuergrößen.
2. **Zyklische Prüfung (alle 30 s)** (`inject` → Function `Nachtentladung Regelung`)
   berechnet in jedem Zyklus zwei Größen und nimmt das Maximum:
   - **Geplante Entladeleistung**, um bis 07:00 auf 10 % zu kommen:
     `(SoC − 10) % × Kapazität ÷ verbleibende Stunden`.
   - **Lastdeckung** (Netz-Rückkopplung): aktuelle Hauslast ≈
     `aktuelle Entladung + Netzleistung`. Ist der berechnete Wert zu klein
     (→ Netzbezug), wird die Entladeleistung erhöht.
   - Ergebnis wird auf die **maximale Wechselrichterleistung** begrenzt
     (`P_MAX`). Reicht die nicht aus, ist minimaler Netzbezug unvermeidbar
     ("im Rahmen der Möglichkeiten").
   - Der Sollwert wird negativ auf `/Hub4/L1/AcPowerSetpoint` geschrieben
     (negativ = entladen) und `flow.PowerL1SetPoint` konsistent gesetzt.
3. **Ziel erreicht**: Sobald `SoC ≤ 10 %`, wird der Sollwert auf 0 gesetzt und
   ESS auf Modus **1** zurückgestellt.
4. **Stop 07:00** (`inject`, cron `00 07 * * *`) → Sollwert 0, ESS-Modus 1.

Die Netzleistung wird über einen zusätzlichen `victron-input-gridmeter`
(`com.victronenergy.grid/40`, `/Ac/Power`) gelesen und in `flow.GridPower`
abgelegt. Der SoC wird aus dem bereits vorhandenen `flow.ESS_SoC` verwendet.

## Wichtige Hinweise / anzupassende Parameter

In der Function **`Nachtentladung Regelung`** oben anpassen:

```js
const CAPACITY_WH = 16000; // Akku-Nutzkapazität in Wh
const P_MAX       = 3800;  // max. Entladeleistung des Wechselrichters (W)
const SOC_TARGET  = 10;    // Ziel-SoC (%)
const GRID_TARGET = -50;   // Ziel-Netzleistung (W); negativ = Einspeise-Puffer (nie Bezug)
```

- **HomeAssistant-Steuerung im Fenster 00:00–07:00 pausieren.** Die bestehende
  2‑s‑Schleife (`PV/MP2/PowerL1SetPoint` → `function 2` → `AcPowerSetpoint`)
  würde sonst gegen diesen Flow arbeiten. Dieser Flow setzt `flow.PowerL1SetPoint`
  selbst, damit die 2‑s‑Schleife denselben Wert anwendet – HA darf in diesem
  Zeitraum aber nichts Eigenes senden.
- Die geplante Entladeleistung darf überschüssige Energie ins Netz einspeisen,
  um 10 % bis 07:00 sicher zu erreichen (so abgestimmt). Kein Netz**bezug**.
- Vorzeichen-Konvention (wie im bestehenden Flow): `AcPowerSetpoint` negativ =
  entladen, positiv = laden.

## Import

Node-RED → Menü → *Import* → Inhalt von `night-discharge-flow.json` einfügen.
Die neuen Nodes hängen sich an den vorhandenen Tab `Flow 1` und nutzen die
bereits konfigurierten Output-Nodes für ESS-Modus und AC-Sollwert.
