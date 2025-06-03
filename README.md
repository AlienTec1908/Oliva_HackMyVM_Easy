# Oliva - HackMyVM (Easy)

![Oliva.png](Oliva.png)

## Übersicht

*   **VM:** Oliva
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=Oliva)
*   **Schwierigkeit:** Easy
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 2. Juni 2025
*   **Original-Writeup:** https://alientec1908.github.io/Oliva_HackMyVM_Easy/
*   **Autor:** Ben C.

## Kurzbeschreibung

Die Challenge "Oliva" ist eine als "Easy" eingestufte virtuelle Maschine von der Plattform HackMyVM. Das Ziel ist es, Benutzerzugriff und anschließend Root-Rechte auf dem System zu erlangen. Der Lösungsweg beinhaltet die Entdeckung einer herunterladbaren, LUKS-verschlüsselten Datei über einen Hinweis auf der Webseite, das Knacken der LUKS-Passphrase, das Extrahieren eines SSH-Passworts aus dem entschlüsselten Container für den initialen Benutzerzugriff und die anschließende Privilegienerweiterung durch Ausnutzung einer Linux Capability des `nmap`-Programms, um Datenbank-Credentials auszulesen und darüber das Root-Passwort zu finden.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap` (inkl. NSE)
*   Texteditor (z.B. `vi`, `nano`)
*   `curl`
*   `feroxbuster`
*   `file`
*   `cryptsetup`
*   `bruteforce-luks`
*   `find`
*   `mkdir`
*   `mount`
*   `ssh`
*   `ss`
*   `mysql` (MariaDB Client)
*   `getcap`
*   Standard Linux-Befehle (`ls`, `cat`, `cp`, `echo`, `su`)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "Oliva" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   Identifizierung der Ziel-IP (`192.168.2.210`) mittels `arp-scan`.
    *   Portscan mit `nmap` offenbarte offene Ports 22 (SSH - OpenSSH 9.2p1) und 80 (HTTP - nginx 1.22.1). Auf Port 80 lief eine Standard-nginx-Seite sowie eine `index.php`.
    *   Hinzufügen eines Eintrags (`192.168.2.210 oliva.hmv`) zur lokalen `/etc/hosts`-Datei.

2.  **Web Enumeration & Initial Foothold Preparation:**
    *   `feroxbuster` auf Port 80 fand die Dateien `/`, `/index.html`, `/index.php` und eine auffällig große Datei/Verzeichnis `/oliva`.
    *   Die Seite `/index.php` enthielt den Text "Hi oliva, Here the pass to obtain root: CLICK!", wobei "CLICK!" ein Link zur Ressource `/oliva` war.
    *   Die heruntergeladene Datei `/oliva` wurde mittels `file` als LUKS-verschlüsselter Container identifiziert.
    *   Die Passphrase für den LUKS-Container wurde mittels `bruteforce-luks` und der `rockyou.txt`-Wortliste als `bebita` geknackt.

3.  **Initial Access (SSH als `oliva`):**
    *   Der LUKS-Container wurde mit `cryptsetup luksOpen` und der Passphrase `bebita` geöffnet und als `/dev/mapper/oliva_decrypted` gemappt.
    *   Das entschlüsselte Dateisystem wurde gemountet (`mount /dev/mapper/oliva_decrypted /mnt/oliva_decrypted_content`).
    *   Im gemounteten Verzeichnis wurde die Datei `mypass.txt` gefunden, die das Passwort `Yesthatsmypass!` enthielt.
    *   Erfolgreicher SSH-Login auf `oliva.hmv` als Benutzer `oliva` mit dem Passwort `Yesthatsmypass!`.
    *   Auslesen der User-Flag `HMVY0H8NgGJqbFzbgo0VMRm` aus `/home/oliva/user.txt`.

4.  **Post-Exploitation / Privilege Escalation (Vorbereitung):**
    *   Enumeration als Benutzer `oliva`. `ss -tulnp` zeigte einen lokal lauschenden MySQL/MariaDB-Dienst auf Port 3306.
    *   Versuche, sich mit dem Passwort `Yesthatsmypass!` bei MySQL als `root` oder `oliva` anzumelden, scheiterten.
    *   `getcap -r / 2>/dev/null` (und spezifischere Suchen) zeigten, dass `/usr/bin/nmap` die Linux Capability `cap_dac_read_search=eip` besaß.

5.  **Privilege Escalation (von `oliva` zu root):**
    *   Ein benutzerdefiniertes Nmap Scripting Engine (NSE) Skript (`/tmp/readshadow.nse`) wurde erstellt, um die `cap_dac_read_search`-Fähigkeit von `nmap` auszunutzen und beliebige Dateien zu lesen.
    *   Zunächst wurde versucht, `/etc/shadow` zu lesen (erfolgreich), das Knacken der Hashes scheiterte jedoch.
    *   Das NSE-Skript wurde angepasst, um `/var/www/html/index.php` zu lesen.
    *   Die Ausführung von `nmap --script /tmp/readshadow.nse 127.0.0.1` als `oliva` las den Inhalt von `/var/www/html/index.php` und offenbarte MySQL-Credentials: User `root`, Passwort `Savingmypass` für die Datenbank `easy` auf `localhost`.
    *   Erfolgreicher Login in die MySQL-Datenbank mit `mysql -u root -pSavingmypass`.
    *   In der Datenbank `easy`, Tabelle `logging`, wurde der Eintrag `id_log=1, uzer=root, pazz=OhItwasEasy!` gefunden.
    *   Erfolgreicher Wechsel zum System-Root-Benutzer mit `su -` und dem Passwort `OhItwasEasy!`.
    *   Auslesen der Root-Flag `HMVnuTkm4MwFQNPmMJHRyW7` aus `/root/rutflag.txt`.

## Wichtige Schwachstellen und Konzepte

*   **Informationspreisgabe auf Webseite:** Ein direkter Hinweis auf eine Datei (`/oliva`), die zum Root-Passwort führt, wurde auf `index.php` gegeben.
*   **Schwache LUKS-Passphrase:** Die Passphrase `bebita` für den LUKS-Container konnte mittels Wörterbuchangriff geknackt werden.
*   **Passwort im Klartext:** Das SSH-Passwort für `oliva` (`Yesthatsmypass!`) wurde im Klartext im entschlüsselten LUKS-Container gespeichert.
*   **Unsichere Linux Capabilities:** Dem `nmap`-Binary war die Capability `cap_dac_read_search=eip` zugewiesen, was das Lesen beliebiger Dateien durch den ausführenden Benutzer (hier `oliva`) ermöglichte, selbst wenn dieser keine direkten Leserechte auf die Dateien hatte.
*   **Datenbank-Credentials im Web-Quellcode:** Die Anmeldeinformationen für den MySQL-Datenbankserver (`root:Savingmypass`) waren im Klartext in der Datei `/var/www/html/index.php` enthalten.
*   **Speicherung von System-Credentials in Datenbank:** Das System-Root-Passwort (`OhItwasEasy!`) wurde im Klartext in einer Datenbanktabelle gespeichert.

## Flags

*   **User Flag (`/home/oliva/user.txt`):** `HMVY0H8NgGJqbFzbgo0VMRm`
*   **Root Flag (`/root/rutflag.txt`):** `HMVnuTkm4MwFQNPmMJHRyW7`

## Tags

`HackMyVM`, `Oliva`, `Easy`, `LUKS`, `Password Cracking`, `Linux Capabilities`, `Nmap NSE`, `MySQL`, `Credentials Exposure`, `Linux`, `Web`, `Privilege Escalation`
