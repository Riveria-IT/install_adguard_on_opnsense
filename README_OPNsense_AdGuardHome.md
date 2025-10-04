# OPNsense + AdGuard Home – vollständige DNS-Filterung über AdGuard

Diese Anleitung beschreibt exakt die Schritte, die wir durchgeführt haben, um **AdGuard Home direkt auf OPNsense** zu installieren und **den gesamten DNS‑Verkehr** (Router + Clients) über AdGuard zu leiten – inkl. DHCP‑Verteilung, NAT‑Redirect, Tests und Rückweg.

> Getestet mit **OPNsense 25.7 (amd64)**. Ersetze IPs & Namen passend zu deiner Umgebung.
> Im Beispiel: **OPNsense LAN‑IP = 192.168.1.1**, AdGuard läuft **auf der Firewall** selbst.

---

## 0) Voraussetzungen / Zielbild

- OPNsense hat Internetzugang.
- Wir installieren **AdGuard Home** als **Plugin** über das **mimugmail** Repository.
- **Unbound** wird **deaktiviert**, damit Port **53/TCP+UDP** für AdGuard frei ist.
- OPNsense selbst **und alle Clients** fragen **AdGuard**.
- Optional erzwingen wir per **NAT‑Redirect**, dass hartkodierte DNS (z. B. 8.8.8.8) ebenfalls über AdGuard laufen.

---

## 1) Dritt‑Repository (mimugmail) hinzufügen

In der **OPNsense‑Shell** (Konsole oder SSH):

```sh
# Ordner anlegen (falls nicht vorhanden)
mkdir -p /usr/local/etc/pkg/repos

# Repo‑Config holen
fetch -o /usr/local/etc/pkg/repos/mimugmail.conf \
  https://www.routerperformance.net/mimugmail.conf

# (optional, häufig nötig) Signatur-Fingerprint hinterlegen
mkdir -p /usr/local/etc/pkg/fingerprints/MIMUGMAIL
fetch -o /usr/local/etc/pkg/fingerprints/MIMUGMAIL/trusted \
  https://www.routerperformance.net/mimugmail.pub

# Repos aktualisieren
pkg update -f
```

> **Hinweis:** Wenn `pkg update` eine Signatur-/Fingerprint‑Meldung bringt, lösche lokale Repo‑Caches und wiederhole:
> ```sh
> rm -rf /var/db/pkg/repo-*.sqlite /var/cache/pkg/*
> pkg update -f
> ```

---

## 2) AdGuard‑Plugin installieren

**Variante A – Shell:**
```sh
pkg search adguard            # sollte u. a. os-adguardhome-maxit anzeigen
pkg install -y os-adguardhome-maxit
```

**Variante B – GUI:**
- **System → Firmware → Plugins**
- Aktualisieren (Kreispfeil), nach **adguard** suchen
- **os-adguardhome-maxit** installieren

Nach der Installation erscheint **Dienste → AdGuard Home** im Menü.

---

## 3) Port 53 freigeben: Unbound deaktivieren

**GUI → Dienste → Unbound‑DNS → Allgemein**  
- Haken bei **„Aktiviere Unbound“** **entfernen** → **Anwenden**

**(Optional) in der Shell zusätzlich absichern:**
```sh
service unbound stop
sysrc unbound_enable=NO
```

**Kontrolle (Port 53):**
```sh
sockstat -4 -l | grep ':53' || true
sockstat -6 -l | grep ':53' || true
```
Es sollte **kein** Dienst mehr auf `:53` lauschen, bevor AdGuard startet.

---

## 4) AdGuard Home auf OPNsense einschalten

**GUI → Dienste → Adguardhome → Allgemein**  
- ✅ **Aktivieren**  
- ✅ **Primary DNS** (OPNsense selbst nutzt dann AdGuard)  
- **Speichern**

**AdGuard Web‑UI öffnen:** `http://192.168.1.1:3000`  
Dort unter **Settings → DNS** eintragen:

- **Upstream DNS servers** (Beispiel DoH – je Zeile eine URL)
  ```
  https://dns.quad9.net/dns-query
  https://cloudflare-dns.com/dns-query
  ```
- **Bootstrap DNS**
  ```
  9.9.9.10
  1.1.1.1
  ```
- **Listen interfaces / Bind Hosts**: Alle Interfaces, Port **53** (TCP/UDP)  
- **Speichern** und Dienst ggf. **neu starten**

> Tipp: Lokale Namen kannst du in **AdGuard → DNS rewrites** pflegen, wenn Unbound komplett aus ist.

---

## 5) OPNsense selbst über AdGuard auflösen lassen

**GUI → System → Einstellungen → Allgemein**  
- **DNS‑Server**: `192.168.1.1` (LAN‑IP der Firewall – darauf läuft AdGuard)  
  → „Verwende Gateway“: **keiner**  
- **Erlaube das Überschreiben der DNS‑Serverliste durch DHCP/PPP auf WAN**: **Haken raus**  
- **Verwenden Sie den lokalen DNS‑Dienst nicht als Nameserver für dieses System**: **Haken setzen**  
- **Speichern**

> Dadurch nutzt OPNsense **AdGuard** als Resolver und nicht (mehr) Unbound.

---

## 6) DHCP: Clients auf AdGuard zeigen lassen

Je **LAN/VLAN** den aktiven DHCP‑Dienst anpassen:

