# imageconverter

`wim2qcow2` konvertiert Windows-`WIM`-Images in `qcow2`.

Es gibt zwei Modi:

- `full-disk-gpt`: bootfaehiges Voll-Disk-Image fuer KVM/Proxmox
- `linuxmuster-partition`: Linuxmuster-/LINBO-Partitionsimage

## Linuxmuster-Zielbild

Der Linuxmuster-Modus soll nicht einfach ein nacktes NTFS-Image erzeugen und den Rest mit immer mehr `.prestart`/`.postsync` reparieren.

Stattdessen soll er sich moeglichst eng an dem orientieren, was LINBO selbst bei `linbo_create_image` aus einem fertig installierten Windows-System macht:

- Das BaseImage ist ein `qcow2` der Windows-Partition, nicht der ganzen Platte.
- In der Windows-Partition liegen Linuxmuster-Bootmetadaten:
  - `.linbo`
  - `.guid.disk`
  - `.guid.efi`
  - `.guid.part`
  - `EFI/Microsoft/Boot/...`
  - gruppenspezifische Backups wie `BCD.<gruppe>.<partition>`, `bsmbr.<gruppe>`, `ntfs.id`
- `linbo_sync` restauriert spaeter genau diese Daten auf dem Zielclient.

Der richtige Weg fuer WIM -> Linuxmuster ist deshalb:

1. WIM auf ein NTFS-Partitionsimage anwenden
2. das fertige `qcow2` offline Linuxmuster-tauglich praegen
3. dabei einen funktionierenden Linuxmuster-Donor verwenden
4. `prestart`/`postsync` nur noch als leere Kompatibilitaetsdateien mitfuehren oder ganz weglassen

## Warum dieser Weg

Ein klassischer Linuxmuster-Windows-Workflow ist:

1. LINBO partitioniert den Client
2. Windows wird normal installiert
3. danach erstellt LINBO aus der fertigen Windows-Partition das Image

Dabei entstehen automatisch die Linuxmuster-Metadaten in der Partition. Ein direkt aus einem `WIM` gebautes Image hat diese Metadaten zunaechst nicht. Genau diese Luecke schliesst der Konverter.

## Aktueller Stand

Der Konverter uebernimmt inzwischen aus einem Linuxmuster-Referenz-`qcow2`:

- `.guid.disk`
- `.guid.efi`
- `.guid.part`
- `EFI/Microsoft/Boot`
- `.linbo` wird passend zum neuen Imagenamen geschrieben
- `BCD.<gruppe>.<partition>` wird aus dem aktuellen `BCD` neu erzeugt
- `bsmbr.<gruppe>` wird auf die Zielgruppe normiert
- `ntfs.id` wird aus dem frisch erzeugten NTFS-Volume neu geschrieben
- fuer dieselbe Partitionsnummer werden mehrere gaengige Partitionsnamen-Aliase erzeugt

Bei donor-basierten Linuxmuster-Builds werden die Hooks jetzt nur noch als `no-op` erzeugt:

- `postsync` ist leer
- `prestart` ist leer

Der Grund ist die inzwischen verifizierte Eigenschaft:

- LINBO verarbeitet den im Image gepraegten Linuxmuster-Bootsatz selbst ausreichend
- der Bootpfad funktioniert damit ohne eigene Hook-Logik

Wenn kein vollstaendiger Donor vorhanden ist, faellt das Tool auf die bisherige interne BCD-/Hook-Logik zurueck.

## Verifizierter Stand

Am `14. April 2026` wurde der donor-basierte Linuxmuster-Pfad praktisch verifiziert:

1. frischer Build aus `39ef39cf-9848-4766-bb75-6cab64845a06.wim`
2. donor `win11_pro_edu.qcow2`
3. `Sync+Start` mit `postsync = no-op`
4. Boot erfolgreich
5. weiterer Test mit `prestart = no-op` und `postsync = no-op`
6. Boot ebenfalls erfolgreich

Damit ist fuer den donor-basierten Linuxmuster-Pfad belegt:

- der Bootsatz im Partitionsimage reicht aus
- `prestart` ist nicht erforderlich
- `postsync` ist nicht erforderlich

## Wichtige Trennung

Ein separates Thema bleibt moeglich:

- das Verhalten des LINBO-Start-Buttons bzw. des LINBO-GUI-Wrappers

Dieses Thema darf nicht mit dem Imageaufbau verwechselt werden.
Der Konverter gilt technisch als erfolgreich, wenn:

- `Sync+Start` in Windows bootet
- das Image nach einem Reboot wieder in Windows startet
- dies ohne aktive Hook-Logik funktioniert

## Benutzung

Interaktiv:

```bash
./wim2qcow2
```

Linuxmuster mit Referenzimage:

```bash
./wim2qcow2 \
  --input /srv/linbo/images/logo/39ef39cf-9848-4766-bb75-6cab64845a06.wim \
  --output /srv/linbo/images/win11_pro/win11_pro.qcow2 \
  --layout linuxmuster-partition \
  --start-conf /srv/linbo/start.conf.win11_pro \
  --copy-sidecars-from /srv/linbo/images/win11_pro_edu \
  --sanitize-logodidakt
```

Im Referenzverzeichnis werden verwendet:

- `.reg`
- `.desc`
- die enthaltene `*.qcow2` als Linuxmuster-Donor fuer `.guid.*` und `EFI/Microsoft/Boot`
