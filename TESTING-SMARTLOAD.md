# Testinstallation SmartLoad (nur Test, nicht für Upstream)

> **Achtung:** Dieser Branch funktioniert nur, wenn vorher das passende Wheel unter
> `/config/wheels/huawei_solar-3.2.4.post5+smartload-py3-none-any.whl` liegt (Prüfsumme siehe unten).
> Ohne die Datei kann HA die Abhängigkeit nicht installieren und die Integration lädt nicht.
> Für den normalen Betrieb und für den Pull Request gilt `smartload-device`.

Seit 24.11. beruhen Bibliothek und Integration auf dem `develop`-Stand (wie die PRs), vorher auf `main`
(alter Stand: Branches `smartload-test-main`). Neu: SmartLoad-Einstellungen schreibbar (Nennleistung,
Intervall, SOC-Grenzen, Modus, Automatik), Schalter ein/aus (40007) und Steuerung manuell/automatisch (40006).
Die Eingabefelder erscheinen nur, wenn in der Integration die Parameterkonfiguration aktiviert ist.

Dieser Branch (`smartload-test`) ist `smartload-device` plus **eine** Teständerung in `manifest.json`:
Statt der Upstream-Abhängigkeit `huawei-solar==3.2.2` (Stand `develop`) wird ein lokal gebautes Wheel der Bibliothek
mit fester Revision installiert. Upstream bleibt unverändert: Der PR-Branch `smartload-device`
verlangt weiterhin `huawei-solar==3.2.2`; nach einem Bibliotheks-Release mit SmartLoad hebt der
Maintainer die Version dort an.

## Feste Revisionen

| Teil | Branch | Commit |
|---|---|---|
| Bibliothek (PR) | `huawei-solar-lib` `smartload-device` | `b805113` |
| Bibliothek (Test-Wheel) | `huawei-solar-lib` `smartload-test` | `98032530383ce0f1b35fe46f2e799997c95845ab` (= PR-Stand `b805113` auf `develop` + feste Version `3.2.4.post5+smartload`) |
| Integration (PR) | `huawei_solar` `smartload-device` | `23a6495` (Basis `develop`) |

Wheel: `huawei_solar-3.2.4.post5+smartload-py3-none-any.whl`,
SHA-256 `09a523df3220a0d44db8e4a5dc6e49c1c0fc1ea552fa792faf9e5b4a9b25406f`
(reproduzierbar: zweimal gebaut, gleiche Prüfsumme). Die Prüfsumme steht auch in `manifest.json`;
HA bricht die Installation ab, wenn das Wheel abweicht.

Bezugsquellen:
- Bibliotheksrevision: https://github.com/mab-wien/huawei-solar-lib/tree/smartload-test
  (Commit `98032530383ce0f1b35fe46f2e799997c95845ab`)
- Wheel: Pre-Release `smartload-test-9803253` im Fork mab-wien/huawei-solar-lib (Asset
  `huawei_solar-3.2.4.post5+smartload-py3-none-any.whl`), oder selbst bauen wie unten.
  Vor der Verwendung immer die Prüfsumme kontrollieren.

Wheel neu bauen:
```
mkdir -p /tmp/hsl
git -C huawei-solar-lib archive --format=tar --prefix=src/ 98032530383ce0f1b35fe46f2e799997c95845ab | tar -x -C /tmp/hsl
pip wheel --no-deps -w /tmp/hsl/out /tmp/hsl/src
sha256sum /tmp/hsl/out/*.whl
```

## Installation (HA)

1. Sicherung, **nur solange dort noch die offizielle Integration liegt**:
   `cp -a /config/custom_components/huawei_solar /config/huawei_solar.bak`
   (prüfen: `grep huawei-solar /config/custom_components/huawei_solar/manifest.json` zeigt
   `huawei-solar==…`). Ist bereits eine Testversion installiert, keine neue Sicherung anlegen und
   für den Rückweg HACS „Erneut installieren“ beim offiziellen Eintrag verwenden.
2. Wheel nach `/config/wheels/` laden und prüfen (Dateiname unverändert):
   ```
   mkdir -p /config/wheels
   wget -O "/config/wheels/huawei_solar-3.2.4.post5+smartload-py3-none-any.whl" \
     "https://github.com/mab-wien/huawei-solar-lib/releases/download/smartload-test-9803253/huawei_solar-3.2.4.post5%2Bsmartload-py3-none-any.whl"
   sha256sum /config/wheels/huawei_solar-3.2.4.post5+smartload-py3-none-any.whl
   ```
3. Inhalt dieses Branches nach `/config/custom_components/huawei_solar/` kopieren.
4. HA neu starten. HA installiert das Wheel, weil die Abhängigkeit eine Datei-URL ist.
5. Prüfen: `docker exec homeassistant pip show huawei-solar` → `Version: 3.2.4.post5+smartload`.
6. Slave-ID der SmartLoad (EMMA-Geräteliste, z. B. 38) unter „Neu konfigurieren“ ergänzen.

## Zurück zur regulären Version

1. Bei der Integration die SmartLoad-Slave-ID wieder entfernen (sonst schlägt die Erkennung mit der
   regulären Bibliothek fehl).
2. In HACS beim offiziellen Eintrag „Erneut installieren“, oder – nur wenn die Sicherung nachweislich
   die offizielle Version enthält (`grep huawei-solar /config/huawei_solar.bak/manifest.json` zeigt
   `huawei-solar==…`) – `cp -a /config/huawei_solar.bak/. /config/custom_components/huawei_solar/`.
3. HA neu starten. `huawei-solar==3.2.4` ist dann nicht erfüllt (`3.2.4.post5+smartload` installiert),
   HA installiert die reguläre Bibliothek 3.2.4 automatisch.
4. Geladene Version kontrollieren: `docker exec homeassistant pip show huawei-solar` muss
   `Version: 3.2.4` zeigen. Steht dort noch `3.2.4.post5+smartload`, HA ein weiteres Mal neu starten
   und im Protokoll nach Fehlern bei der Installation von `huawei-solar` suchen.
5. Erst danach `/config/wheels/huawei_solar-3.2.4.post5+smartload-py3-none-any.whl` löschen.