**ISC DHCPv4:**
- **Dienste → DHCPv4 → [LAN/VLAN]**  
  **DNS‑Server**: `192.168.1.1` → **Speichern / Übernehmen**

**Kea DHCPv4:**
- **Dienste → Kea DHCPv4 → Subnetze → Bearbeiten (Stift)**  
  **DNS‑Server (IPv4)**: `192.168.1.1` → **Speichern / Anwenden**

*(IPv6: analog in DHCPv6/RA die IPv6 der Firewall eintragen, wenn AdGuard auch v6 bedient.)*

**Clients neu verbinden / Lease erneuern** (WLAN aus/an, `ipconfig /renew`, …).

---

## 7) (Empfohlen) DNS‑Redirect per NAT (gegen hartkodierte DNS)

Für **jedes** interne Interface/VLAN eine NAT‑Weiterleitung anlegen:

**GUI → Firewall → NAT → Port Forward → „+“**  
- **Interface:** jeweiliges LAN/VLAN  
- **TCP/IP Version:** IPv4 (oder **IPv4+IPv6**, wenn AdGuard v6 bedient)  
- **Protocol:** **TCP/UDP**  
- **Source:** any (oder das jeweilige Netz)  
- **Destination:** any  
- **Destination Port:** **DNS (53)**  
- **Redirect target IP:** `192.168.1.1`  
- **Redirect target port:** **53**  
- **Filter rule association:** **Add associated filter rule**  
- **Save → Apply**

**Optional (Bypass erschweren):**  
- **LAN/VLAN‑Regel:** **Block** `TCP 853` (DNS‑over‑TLS) nach *any*  
- **LAN/VLAN‑Regel:** **Block** `TCP/UDP 53` **zu „This firewall“**, wenn Clients den Router selbst nicht direkt fragen dürfen

---

## 8) Funktion testen

**Client im LAN:**
```sh
nslookup example.com 192.168.1.1
nslookup example.com 8.8.8.8   # sollte dank NAT-Redirect trotzdem über AdGuard gehen
```
Die Anfragen müssen im **AdGuard → Query Log** sichtbar sein.

**OPNsense selbst (Web‑GUI):**  
**System → Diagnosen → DNS‑Lookup** – Domain testen.

**OPNsense Shell (optional):**
```sh
drill example.com @127.0.0.1
sockstat -4 -l | grep ':53'
```

---

## 9) Troubleshooting

- **Plugin nicht sichtbar / Repo‑Fehler**
  - Repo + Fingerprint wie oben anlegen, dann:  
    ```sh
    rm -rf /var/db/pkg/repo-*.sqlite /var/cache/pkg/*
    pkg update -f
    ```

- **Port 53 belegt / AdGuard startet nicht**
  - Unbound wirklich aus?  
    ```sh
    service unbound stop; sysrc unbound_enable=NO
    sockstat | grep ':53'
    ```
  - Anderer Dienst auf 53? Dann anhalten/deaktivieren.

- **Clients umgehen AdGuard**
  - DHCP liefert `192.168.1.1`? Lease erneuert?
  - NAT‑Redirect pro Interface angelegt? (TCP/UDP 53 → 192.168.1.1:53, mit verknüpfter Filterregel)

- **OPNsense hat kein DNS, wenn AdGuard down ist**
  - In diesem Setup (**Alles über AdGuard**) ist das erwartbar. Für mehr Robustheit: OPNsense auf **Unbound** umstellen **oder** zweite AGH‑Instanz als Fallback eintragen.

---

## 10) Rückgängig machen / Entfernen

- **AdGuard deaktivieren** (GUI → Dienste → Adguardhome → Allgemein → Haken raus)
- **Unbound wieder aktivieren** (GUI → Dienste → Unbound‑DNS → Allgemein → Haken setzen → Anwenden)
- **System → Einstellungen → Allgemein**: DNS‑Server auf gewünschte Resolver ändern (z. B. `127.0.0.1` für Unbound) und „lokalen DNS‑Dienst nicht als Nameserver…“ **Haken entfernen**
- **NAT‑Redirect‑Regeln** löschen
- **DHCP DNS‑Server** zurück auf Wunschwerte
- **Plugin entfernen (optional):**
  ```sh
  pkg delete -y os-adguardhome-maxit
  ```

---

## 11) Kurzreferenz – Shell‑Kommandos

```sh
# Repo + Fingerprint
mkdir -p /usr/local/etc/pkg/repos
fetch -o /usr/local/etc/pkg/repos/mimugmail.conf https://www.routerperformance.net/mimugmail.conf
mkdir -p /usr/local/etc/pkg/fingerprints/MIMUGMAIL
fetch -o /usr/local/etc/pkg/fingerprints/MIMUGMAIL/trusted https://www.routerperformance.net/mimugmail.pub
pkg update -f

# Plugin installieren
pkg install -y os-adguardhome-maxit

# Unbound aus, AdGuard an
service unbound stop; sysrc unbound_enable=NO
sysrc adguardhome_enable=YES; service adguardhome start

# Prüfung Port 53
sockstat -4 -l | grep ':53' || true
sockstat -6 -l | grep ':53' || true
```

---

**Fertig!** Jetzt läuft deine OPNsense mit **AdGuard Home** als zentralem DNS‑Filter.  
Wenn du willst, ergänze ich gerne noch **IPv6‑Beispiele**, **Fallback‑Setup mit zweitem AdGuard** oder **DoH/DoT‑Sperrlisten**.
