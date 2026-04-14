# Linuxmuster-Zielzustand

## Problem

Ein direkt aus einem `WIM` erzeugtes Windows-Partitionsimage ist nicht automatisch so strukturiert, wie Linuxmuster/LINBO ein selbst erstelltes Windows-Image erwartet.

Bei einem nativen Linuxmuster-Windows-Workflow passiert mehr als nur `Partition -> qcow2`:

- EFI-Bootdateien werden in die Windows-Partition zurueckgesichert
- GUID-Metadaten werden in der Windows-Partition abgelegt
- BCD-Backups werden gruppen- und partitionsspezifisch gespeichert
- `linbo_sync` restauriert spaeter genau diese Daten

## Massstab

Der technische Massstab ist nicht „Windows irgendwie bootfaehig machen“, sondern:

> Das WIM-basierte Image soll sich fuer LINBO moeglichst wie ein mit `linbo_create_image` erzeugtes Linuxmuster-Windows-Image verhalten.

## Relevantes Linuxmuster-Verhalten

Aus `linbofs64` extrahierte zentrale Stellen:

- `linbo_create_image`:
  - speichert EFI-Bootdateien aus der echten EFI-Partition in der Windows-Partition
  - schreibt `.guid.disk`, `.guid.efi`, `.guid.part`
  - legt `BCD.<HOSTGROUP>.<partition>`, `bsmbr.<HOSTGROUP>`, `ntfs.id` ab
- `linbo_sync`:
  - restauriert `BCD`
  - restauriert GPT-/Partition-GUIDs
  - restauriert `ntfs.id`
  - sourced erst danach optional `.postsync`
- `linbo_start`:
  - sourced optional `.prestart`

Das bedeutet:

- `.postsync` und `.prestart` sind in Linuxmuster erlaubt, aber nicht der eigentliche Kernmechanismus
- der Kernmechanismus sitzt in der **vorbereiteten Windows-Partition selbst**

## Zielarchitektur fuer `wim2qcow2`

Der Linuxmuster-Modus soll in dieser Reihenfolge arbeiten:

1. `WIM` auf ein NTFS-Partitionsimage anwenden
2. Windows-Setup-/Sysprep-/LogoDidakt-Reste offline bereinigen
3. das fertige Partitionsimage offline Linuxmuster-tauglich praegen

Dabei sollen moeglichst dieselben Artefakte entstehen wie nach `linbo_create_image`:

- `.linbo`
- `.guid.disk`
- `.guid.efi`
- `.guid.part`
- `EFI/Microsoft/Boot/...`
- nach Moeglichkeit:
  - `BCD.<gruppe>.<partition>`
  - `bsmbr.<gruppe>`
  - `ntfs.id`

## Donor-Strategie

Da beim WIM-Weg keine zuvor von LINBO erstellte Windows-Partition existiert, wird ein funktionierendes Linuxmuster-Referenzimage als Donor verwendet.

Wichtig:

- Das Referenz-`qcow2` ist typischerweise selbst nur ein NTFS-Partitionsimage.
- Es enthaelt **keine echte EFI-Partition**, aber sehr wohl:
  - `.guid.*`
  - `EFI/Microsoft/Boot`
  - Linuxmuster-spezifische Backups im Image

Deshalb ist der richtige Donor-Ansatz:

- nicht „EFI-Partition aus dem Donor ziehen“
- sondern „Linuxmuster-Bootsatz aus der Donor-Windows-Partition uebernehmen“

## Hook-Philosophie

Ziel:

- `prestart` leer oder entbehrlich
- `postsync` leer oder entbehrlich

Erlaubt bleibt:

- leere Kompatibilitaetsdateien, wenn LINBO/Workflows sie erwarten

Nicht gewuenscht ist:

- komplexe BCD-Neuerzeugung im Hook
- umfangreiche GUID-/BCD-Sonderlogik im Hook
- dauerhafte Abhaengigkeit von Workaround-Logik, die am nativen Linuxmuster-Mechanismus vorbeigeht

## Aktueller Umsetzungsstand

Bereits umgesetzt:

- Donor-`qcow2` wird aus `--copy-sidecars-from` erkannt
- `.guid.disk`, `.guid.efi`, `.guid.part` werden aus dem Donor ins neue Image uebernommen
- `EFI/Microsoft/Boot` wird aus dem Donor ins neue Image uebernommen
- `.linbo` wird passend zum neuen Imagenamen geschrieben
- `BCD.<gruppe>.<partition>` wird aus dem aktuellen `BCD` des neuen Images neu erzeugt
- fuer dieselbe Partitionsnummer werden mehrere gaengige Geraetenamen-Aliase erzeugt (`disk0p3`, `sda3`, `vda3`, `nvme0n1p3`, ...)
- `bsmbr.<gruppe>` wird donor-basiert auf die Zielgruppe normiert
- `ntfs.id` wird aus dem frisch erzeugten NTFS-Volume des neuen Images gelesen und im Bootsatz aktualisiert
- bei vorhandenem Donor werden `prestart` und `postsync` nur noch als `no-op` erzeugt
- ohne vollstaendigen Donor bleibt der alte Fallback aktiv

Noch offen:

- pruefen, ob die `no-op`-Hooks ganz entfallen koennen
- testen, welche der erzeugten Partitions-Aliase fuer `BCD.<gruppe>.<partition>` in gemischter Hardware wirklich notwendig sind

## Verifikation

Am `14. April 2026` wurde der donor-basierte Linuxmuster-Pfad praktisch verifiziert:

1. frischer Build aus dem WIM
2. Donor-Seed aus `win11_pro_edu.qcow2`
3. `postsync` auf `no-op`
4. `Sync+Start` erfolgreich
5. anschliessend `prestart` ebenfalls auf `no-op`
6. Reboot/Startpfad ebenfalls erfolgreich

Der relevante Schluss daraus:

- die Bootinformationen im Partitionsimage sind fuer LINBO ausreichend
- `prestart` und `postsync` sind im donor-basierten Linuxmuster-Pfad nicht mehr der Mechanismus
- Hooks koennen in diesem Pfad leer bleiben

## Separates Thema

Nicht Teil des Konverter-Ziels ist ein moegliches GUI-/Wrapper-Verhalten von LINBO, bei dem ein Start aus der Oberflaeche nicht immer sichtbar sofort rebootet.

Das muss getrennt betrachtet werden:

- **Konverter-/Image-Thema**: Windows bootet mit dem gepraegten Linuxmuster-Image
- **LINBO-GUI-/Wrapper-Thema**: wie der Startbutton den Reboot ausloest

Diese Themen duerfen in kuenftigen Sessions nicht vermischt werden.

## Leitregel fuer kuenftige Sessions

Wenn eine neue Idee auftaucht, immer zuerst pruefen:

1. Entspricht sie dem Verhalten von `linbo_create_image` / `linbo_sync`?
2. Verschiebt sie Logik aus Hooks zurueck ins Partitionsimage?
3. Reduziert sie Sonderlogik statt neue Boot-Workarounds zu stapeln?

Wenn die Antwort `nein` ist, ist es wahrscheinlich nicht der richtige Weg.
