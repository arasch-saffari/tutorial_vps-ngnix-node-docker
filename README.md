# tutorial_vps-ngnix-node-docker


Bitte ersetze "deinedomain.de", "DEINE.IP.ADRESSE", "DEINE.IPv6.ADRESSE" und "DEIN_BENUTZERNAME" durch deine tatsächlichen Informationen.

Schritt 1: Domain-Registrierung
1. Domain kaufen:
   - Besuche Namecheap und erwerbe eine Domain.
   - Anschließend gehe zu "Manage Domains".
   - Wähle "Custom DNS" und trage die DNS-Server von Contabo ein:
     - ns1.contabo.de
     - ns2.contabo.de
     - ns3.contabo.de

		- Für Hetzner sind das folgende DNS Server:
			- hydrogen.ns.hetzner.com.
			- oxygen.ns.hetzner.com.
			- helium.ns.hetzner.de.

:warning: Es kann bis zu 48 Stunden dauern, bis die DNS-Server weltweit aktualisiert sind.

Schritt 2: VPS-Bestellung und DNS-Konfiguration
1. VPS bestellen:
   - Gehe zu Contabo oder Hetzner und bestelle einen VPS.
   - Als Betriebssystem empfehlen ich Debian 12 oder Ubuntu 22.04.

2. DNS konfigurieren:
   - In deinem Contabo-Konto, gehe zu "DNS-Zonen-Verwaltung" und füge deine Domain hinzu.
   - Richte die DNS-Einträge wie folgt ein:

     *.deinedomain.de    86400    A    0    DEINE.IP.ADRESSE
     deinedomain.de      86400    A    0    DEINE.IP.ADRESSE
     www.deinedomain.de  86400    A    0    DEINE.IP.ADRESSE

     *.deinedomain.de    86400    AAAA    0    DEINE.IPv6.ADRESSE
     deinedomain.de      86400    AAAA    0    DEINE.IPv6.ADRESSE
     www.deinedomain.de  86400    AAAA    0    DEINE.IPv6.ADRESSE
     
     - Das "*" ist eine Wildcard, die alle Subdomains umfasst.

3. Reverse DNS einrichten:
   - Gehe zu "Reverse DNS Verwaltung" und trage auch dort deine Domain ein.
   - Beide, die IPV4 und IPV6 Adressen, sollten auf die Domain zeigen.

4. Verbindung testen:
   - Öffne ein Terminal und führe ping deinedomain.endung aus.
   - Die Ausgabe sollte deine IP-Adresse anzeigen.
   - Drücke Strg+C, um den Ping-Prozess zu stoppen.

Schritt 3: Servereinrichtung
1. Verbindung via SSH:
   - Öffne dein Terminal.
   - Verbinde dich mit deinem Server: ssh root@deinedomain.de.
   - Das Passwort hast du entweder per Email zugeschickt bekommen oder musstest es selbstständig eingeben.

2. Neuen Benutzer anlegen:
   - adduser DEIN_BENUTZERNAME
   - Füge ihn zur "sudo"-Gruppe hinzu: usermod -aG sudo DEIN_BENUTZERNAME.

