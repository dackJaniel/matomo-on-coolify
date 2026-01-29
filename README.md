# Matomo auf Coolify deployen

Dieses Repository enthält ein fertig konfiguriertes Docker-Compose-Setup, um [Matomo](https://matomo.org/) als Self-Hosted-Service auf einem [Coolify](https://coolify.io/)-Server zu deployen.

## Was wird deployed?

| Service | Image | Aufgabe |
|---------|-------|---------|
| `db` | `mariadb:lts` | MariaDB-Datenbank, wird beim ersten Start automatisch erstellt und konfiguriert |
| `matomo` | `matomo:latest` | Matomo-Webapplikation (Apache + PHP) |

Beide Services nutzen persistente Docker Volumes, sodass Daten bei Neustarts oder Updates erhalten bleiben.

## Voraussetzungen

- Ein laufender Coolify-Server
- Eine Domain (oder Subdomain), die auf den Coolify-Server zeigt (z.B. `analytics.example.com`)

## Deployment-Anleitung

### 1. Service in Coolify anlegen

1. Im Coolify-Dashboard auf **Add Resource** klicken
2. **Docker Compose** als Typ auswaehlen
3. Entweder den Inhalt der `docker-compose.yml` direkt einfuegen oder dieses Repository als Git-Quelle verlinken

### 2. Environment Variables setzen

In Coolify unter **Environment Variables** die folgenden Werte eintragen:

#### Pflichtfelder

| Variable | Beschreibung | Beispiel |
|----------|-------------|---------|
| `MARIADB_PASSWORD` | Passwort fuer den Datenbank-Benutzer | `ein-sicheres-passwort` |
| `MARIADB_ROOT_PASSWORD` | Root-Passwort fuer MariaDB | `ein-anderes-sicheres-passwort` |

#### Optionale Felder

Diese haben bereits sinnvolle Standardwerte und muessen nur geaendert werden, wenn gewuenscht:

| Variable | Default | Beschreibung |
|----------|---------|-------------|
| `MARIADB_DATABASE` | `matomo` | Name der Datenbank |
| `MARIADB_USER` | `matomo` | Datenbank-Benutzername |
| `MATOMO_DATABASE_TABLES_PREFIX` | `matomo_` | Praefix fuer alle Datenbank-Tabellen |
| `PHP_MEMORY_LIMIT` | `256M` | PHP Memory Limit fuer Matomo |

### 3. Domain zuweisen

1. In den Service-Einstellungen den Container `matomo` als oeffentlichen Service markieren
2. Die gewuenschte Domain eintragen (z.B. `analytics.example.com`)
3. Coolify uebernimmt SSL-Zertifikate und Reverse-Proxy (Traefik) automatisch

### 4. Service starten

Den Service ueber das Coolify-Dashboard deployen. Beim Start passiert folgendes:

1. **MariaDB** startet und erstellt automatisch die Datenbank `matomo` mit dem konfigurierten Benutzer
2. **Matomo** wartet per Healthcheck, bis die Datenbank bereit ist
3. Matomo verbindet sich automatisch mit der Datenbank -- keine manuelle DB-Konfiguration noetig

### 5. Matomo einrichten

Beim ersten Aufruf der Domain erscheint der **Matomo-Setup-Wizard**. Die Datenbank-Verbindung ist bereits vorkonfiguriert. Im Wizard muss nur noch:

1. Der **Systemcheck** bestaetigt werden
2. Der Datenbank-Schritt uebersprungen werden (bereits konfiguriert)
3. Ein **Admin-Account** angelegt werden (Benutzername, Passwort, E-Mail)
4. Die erste **Website** zum Tracking hinzugefuegt werden
5. Der **Tracking-Code** kopiert und in die Zielwebsite eingebaut werden

## Projektstruktur

```
matomo-coolify/
├── docker-compose.yml   # Docker Compose Konfiguration fuer Coolify
├── .env.example         # Vorlage fuer Environment Variables
└── README.md            # Diese Datei
```

## Technische Details

### Healthchecks

Beide Services haben Healthchecks konfiguriert:

- **MariaDB:** Prueft, ob die Datenbank verbunden und InnoDB initialisiert ist
- **Matomo:** Prueft per HTTP-Request, ob der Webserver antwortet

Matomo startet erst, wenn MariaDB als `healthy` gemeldet wird (`depends_on` mit `condition: service_healthy`).

### Volumes

| Volume | Pfad im Container | Inhalt |
|--------|-------------------|--------|
| `db-data` | `/var/lib/mysql` | Datenbankdateien |
| `matomo-data` | `/var/www/html` | Matomo-Applikation, Konfiguration, Plugins |

### Kein Port-Mapping

Die `docker-compose.yml` enthaelt bewusst keine `ports`-Eintraege. Coolify routet den Traffic ueber seinen integrierten Reverse-Proxy (Traefik) und kuemmert sich automatisch um SSL-Terminierung und Domain-Routing.

### MariaDB-Konfiguration

- `--max-allowed-packet=64MB`: Erlaubt groessere Datenpakete, wichtig fuer umfangreiche Matomo-Reports
- `MARIADB_AUTO_UPGRADE=1`: Fuehrt bei Image-Updates automatisch Datenbank-Upgrades durch
- `MARIADB_INITDB_SKIP_TZINFO=1`: Ueberspringt den Zeitzonen-Import beim Start (beschleunigt den ersten Start)

## Updates

Matomo aktualisieren:

1. In Coolify den Service neu deployen (Pull latest images)
2. Matomo zeigt nach dem Start ggf. einen Datenbank-Update-Hinweis im Dashboard an -- diesen bestaetigen
3. MariaDB fuehrt Schema-Upgrades dank `MARIADB_AUTO_UPGRADE=1` automatisch durch

## Referenzen

- [Matomo Docker Image (Docker Hub)](https://hub.docker.com/_/matomo/)
- [Matomo Docker Repository (GitHub)](https://github.com/matomo-org/docker)
- [Matomo Self-Hosted Installationsanleitung](https://matomo.org/guide/installation-maintenance/matomo-on-premise-self-hosted/)
- [Coolify Dokumentation](https://coolify.io/docs)
