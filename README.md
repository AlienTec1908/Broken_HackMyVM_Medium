# Broken - HackMyVM Writeup

![Broken VM Icon](Broken.png)

Dieses Repository enthält das Writeup für die HackMyVM-Maschine "Broken" (Schwierigkeitsgrad: Medium), erstellt von DarkSpirit. Ziel war es, initialen Zugriff auf die virtuelle Maschine zu erlangen und die Berechtigungen bis zum Root-Benutzer zu eskalieren.

## VM-Informationen

*   **VM Name:** Broken
*   **Plattform:** HackMyVM
*   **Autor der VM:** DarkSpirit
*   **Schwierigkeitsgrad:** Medium
*   **Link zur VM:** [https://hackmyvm.eu/machines/machine.php?vm=Broken](https://hackmyvm.eu/machines/machine.php?vm=Broken)

## Writeup-Informationen

*   **Autor des Writeups:** Ben C.
*   **Datum des Berichts:** 25. Mai 2024
*   **Link zum Original-Writeup (GitHub Pages):** [https://alientec1908.github.io/Broken_HackMyVM_Medium/](https://alientec1908.github.io/Broken_HackMyVM_Medium/)

## Kurzübersicht des Angriffspfads

Der Angriff auf die Broken-Maschine umfasste die folgenden Schritte:

1.  **Reconnaissance:**
    *   Identifizierung der Ziel-IP (`192.168.2.113` unter dem Hostnamen `broken.hmv`) mittels `arp-scan`.
    *   Ein `nmap`-Scan offenbarte einen offenen Port: HTTP (80, Apache 2.4.38).
2.  **Web Enumeration:**
    *   `gobuster` fand `index.php`, `/textpattern/` und `file.php`.
    *   `wfuzz` identifizierte den Parameter `file` in `file.php` als anfällig für Local File Inclusion (LFI).
    *   Mittels LFI (`file.php?file=../../../../etc/passwd`) wurde die `/etc/passwd`-Datei ausgelesen, was den Benutzer `heart` offenbarte.
    *   Ein Versuch, den SSH-Schlüssel von `heart` via LFI zu lesen, gab stattdessen den Quellcode von `file.php` aus, der einen (unzureichenden) `strpos` Filter für `..` zeigte.
    *   Ein `hydra`-Brute-Force-Angriff auf das Login-Formular von Textpattern (`/textpattern/textpattern/index.php`) war erfolgreich und ergab die Zugangsdaten `admin:angel`.
3.  **Initial Access (via LFI & Log Poisoning / File Upload):**
    *   **Log Poisoning:** Ein bösartiger User-Agent mit PHP-Code (``) wurde an den Server gesendet. Anschließend wurde die Nginx-Logdatei (`/var/log/nginx/access.log`) über die LFI in `file.php` inkludiert und der `cmd`-Parameter verwendet, um RCE als `www-data` zu erlangen.
    *   **Alternative (impliziert):** Nach dem Login in Textpattern wurde eine PHP-Webshell (`ben.php`) hochgeladen (vermutlich nach `/tmp` oder `content/files`). Diese Shell wurde dann über die LFI in `file.php` ausgeführt, ebenfalls mit einem `cmd`-Parameter, um RCE zu erlangen.
    *   Eine Netcat-Reverse-Shell wurde initiiert, was zu einer Shell als `www-data` führte.
4.  **Privilege Escalation (www-data zu heart):**
    *   `sudo -l` für `www-data` zeigte: `(heart) NOPASSWD: /usr/bin/pydoc3.7`.
    *   `pydoc3.7` wurde mit `sudo -u heart` gestartet. Innerhalb von `pydoc` wurde mit `!/bin/sh` eine Shell als Benutzer `heart` erlangt.
5.  **Privilege Escalation (heart zu root via `sudo patch`):**
    *   Als `heart` wurde der private SSH-Schlüssel (`/home/heart/.ssh/id_rsa`) ausgelesen und für einen direkten SSH-Zugang verwendet.
    *   `sudo -l` für `heart` (nicht explizit im Log gezeigt, aber für den Exploit notwendig) erlaubte vermutlich die Ausführung von `/usr/bin/patch` ohne Passwort.
    *   Die Datei `/etc/passwd` wurde kopiert.
    *   Ein neuer Benutzer `myroot` mit UID/GID 0 und einem bekannten Passwort-Hash (für das Passwort `benni`) wurde zur Kopie hinzugefügt.
    *   Eine Patch-Datei (`passwd.patch`) wurde mit `diff` erstellt.
    *   Der Patch wurde mit `sudo patch -b /etc/passwd < passwd.patch` auf die Systemdatei `/etc/passwd` angewendet.
    *   Mit `su myroot` und dem Passwort `benni` wurde Root-Zugriff erlangt.
6.  **Flags:**
    *   Die User-Flag (`/home/heart/user.txt`) wurde als `heart` gelesen.
    *   Die Root-Flag (`/root/r0otfl4g.sh`) wurde als `root` gelesen.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `wfuzz`
*   `curl`
*   `hydra`
*   `nc` (netcat)
*   `pydoc3.7`
*   `mkpasswd`
*   `nano`
*   `diff`
*   `patch`
*   `su`
*   `cat`
*   `chmod`
*   `ssh`

## Identifizierte Schwachstellen (Zusammenfassung)

*   **Local File Inclusion (LFI):** Der `file`-Parameter in `file.php` war anfällig für LFI, trotz eines unzureichenden `strpos`-Filters.
*   **Schwaches CMS-Passwort:** Das Passwort für den `admin`-Benutzer des Textpattern-CMS (`angel`) konnte mit `rockyou.txt` gebruteforced werden.
*   **Unsicherer Dateiupload im CMS (Impliziert):** Ermöglichte das Hochladen einer PHP-Webshell.
*   **Log Poisoning:** In Kombination mit LFI ermöglichte das Schreiben von PHP-Code in die Nginx `access.log` (via User-Agent) Remote Code Execution.
*   **Unsichere `sudo`-Konfiguration (www-data zu heart):** `www-data` konnte `/usr/bin/pydoc3.7` als `heart` ohne Passwort ausführen, was einen Shell-Escape ermöglichte.
*   **Unsichere `sudo`-Konfiguration (heart zu root):** `heart` konnte `/usr/bin/patch` ohne Passwort (impliziert) ausführen, was die Manipulation von `/etc/passwd` und somit die Erstellung eines neuen Root-Benutzers ermöglichte.

## Flags

*   **User Flag (`/home/heart/user.txt`):** `HMVPatchtowin`
*   **Root Flag (`/root/r0otfl4g.sh`):** `HMVPatchedyeah`

---

**Wichtiger Hinweis:** Dieses Dokument und die darin enthaltenen Informationen dienen ausschließlich zu Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die beschriebenen Techniken und Werkzeuge sollten nur in legalen und autorisierten Umgebungen (z.B. eigenen Testlaboren, CTFs oder mit expliziter Genehmigung) angewendet werden. Das unbefugte Eindringen in fremde Computersysteme ist eine Straftat und kann rechtliche Konsequenzen nach sich ziehen.

---
*Bericht von Ben C. - Cyber Security Reports*
