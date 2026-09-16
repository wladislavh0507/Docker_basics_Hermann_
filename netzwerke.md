# Docker Compose – Networking (Antworten & Abgabe)

## Schritt 1: Fragen beantworten

### 1) Standardnetzwerk von Docker Compose

Wenn in einer `compose.yaml` kein Netzwerk angegeben ist, erstellt Docker Compose beim Start automatisch ein eigenes Standardnetzwerk für das Projekt. Dieses Netzwerk verwendet normalerweise den Bridge-Treiber und heißt zum Beispiel `meinprojekt_default`. Alle Services der Compose-Datei werden automatisch mit diesem Netzwerk verbunden.

Dadurch sind die Container innerhalb des Projekts voneinander erreichbar, ohne dass für jeden Container manuell ein Netzwerk eingerichtet werden muss. Das Standardnetzwerk ist grundsätzlich auf das jeweilige Compose-Projekt begrenzt.

```yaml
services:
  web:
    image: nginx
  datenbank:
    image: postgres
```

Beim Ausführen von `docker compose up -d` werden beide Services mit demselben automatisch erzeugten Standardnetzwerk verbunden.

### 2) Kommunikation über Servicenamen

Container im gleichen Compose-Netzwerk sollten sich nicht über ihre IP-Adresse, sondern über den in der Compose-Datei festgelegten Servicenamen ansprechen. Docker stellt dafür eine interne DNS-Auflösung bereit.

Im folgenden Beispiel kann der Service `web` die Datenbank unter dem Namen `datenbank` erreichen:

```yaml
services:
  web:
    image: meine-webanwendung
    environment:
      DATABASE_HOST: datenbank
  datenbank:
    image: postgres
```

Die Verbindung würde innerhalb des Docker-Netzwerks beispielsweise zu `datenbank:5432` aufgebaut. Das ist zuverlässiger als die Verwendung einer IP-Adresse, da sich die IP eines Containers nach dem Neuerstellen ändern kann, der Servicename aber gleich bleibt.

### 3) Benutzerdefinierte Netzwerke

Benutzerdefinierte Netzwerke werden eingesetzt, wenn die Netzwerkstruktur genauer gesteuert oder einzelne Bereiche voneinander getrennt werden sollen. Die Netzwerke werden im obersten Abschnitt `networks:` definiert und anschließend den gewünschten Services zugewiesen.

```yaml
services:
  frontend:
    image: nginx
    networks:
      - frontend-netz

  backend:
    image: meine-api
    networks:
      - frontend-netz
      - backend-netz

  datenbank:
    image: postgres
    networks:
      - backend-netz

networks:
  frontend-netz:
    driver: bridge
  backend-netz:
    driver: bridge
```

In diesem Aufbau kann das Backend sowohl mit dem Frontend als auch mit der Datenbank kommunizieren. Frontend und Datenbank teilen dagegen kein gemeinsames Netzwerk und können sich deshalb nicht direkt erreichen. Dadurch lässt sich eine Anwendung sinnvoll aufteilen und besser abschotten.

### 4) Netzwerkmodi `host` und `none`

Mit `network_mode` kann festgelegt werden, wie ein Service das Netzwerk verwendet.

- `host`: Der Container verwendet direkt den Netzwerkbereich des Hosts. Er besitzt dadurch keine getrennte Netzwerkumgebung mit eigener Container-IP. Eine Portweiterleitung über `ports:` ist in diesem Modus nicht erforderlich beziehungsweise nicht wirksam, da der Dienst direkt an einem Port des Hosts lauscht. Dieser Modus sollte bewusst eingesetzt werden, weil die Netzwerkisolation geringer ist.
- `none`: Der Container erhält keine normale Netzwerkverbindung. Er ist damit vollständig vom Netzwerk getrennt und eignet sich für Prozesse, die keine Netzwerkkommunikation benötigen.
- `service:<servicename>`: Der Container verwendet den Netzwerkbereich eines anderen Compose-Services gemeinsam.
- `container:<containername>`: Der Container verwendet den Netzwerkbereich eines bereits vorhandenen Containers gemeinsam.

Beispiel für den Host-Modus:

```yaml
services:
  web:
    image: nginx
    network_mode: host
```

Beispiel für einen Container ohne Netzwerkanbindung:

```yaml
services:
  aufgabe:
    image: alpine
    network_mode: none
```

`network_mode` und eine normale Zuweisung über `networks:` dürfen beim selben Service nicht gleichzeitig verwendet werden.

### 5) Unterschied zwischen `docker compose stop` und `docker compose down`

`docker compose stop` hält die laufenden Container an, entfernt sie aber nicht. Die vorhandenen Container und ihre Konfiguration bleiben bestehen und können später mit `docker compose start` wieder gestartet werden.

```powershell
docker compose stop
docker compose start
```

`docker compose down` beendet die Umgebung und entfernt standardmäßig die zugehörigen Service-Container sowie die von Compose angelegten Netzwerke. Das automatisch erzeugte Standardnetzwerk wird ebenfalls entfernt. Externe Netzwerke werden nicht gelöscht. Volumes werden standardmäßig nicht entfernt; benannte und anonyme Volumes können zusätzlich mit `docker compose down --volumes` gelöscht werden.

```powershell
docker compose down
```