3. IPv6 aktivieren:
   - Bearbeite die Netzwerkkonfiguration: nano /etc/netplan/01-netcfg.yaml.
   - Suche nach dem IPv6-Block und entferne den Kommentar (das # Zeichen).
   - Führe netplan apply aus, um die Änderungen zu übernehmen.

4. Hostname ändern:
   - hostnamectl set-hostname deinedomain.de.

5. Docker installieren:
   - Führe folgende Befehle aus:

     sudo apt update
     sudo apt install apt-transport-https ca-certificates curl software-properties-common
     curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
     sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
     sudo apt update
     sudo apt install docker-ce
     

6. Docker Compose installieren:

   sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
   sudo chmod +x /usr/local/bin/docker-compose
   

7. Docker automatisch starten:
   - sudo systemctl enable docker.

8. Benutzer zur Docker-Gruppe hinzufügen:
   - So musst du nicht jedes Mal "sudo" verwenden: sudo usermod -aG docker DEIN_BENUTZERNAME.

9. Node, npm und nvm installieren:
   - Führe folgende Befehle aus:

     curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
     source ~/.bashrc
     nvm install node
     

10. Nginx installieren:
    - sudo apt update
    - sudo apt install nginx

11. Certbot für SSL-Zertifikate installieren:

    sudo apt install software-properties-common
    sudo add-apt-repository ppa:certbot/certbot
    sudo apt update
    sudo apt install certbot python3-certbot-nginx
    

12. Firewall (UFW) einrichten:

    sudo apt update
    sudo apt install ufw
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw allow ssh
    sudo ufw allow http
    sudo ufw allow https
    sudo ufw enable
    
    - Um weitere Regeln hinzuzufügen oder zu entfernen, verwende sudo ufw allow PORT/tcp und sudo ufw delete allow PORT/tcp.



Schritt 4: Vite.js-Anwendung einrichten
1. Anwendung vorbereiten:
   - Navigiere zum gewünschten Verzeichnis, z.B., /home/DEIN_BENUTZERNAME/apps:

     cd /home/DEIN_BENUTZERNAME/apps
     
   - Erstelle eine neue Vite.js-Anwendung namens "testapp":

     npm init @vitejs/app testapp --template vanilla
     cd testapp
     npm install
     npm run build
     

2. Nginx als Reverse-Proxy konfigurieren:
   - Erstelle eine neue Nginx-Konfigurationsdatei für deine Subdomain:

     sudo nano /etc/nginx/sites-available/testapp.deinedomain.de
     
   - Füge die Konfiguration hinzu:

     server {
         listen 80;
         server_name testapp.deinedomain.de;

         location / {
             proxy_pass http://localhost:3000; # oder den Port, den deine App verwendet
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     
   - Aktiviere den Serverblock:

     sudo ln -s /etc/nginx/sites-available/testapp.deinedomain.de /etc/nginx/sites-enabled/
     
   - Überprüfe Nginx auf Syntaxfehler und starte den Service neu:

     sudo nginx -t
     sudo systemctl restart nginx
     

3. SSL-Zertifikat mit Certbot einrichten:
   - Führe Certbot aus, um ein kostenloses SSL-Zertifikat von Let's Encrypt zu erhalten:

     sudo certbot --nginx -d testapp.deinedomain.de
     
   - Befolge die Anweisungen auf dem Bildschirm, um die SSL-Konfiguration abzuschließen.

Schritt 5: Portainer einrichten
1. Portainer-Container starten:
   - Führe den folgenden Befehl aus, um Portainer zu starten:

     docker run -d -p 8000:8000 -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce
     
   - Öffne Ports in der Firewall:

     sudo ufw allow 8000/tcp
     sudo ufw allow 9000/tcp
     

2. Nginx-Konfiguration für Portainer:
   - Erstelle eine neue Nginx-Konfigurationsdatei:

     sudo nano /etc/nginx/sites-available/portainer.deinedomain.de
     
   - Füge die Konfiguration hinzu:

     server {
         listen 80;
         server_name portainer.deinedomain.de;

         location / {
             proxy_pass http://localhost:9000;
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     
   - Aktiviere den Serverblock:

     sudo ln -s /etc/nginx/sites-available/portainer.deinedomain.de /etc/nginx/sites-enabled/
     
   - Überprüfe Nginx auf Syntaxfehler und starte den Service neu:

     sudo nginx -t
     sudo systemctl restart nginx
     

3. SSL für Portainer:
   - Verwende Certbot, um ein SSL-Zertifikat für die Subdomain zu erhalten:

     sudo certbot --nginx -d portainer.deinedomain.de
     
   - Befolge die Anweisungen auf dem Bildschirm.


Schritt 6: Installation von Appwrite

1. Appwrite-Dienste herunterladen und ausführen:
   - Öffne ein Terminal auf deinem Server.
   - Navigiere zum Verzeichnis, das deine Anwendungen enthält, und erstelle dort einen neuen Ordner für Appwrite:

     cd /home/DEIN_BENUTZERNAME/apps
     mkdir appwrite
     cd appwrite
     
   - Lade den Appwrite Docker-Stack herunter und starte die Dienste:

     docker run -it --rm \
     --volume /var/run/docker.sock:/var/run/docker.sock \
     --volume "$(pwd)"/appwrite:/usr/src/code/appwrite:rw \
     --entrypoint="install" \
     appwrite/appwrite:latest
     

2. Öffnen der erforderlichen UFW-Ports:
   - Appwrite benötigt verschiedene Ports, um ordnungsgemäß zu funktionieren. Öffne die erforderlichen Ports mit den folgenden Befehlen:

     sudo ufw allow 80/tcp      # HTTP
     sudo ufw allow 443/tcp     # HTTPS
     sudo ufw allow 3301/tcp    # Realtime
     sudo ufw allow 5901/tcp    # ClamAV
     # Füge weitere Ports hinzu, je nach deiner Appwrite-Konfiguration
     
   - Aktualisiere die Firewall, um die neuen Regeln zu übernehmen:

     sudo ufw reload
     

3. Nginx als Reverse-Proxy konfigurieren:
   - Erstelle eine neue Nginx-Konfigurationsdatei für deine Subdomain:

     sudo nano /etc/nginx/sites-available/appwrite.deinedomain.de
     
   - Füge die folgende Konfiguration hinzu:

     server {
         listen 80;
         server_name appwrite.deinedomain.de;

         location / {
             proxy_pass http://localhost:80; # Standard-Port für Appwrite
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_set_header X-Forwarded-Proto $scheme;
         }
     }
     
   - Aktiviere den Serverblock, indem du einen symbolischen Link erstellst:

     sudo ln -s /etc/nginx/sites-available/appwrite.deinedomain.de /etc/nginx/sites-enabled/
     
   - Überprüfe Nginx auf Syntaxfehler und starte den Service neu:

     sudo nginx -t
     sudo systemctl restart nginx
     

4. SSL-Zertifikat mit Certbot einrichten:
   - Führe Certbot aus, um ein kostenloses SSL-Zertifikat von Let's Encrypt zu erhalten und automatisch deine Nginx-Konfiguration für HTTPS zu konfigurieren:

     sudo certbot --nginx -d appwrite.deinedomain.de
     
   - Befolge die Anweisungen auf dem Bildschirm, um die Herausforderung zu bestehen und das SSL-Zertifikat zu installieren. Wähle bei Aufforderung die Option, HTTP-Anfragen automatisch auf HTTPS umzuleiten.

5. Appwrite testen:
   - Nach erfolgreicher Installation und Konfiguration kannst du auf die Appwrite-Konsole über deinen Webbrowser zugreifen, indem du zu https://appwrite.deinedomain.de navigierst.

*Schritt 7: Verwendung von PM2 zur Anwendungsverwaltung*

1. *PM2 installieren:*
   - Führe den folgenden Befehl aus, um PM2 über NPM zu installieren:

sh
     npm install pm2 -g
     

2. *Anwendung mit PM2 starten:*
   - Navigiere zum Verzeichnis deiner Vite-Anwendung und führe sie mit PM2 aus:

     cd /home/DEIN_BENUTZERNAME/apps/vite-app-name
     pm2 start npm --name "meine-vite-app" -- start
     
   - Du kannst den Status deiner Anwendung mit dem Befehl pm2 list überprüfen.

3. *PM2 Startup Script:*
   - Um sicherzustellen, dass PM2 beim Booten des Systems startet, verwende:

     pm2 startup
     
   - Führe den Befehl aus, den PM2 vorschlägt, um es als Dienst einzurichten.

4. *Anwendungskonfiguration speichern:*
   - Speichere die aktuelle Anwendungskonfiguration, damit PM2 sie beim Neustart laden kann:

     pm2 save



_______________________






Ich habe den automatisierten Deployment-Prozess mit GitHub Actions konfiguriert und erfolgreich in Betrieb genommen. 
Dies ist ein Verfahren, das die automatische Übertragung und Implementierung von Software-Änderungen auf einen Server ermöglicht, sobald neue Code-Änderungen in das Repository gepusht werden.



_______________________


Hier ist eine detailliertere Anleitung und Erklärung:

1. *Vorbereitung der Secrets:*
   Zuerst speichern wir unsere Server-IP, den Benutzernamen und den SSH-Schlüssel als Secrets in GitHub. Dazu navigierst du in deinem Repository zu "Settings", dann zum Abschnitt "Security" und wählen "Secrets and variables" unter "Actions". Hier erstellen Sie drei Repository-Secrets:
   - SERVER_IP: Tragen Sie hier Ihre Server-IP-Adresse oder Domain ein.
   - SERVER_USERNAME: Dies ist der Username, mit dem Sie sich auf dem Server authentifizieren.
   - SSH_PRIVATE_KEY: Fügen Sie hier den privaten SSH-Schlüssel ein, der auf Ihrem Server verwendet wird.

Achtet darauf, das beim Kopieren euren SSH Keys kein Leerzeichen am ende mit Kopiert wird. Dieser Fehler hat mich gestern zum verzweifeln gebracht.

2. *Einrichten des Workflow:*
   Anschließend gehst du zum Tab "Actions" und wählen "New Workflow". Dadurch wird eine neue Datei namens main.yml im Verzeichnis .github/workflows Ihres Repositories erstellt.

Die main.yml Datei ist das Herzstück Ihres automatisierten Deployments und sieht wie folgt aus:

yaml
name: Deploy Next.js App

on:
  push:
    branches:
      - main

jobs:
  build:
    name: Build and Deploy
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Setup Node.js environment
        uses: actions/setup-node@v2
        with:
          node-version: '20.8.0'

      - name: Install SSH key
        uses: webfactory/ssh-agent@v0.5.4
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy to Server
        run: |
          ssh -t -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${{ secrets.SERVER_USERNAME }}@${{ secrets.SERVER_IP }} <<'EOF'
          cd /home/arasch/apps/nextui #Hier sollte der Pfad zu eurem Ordner auf dem server sein
          export PATH=$PATH:/home/arasch/.nvm/versions/node/v20.8.0/bin 
          #Dieser Befehl war bei mir nötig, weil ich Node via NVM installiert habe. Vielleicht ist der Befehl bei euch auch überflüssig, wenn ihr Node per Apt installiert habt.
          echo $PATH
          npm install
          npm run build
          pm2 reload nextui-app #Hier muss der name eurer pm2 Instanz stehen, den ihr auch unter pm2 List stehen habt.
          exit
          EOF

*Erklärung des Workflows:*

Die Kommentare bitte entfernen, bevor Ihr die Datei ausführt.

- Der Workflow wird ausgelöst, wenn Änderungen (push events) zum main Branch gemacht werden.
- Der Job build wird auf einer Ubuntu-Maschine ausgeführt und durchläuft verschiedene Schritte:
   1. *Checkout code:* Der aktuelle Code des Repositories wird ausgecheckt.
   2. *Setup Node.js environment:* Die Node.js-Umgebung wird mit der spezifizierten Version eingerichtet.
   3. *Install SSH key:* Der private SSH-Schlüssel wird aus den Secrets geholt und für die SSH-Verbindung verwendet.
   4. *Deploy to Server:* Es wird eine SSH-Verbindung zum Server hergestellt und eine Reihe von Befehlen ausgeführt, um die Anwendung zu navigieren, Abhängigkeiten zu installieren, das Projekt zu bauen und die Anwendung mit pm2 neu zu laden.
