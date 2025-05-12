# Flossy - HackMyVM (Medium) - Writeup

![Flossy Icon](Flossy.png)

Dieses Repository enthält einen zusammengefassten Bericht über die Kompromittierung der HackMyVM-Maschine "Flossy" (Schwierigkeitsgrad: Medium).

## Metadaten

*   **Maschine:** Flossy (HackMyVM - Medium)
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Flossy](https://hackmyvm.eu/machines/machine.php?vm=Flossy)
*   **Autor des Writeups:** DarkSpirit
*   **Datum:** 7. Dezember 2023
*   **Original Writeup:** [https://alientec1908.github.io/Flossy_HackMyVM_Medium/](https://alientec1908.github.io/Flossy_HackMyVM_Medium/)

## Zusammenfassung des Angriffspfads

Die initiale Erkundung mittels `nmap` identifizierte einen SSH-Dienst (Port 22) und einen HTTP-Server (Port 80), der von einem Node.js Express Framework betrieben wurde. Die Web-Enumeration (Nikto, Gobuster) war zunächst wenig erfolgreich, doch die manuelle Analyse des JavaScript-Codes der `index.html` offenbarte einen GraphQL-Endpunkt unter `/graphql`.

Durch Senden einer GraphQL-Introspection-Query an diesen Endpunkt wurde das API-Schema aufgedeckt, einschließlich potenziell sensibler Queries wie `users`. Die Ausnutzung der `users`-Query (vermutlich mittels Burp Intruder oder Skripting, um IDs zu iterieren) führte zum Leck von Klartext-Zugangsdaten (`malo`:`8YdsA3CkiWx968`).

Mit diesen Zugangsdaten gelang der initiale Zugriff via SSH als Benutzer `malo`. Die weitere Enumeration deckte ein Skript `/home/sophie/SSHKeySync` auf, das (vermutlich via Cronjob als `sophie`) minütlich den privaten SSH-Schlüssel von `sophie` an ein spezifisches TTY (`/dev/pts/24`) sendete. Durch Überwachen dieses TTYs (zufällig das richtige) konnte der private Schlüssel von `sophie` abgefangen werden. Dies ermöglichte den SSH-Login als `sophie`.

Als `sophie` wurde die User-Flag gelesen. Die `sudo -l`-Ausgabe zeigte, dass `sophie` das Skript `/home/sophie/network` als `root` ohne Passwort ausführen durfte. Die Analyse dieses Skripts offenbarte, dass die Option `disp` alle Dateien aus `/opt` nach `/etc/NetworkManager/dispatcher.d/` kopiert und ausführbar macht. Durch Erstellen eines bösartigen Skripts in `/opt` (das `chmod u+s /bin/bash` ausführt) und anschließendes Ausführen von `sudo /home/sophie/network disp` wurde dieses Skript in den NetworkManager-Dispatcher-Pfad kopiert. Das Auslösen eines Netzwerkereignisses (`nmcli con up lo`) führte zur Ausführung des Skripts als Root, wodurch das SUID-Bit auf `/bin/bash` gesetzt wurde. Das Ausführen von `bash -p` gewährte schließlich eine Root-Shell und Zugriff auf die Root-Flag.

## Verwendete Tools (Auswahl)

*   `arp-scan`
*   `vi` / `nano` (implied)
*   `nmap`
*   `nikto`
*   `gobuster`
*   Web Browser (Source Inspection)
*   Burp Suite (Intruder, oder alternative Skripting-Methode)
*   `curl`
*   `ssh`
*   `find`
*   `getcap`
*   `id`
*   `cat`
*   `ll` (alias for `ls -l`)
*   `ssh-keygen`
*   `tty`
*   `sudo`
*   `echo`
*   `chmod`
*   `nmcli`
*   `bash`
*   `ls`
*   `cd`

## Angriffsschritte (Übersicht)

1.  **Reconnaissance:** Ziel-IP (`arp-scan`), Hostname (`flossy.hmv` in `/etc/hosts`), Portscan (`nmap`) -> Port 22 (SSH), Port 80 (Node.js/Express).
2.  **Web Enumeration:** `nikto`, `gobuster` (wenig Erfolg). Manuelle Code-Inspektion (`index.html`) -> `/graphql` Endpunkt gefunden.
3.  **GraphQL Exploitation:** Introspection-Query (`{__schema{...}}`) -> `users`-Query entdeckt. Iteration über IDs (z.B. mit Burp Intruder) für `users(id:§ID§){username,password}` -> Credentials `malo`:`8YdsA3CkiWx968` geleakt.
4.  **Initial Access:** Login via `ssh malo@flossy.hmv` mit geleaktem Passwort.
5.  **Privilege Escalation (malo -> sophie):** `/home/sophie/SSHKeySync`-Skript und zugehörigen Cronjob (impliziert) identifizieren. Eigenes TTY prüfen (`tty`). Auf dem Ziel-TTY (`/dev/pts/24`) auf Schlüssel-Leak warten. Privaten Schlüssel von `sophie` kopieren (`idrsa`). Login via `ssh sophie@flossy.hmv -i idrsa`.
6.  **User Flag:** `cat /home/sophie/user.txt`
7.  **Privilege Escalation (sophie -> root):** `sudo -l` -> `(ALL : ALL) NOPASSWD: /home/sophie/network`. `/home/sophie/network`-Skript analysieren (`disp`-Option kopiert `/opt/*` nach `/etc/NetworkManager/dispatcher.d/` und macht ausführbar). Exploit-Skript erstellen: `echo "chmod u+s /bin/bash" > /opt/SUID_bash`. `sudo /home/sophie/network disp` ausführen. NetworkManager-Dispatcher triggern: `nmcli con up lo`. SUID auf Bash prüfen (`ls -l /bin/bash`).
8.  **Root Access:** Root-Shell mit `bash -p` erhalten.
9.  **Root Flag:** `cat /root/root.txt`

## Flags

*   **User Flag (`/home/sophie/user.txt`):** `8926c8ba832369c1dc13eed7880585c6`
*   **Root Flag (`/root/root.txt`):** `355cec17306ab25389f376ef4a21422e`

---

## Disclaimer

Die hier dargestellten Informationen und Techniken dienen ausschließlich Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Methoden dürfen nur auf Systemen angewendet werden, für die eine ausdrückliche Genehmigung vorliegt (z.B. in CTF-Umgebungen wie HackMyVM, Penetrationstests mit schriftlicher Erlaubnis). Das unbefugte Eindringen in fremde Computersysteme ist illegal und strafbar. Die Autoren übernehmen keine Haftung für missbräuchliche Verwendung der bereitgestellten Informationen. Handeln Sie stets legal und ethisch verantwortlich.

---