Kurz gesagt: `stop` ist eine Pause der vorhandenen Umgebung, während `down` die Compose-Umgebung weitgehend abbaut.

### 6) Kommunikation zwischen verschiedenen Compose-Projekten

Jedes Compose-Projekt besitzt normalerweise sein eigenes Netzwerk. Deshalb können Container aus unterschiedlichen Projekten nicht automatisch über ihre Servicenamen miteinander kommunizieren. Für eine projektübergreifende Kommunikation kann ein gemeinsames externes Netzwerk verwendet werden.

Zuerst wird das Netzwerk einmalig angelegt:

```powershell
docker network create gemeinsames-netz
```

Anschließend wird es in beiden Compose-Projekten eingebunden:

```yaml
services:
  app:
    image: meine-app
    networks:
      - gemeinsam

networks:
  gemeinsam:
    name: gemeinsames-netz
    external: true
```

Sobald Services beider Projekte mit `gemeinsames-netz` verbunden sind, können sie sich innerhalb dieses Netzwerks über einen eindeutigen Servicenamen oder Netzwerk-Alias erreichen. Das externe Netzwerk wird bei `docker compose down` nicht entfernt.

### 7) Netzwerk-Aliase

Ein Netzwerk-Alias ist ein zusätzlicher DNS-Name für einen Service. Andere Container im gleichen Netzwerk können den Service dann entweder über seinen normalen Servicenamen oder über den Alias ansprechen. Aliase gelten nur in dem Netzwerk, in dem sie eingetragen wurden.

```yaml
services:
  datenbank:
    image: postgres
    networks:
      backend-netz:
        aliases:
          - db
          - postgres-server

networks:
  backend-netz:
```

Die Datenbank ist in diesem Netzwerk damit unter `datenbank`, `db` und `postgres-server` erreichbar. Bei der projektübergreifenden Nutzung eines gemeinsamen Netzwerks sollten Aliase eindeutig gewählt werden, damit nicht mehrere Container denselben Namen verwenden.

### 8) Dynamische und statische IP-Adressen

Docker vergibt Container-IP-Adressen standardmäßig dynamisch aus dem Adressbereich des jeweiligen Netzwerks. Bei einem Neustart oder beim einfachen Stoppen und Starten kann eine Adresse erhalten bleiben, sie ist jedoch nicht als dauerhaft garantiert. Besonders beim Neuerstellen eines Containers kann eine andere IP vergeben werden. Deshalb sollten Anwendungen normalerweise Servicenamen verwenden.

Falls eine Anwendung zwingend eine feste Adresse benötigt, kann im Netzwerk über IPAM ein Subnetz definiert und dem Service eine statische IP-Adresse zugewiesen werden:

```yaml
services:
  server:
    image: nginx
    networks:
      festes-netz:
        ipv4_address: 172.28.0.10

networks:
  festes-netz:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/24
```

Die feste IP muss innerhalb des angegebenen Subnetzes liegen und darf nicht bereits verwendet werden. Statische IP-Adressen sollten nur eingesetzt werden, wenn sie wirklich erforderlich sind, weil Servicenamen flexibler und weniger fehleranfällig sind.

### 9) Unterschied zwischen Host-Port und Container-Port

Bei einer Portzuordnung steht links der Host-Port und rechts der Container-Port:

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
```

- `8080` ist der Host-Port. Über diesen Port wird der Dienst vom Rechner beziehungsweise von außerhalb des Docker-Netzwerks aufgerufen, zum Beispiel mit `localhost:8080`.
- `80` ist der Container-Port. Auf diesem Port lauscht die Anwendung innerhalb des Containers.

Container, die sich im selben Docker-Netzwerk befinden, verwenden normalerweise den Servicenamen und den Container-Port, zum Beispiel `web:80`. Der Host-Port ist dafür nicht notwendig. Er wird nur benötigt, wenn der Dienst vom Host oder von außerhalb des Docker-Netzwerks erreichbar sein soll.

## Zusammenfassung

Docker Compose erleichtert die Kommunikation zwischen Containern durch automatisch erzeugte Netzwerke und eine interne Namensauflösung. Für einfache Projekte reicht das Standardnetzwerk meistens aus. Benutzerdefinierte oder externe Netzwerke sind sinnvoll, wenn Services getrennt, gezielt verbunden oder über mehrere Compose-Projekte hinweg erreichbar gemacht werden sollen. Für stabile Verbindungen sollten Servicenamen oder Aliase anstelle von Container-IP-Adressen verwendet werden. Außerdem ist wichtig, zwischen internen Container-Ports und veröffentlichten Host-Ports sowie zwischen dem Anhalten mit `stop` und dem Abbauen mit `down` zu unterscheiden.

## Quellen

- [Docker Docs: Networking in Compose](https://docs.docker.com/compose/how-tos/networking/)
- [Docker Docs: Netzwerke in der Compose-Datei](https://docs.docker.com/reference/compose-file/networks/)
- [Docker Docs: Services und network_mode](https://docs.docker.com/reference/compose-file/services/#network_mode)
- [Docker Docs: docker compose stop](https://docs.docker.com/reference/cli/docker/compose/stop/)
- [Docker Docs: docker compose down](https://docs.docker.com/reference/cli/docker/compose/down/)
