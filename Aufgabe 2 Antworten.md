# Docker Compose – Networking (Antworten & Abgabe)

**Schritt 2: Fragen beantworten**

1) **Standard‑Netzwerkverhalten ohne explizite Konfiguration**  
   Compose erzeugt beim `docker compose up` automatisch **ein Standardnetzwerk** für das Projekt (benannt nach dem Projekt, z. B. `myapp_default`). **Jeder Dienst** tritt diesem Netzwerk bei und ist **unter seinem Servicenamen per DNS** erreichbar (Service‑zu‑Service über Container‑Ports). 

2) **Benutzerdefiniertes Netzwerk in `docker-compose.yml` anlegen**  
   Mit dem Top‑Level‑Schlüssel `networks:` ein oder mehrere Netzwerke definieren und die Services per `services.<name>.networks` daran anschließen. Beispiel (Bridge‑Treiber, Driver‑Optionen, benutzerdefinierter Name möglich): 
   ```yaml
   services:
     app:
       image: demo
       networks: [frontend, backend]
   networks:
     frontend:
       driver: bridge
     backend:
       driver: bridge
   ```

3) **Netzwerkmodi, die für Dienste gesetzt werden können** (`services.<name>.network_mode`)  
   - `none` – komplette Netzwerktrennung  
   - `host` – direkter Zugriff auf das Host‑Netz (keine Port‑Mappings)  
   - `service:<name>` – Netzwerkstack eines anderen Compose‑Services mitbenutzen  
   - `container:<id|name>` – Netzwerkstack eines bestimmten Containers mitbenutzen  
   Wenn `network_mode` gesetzt ist, darf `networks:` für diesen Dienst **nicht** verwendet werden. 

4) **Wofür eignet sich der `host`‑Modus besonders?**  
   Wenn ein Container **wie ein Prozess direkt auf dem Host** am Netzwerk teilnehmen soll (keine NAT/Port‑Mappings) – z. B. für maximale Netzwerk‑Performance oder Dienste, die **direkt an Host‑Ports binden** müssen. Beachte: Der `host`‑Treiber **funktioniert auf Linux**; in Docker Desktop ist er erst **ab 4.34** als **Opt‑in** verfügbar.

5) **Was passiert mit Netzwerken bei `docker compose down`?**  
   `down` **stoppt** Container und **entfernt**:  
   • die Service‑Container,  
   • **Netzwerke** aus dem `networks:`‑Abschnitt,  
   • sowie das **Default‑Netzwerk**.  
   **Externe** Netzwerke/Volumes (`external: true`) werden **nie** entfernt.

6) **Kommunikation zwischen Containern aus verschiedenen Compose‑Projekten**  
   Ein **benutzerdefiniertes (externes) Netzwerk** einmalig mit `docker network create` anlegen und es in **allen Projekten** per `networks: { name: <netz>, external: true }` einbinden. So hängen die Container projektübergreifend im gleichen Netz und erreichen sich per DNS‑Namen/Alias. 

7) **Rolle von Aliassen in benutzerdefinierten Netzwerken**  
   `aliases` definieren **zusätzliche Hostnamen** für einen Dienst **netzwerk‑spezifisch** (ein Dienst kann auf unterschiedlichen Netzen unterschiedliche Aliasse haben). Andere Container im selben Netz können den Dienst dann unter **Servicename oder Alias** auflösen. 

8) **IP‑Vergabe: dynamisch oder statisch? Wie ändern?**  
   Standardmäßig vergibt Docker **dynamische IP‑Adressen** über die integrierte IPAM‑Verwaltung. Für **statische IPs** kann man am Dienst `ipv4_address`/`ipv6_address` setzen **und** im zugehörigen Netz unter `ipam` ein Subnetz definieren.

---