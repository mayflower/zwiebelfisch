# Zwiebelfisch: Implementierungsplan

## Kontext

Die FiBu erhält Rechnungen in unterschiedlichen Formaten (XRechnung als CII/UBL-XML, ZUGFeRD als PDF/A-3). Kunden können jeweils nur eines der Formate verarbeiten. Ziel ist **Zwiebelfisch** -- ein REST-API-Service, der bidirektional zwischen XRechnung und ZUGFeRD konvertiert und die Eingaben/Ausgaben gegen EN 16931 / XRechnung CIUS validiert. Der Name ist Druckerjargon fuer ein Zeichen aus der falschen Schriftart -- Zwiebelfisch sorgt dafuer, dass Rechnungen im richtigen Format ankommen.

**Anforderungen:**
- Richtung: Beide (XRechnung <-> ZUGFeRD)
- Input-Syntax: CII + UBL
- ZUGFeRD-Profile: Alle (Minimum bis Extended + XRechnung)
- Validierung: XSD + Schematron (EN 16931 + XRechnung CIUS)
- Tech: TypeScript, Node.js, Fastify, REST API
- Projektverzeichnis: `~/Code/zwiebelfisch`

---

## Projektstruktur

```
zwiebelfisch/
├── package.json
├── tsconfig.json
├── .env.example
├── docker-compose.yml              # KoSIT Validator + App (Fallback)
├── Dockerfile                       # Multi-Stage: node Build -> gcr.io/distroless/nodejs22-debian12
├── Tiltfile                        # Lokale K8s-Entwicklung mit Tilt
├── k8s/
│   ├── deployment.yaml             # App Deployment + Service
│   ├── kosit-validator.yaml        # KoSIT Validator Deployment + Service
│   └── config.yaml                 # ConfigMap (Env-Variablen)
├── jest.config.ts
├── src/
│   ├── index.ts                    # Bootstrap Fastify
│   ├── config.ts                   # Env-Konfiguration (validiert via Zod env.schema.ts)
│   ├── app.ts                      # Fastify App Factory (mit Zod Type Provider)
│   ├── routes/
│   │   ├── convert.ts              # POST /convert/xrechnung-to-zugferd
│   │   │                           # POST /convert/zugferd-to-xrechnung
│   │   │                           # Request-Validierung via Zod-Schemas
│   │   ├── validate.ts             # POST /validate
│   │   └── health.ts               # GET /health
│   ├── services/
│   │   ├── detection.service.ts    # Format-Erkennung (CII/UBL/PDF)
│   │   ├── xml-parser.service.ts   # XML -> Invoice-Daten extrahieren
│   │   ├── conversion.service.ts   # Pipeline-Orchestrator
│   │   ├── cii-ubl-transform.service.ts  # CII<->UBL via SaxonJS/XSLT
│   │   ├── zugferd-pdf.service.ts  # PDF/A-3 Erzeugung + XML-Extraktion
│   │   ├── pdf-render.service.ts   # Invoice-Daten -> PDF via pdfmake
│   │   └── validation.service.ts   # XSD + Schematron + KoSIT
│   ├── zod/
│   │   ├── invoice.schema.ts       # Zod-Schemas fuer Invoice-Daten (Typen via z.infer<>)
│   │   ├── conversion.schema.ts    # Zod-Schemas fuer Request/Response DTOs
│   │   ├── validation.schema.ts    # Zod-Schemas fuer Validierungsergebnisse
│   │   └── env.schema.ts           # Zod-Schema fuer Env-Konfiguration
│   ├── templates/
│   │   └── invoice.template.ts     # pdfmake Dokumentdefinition fuer Rechnungslayout
│   ├── standards/
│   │   ├── xsd/                    # EN16931 CII + UBL XSD (vendored)
│   │   └── schematron/             # EN16931 + XRechnung .sch (vendored)
│   ├── xslt/
│   │   ├── cii-to-ubl.sef.json    # Vorkompiliertes XSLT (SaxonJS SEF)
│   │   └── ubl-to-cii.sef.json
│   ├── plugins/
│   │   └── multipart.ts            # Fastify Multipart Config
│   ├── errors/
│   │   ├── conversion.error.ts
│   │   └── validation.error.ts
│   └── utils/
│       ├── xml.utils.ts
│       └── pdf.utils.ts
├── test/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│       ├── cii/                    # Beispiel CII-XML
│       ├── ubl/                    # Beispiel UBL-XML
│       └── zugferd/                # Beispiel ZUGFeRD-PDFs
└── scripts/
    ├── compile-xslt.ts             # XSLT -> SEF kompilieren
    └── download-schemas.ts         # XSD/Schematron herunterladen
```

---

## Architektur: Konvertierungs-Pipeline

### XRechnung -> ZUGFeRD

```
Input (XML)
  │
  ▼
[1. Format-Erkennung] ── CII oder UBL?
  │
  ▼
[2. Validierung] ── XSD + Schematron (optional)
  │
  ▼
[3. UBL->CII Transform] ── nur bei UBL-Input (SaxonJS + XSLT)
  │
  ▼
[4. Profil-Mapping] ── Daten fuer Ziel-ZUGFeRD-Profil pruefen
  │
  ▼
[5. PDF rendern] ── Invoice-Daten -> PDF via pdfmake
  │
  ▼
[6. PDF/A-3 erzeugen] ── CII-XML in PDF einbetten (@stafyniaksacha/facturx)
  │
  ▼
Output (PDF/A-3)
```

### ZUGFeRD -> XRechnung

```
Input (PDF)
  │
  ▼
[1. XML extrahieren] ── aus PDF/A-3 (@stafyniaksacha/facturx)
  │
  ▼
[2. Profil erkennen] ── GuidelineSpecifiedDocumentContextParameter auswerten
  │
  ▼
[3. Validierung] ── extrahiertes CII-XML pruefen (optional)
  │
  ▼
[4. CII->UBL Transform] ── nur bei UBL-Output (SaxonJS + XSLT)
  │
  ▼
[5. XRechnung-Anreicherung] ── Leitweg-ID (BT-10), BT-23, BT-34, BT-49 pruefen
  │
  ▼
[6. Output-Validierung] ── gegen XRechnung CIUS Schematron
  │
  ▼
Output (XML, CII oder UBL)
```

### Architektur-Entscheidung: Kein internes Datenmodell in V1

Statt alle ~150 Business Terms in ein TypeScript-Objektmodell zu mappen, arbeitet V1 **direkt auf XML**. CII<->UBL-Transformation erfolgt via XSLT. Das ist der Ansatz der offiziellen EU-Referenzimplementierungen. Ein internes Modell kann spaeter ergaenzt werden.

---

## Bibliotheken

| Zweck | Bibliothek | Begruendung |
|---|---|---|
| HTTP Framework | **Fastify** v5 | Schnell, TypeScript-first, Schema-Validierung, `@fastify/multipart` |
| Request/Config-Validierung | **Zod** + **fastify-type-provider-zod** | Typsichere Request-Validierung, Env-Validierung, automatische OpenAPI-Schema-Generierung aus Zod-Schemas |
| XML aus PDF extrahieren + PDF/A-3 erzeugen | **@stafyniaksacha/facturx** | Einzige Node-Lib mit `extract()` und `generate()` fuer ZUGFeRD |
| ZUGFeRD-Profil-Typen | **node-zugferd** | TypeScript-Typdefinitionen fuer alle 6 Profile |
| CII <-> UBL Transformation | **saxon-js** + vorkompiliertes XSLT (SEF) | Einzige produktionsreife XSLT 3.0 Engine fuer Node.js |
| XML Parsing | **fast-xml-parser** | Schnellster reiner JS XML-Parser |
| Schematron-Validierung | **node-schematron** | Reines JS, XPath 3.1, kein Java noetig |
| XSD + Vollvalidierung | **KoSIT Validator** (Docker Sidecar) | Offizieller deutscher Validator, umfassend |
| PDF-Rendering | **pdfmake** | Deklarative Dokumentdefinition (JSON), Tabellen/Spalten/Kopf-/Fusszeilen eingebaut, kein Browser noetig, ~5MB statt ~400MB |

---

## API-Endpunkte

### `POST /convert/xrechnung-to-zugferd`

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `xml` | file | ja | XRechnung XML (CII oder UBL) |
| `profile` | string | nein | Ziel-Profil. Default: `EN16931`. Optionen: `MINIMUM`, `BASIC_WL`, `BASIC`, `EN16931`, `EXTENDED`, `XRECHNUNG` |
| `validate` | boolean | nein | Eingabe validieren. Default: `true` |

**Response (Erfolg):** `application/pdf` (PDF/A-3 Binary)

**Response (Validierungsfehler, 422):**
```json
{
  "error": "VALIDATION_FAILED",
  "details": {
    "xsd": [{ "line": 42, "message": "Element 'ram:BuyerReference' is missing" }],
    "schematron": [{ "rule": "BR-DE-15", "message": "Leitweg-ID is required" }]
  }
}
```

### `POST /convert/zugferd-to-xrechnung`

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `pdf` | file | ja | ZUGFeRD PDF/A-3 |
| `outputSyntax` | string | nein | `CII` oder `UBL`. Default: `CII` |
| `validate` | boolean | nein | Output validieren. Default: `true` |

**Response (Erfolg):** `application/xml` (XRechnung XML)

### `POST /validate`

| Feld | Typ | Pflicht | Beschreibung |
|---|---|---|---|
| `file` | file | ja | XML oder PDF |
| `standard` | string | nein | `EN16931`, `XRECHNUNG`. Default: auto-detect |

**Response:**
```json
{
  "valid": true,
  "format": { "syntax": "CII", "standard": "XRechnung", "version": "3.0" },
  "profile": "EN16931",
  "errors": [],
  "warnings": []
}
```

### `GET /health`

Health-Check inkl. KoSIT-Validator-Erreichbarkeit.

---

## Validierungsstrategie: Zwei Ebenen

### Ebene 1: API-Validierung (Zod)

Request-Parameter und Konfiguration werden beim Eingang via Zod validiert:
- Multipart-Felder (`profile`, `outputSyntax`, `validate`, `standard`) gegen Zod-Enums
- Env-Variablen beim App-Start gegen `env.schema.ts`
- Response-Shapes fuer konsistente API-Antworten

Typen werden direkt aus den Zod-Schemas abgeleitet (`z.infer<>`), kein manuelles Doppeln.

### Ebene 2: Fachliche Rechnungsvalidierung (XSD + Schematron)

1. **Strukturell (XSD):** via KoSIT Validator Docker Sidecar (HTTP POST)
2. **Geschaeftsregeln (EN16931 Schematron):** via `node-schematron` in-process + KoSIT als Fallback
3. **XRechnung CIUS:** via `node-schematron` mit offiziellen KoSIT-Schematron-Regeln

**Primaerer Validierungspfad:** KoSIT Validator (umfassend, autoritativ). `node-schematron` als schnelles In-Process-Feedback.

---

## PDF-Erzeugung

Fuer die Richtung XRechnung->ZUGFeRD wird ein visuelles PDF benoetigt:

1. CII-XML parsen -> Business Terms extrahieren
2. pdfmake-Dokumentdefinition (`invoice.template.ts`) befuellen mit deutschem Rechnungslayout:
   - Kopf: Verkaeufer, Rechnungsnr., Datum
   - Empfaenger-Block
   - Positionstabelle: Artikelbezeichnung, Menge, Einzelpreis, MwSt., Gesamtpreis
   - Summen-Block: Netto, MwSt.-Aufschluesselung, Brutto, Faelliger Betrag
   - Zahlungsinformationen: IBAN/BIC, Verwendungszweck
   - Fusszeile: Steuernummer, Handelsregister (Pflichtangaben nach UStG)
3. pdfmake rendert Dokumentdefinition -> PDF-Buffer (in-memory, kein Browser)
4. `@stafyniaksacha/facturx` wandelt PDF -> PDF/A-3 und bettet XML ein

**Warum pdfmake statt Puppeteer:** Rechnungen sind strukturierte Geschaeftsdokumente (Tabellen, Spalten, Kopf-/Fusszeilen) -- genau pdfmakes Kernkompetenz. Kein ~400MB Chrome-Binary im Docker-Image, kein Browser-Pool fuer Parallelitaet, Rendering in Millisekunden statt Sekunden.

---

## Fehlerbehandlung

Eigene Error-Hierarchie:

- `EInvoiceError` (Basis, mit `code`, `statusCode`, `details`)
  - `FormatDetectionError` (400) -- unbekanntes Format
  - `ValidationError` (422) -- Validierung fehlgeschlagen, mit strukturierten Fehlern
  - `ExtractionError` (422) -- kein XML in PDF gefunden
  - `TransformationError` (500) -- XSLT-Fehler
  - `PdfRenderError` (500) -- pdfmake-Fehler

**Wichtiger Edge Case:** ZUGFeRD Minimum/Basic WL Profile enthalten keine Positionsdaten. Konvertierung zu XRechnung (erfordert Positionen, BR-16) ist nicht moeglich -> klare Fehlermeldung.

---

## Test-Strategie

### Testdaten
- **KoSIT xrechnung-testsuite** -- offizielle valide/invalide CII- und UBL-Beispiele
- **Mustang-Projekt** -- ZUGFeRD-Beispiel-PDFs aller Profile
- **Eigene Fixtures** -- Edge Cases (mehrere MwSt.-Saetze, Gutschriften, etc.)

### Unit Tests (Jest)
- `detection.service` -- CII/UBL/PDF korrekt erkennen, fehlerhafte Eingaben abfangen
- `xml-parser.service` -- Business Terms aus CII und UBL extrahieren
- `cii-ubl-transform.service` -- Roundtrip CII->UBL->CII, Datenvollstaendigkeit
- `validation.service` -- Valide Dokumente bestehen, invalide liefern korrekte Fehler
- `zugferd-pdf.service` -- XML-Extraktion aus PDFs, eingebettetes XML in erzeugten PDFs

### Integration Tests
- Vollstaendiger Roundtrip: CII-XML -> ZUGFeRD-PDF -> XML-Extraktion -> Vergleich
- UBL-Input -> ZUGFeRD -> UBL-Output -> Semantische Aequivalenz
- Jedes ZUGFeRD-Profil einzeln testen
- Validierungs-Endpunkt mit invaliden Dokumenten

### Infrastruktur
- `docker-compose` fuer KoSIT-Validator in Tests
- Fastify `inject()` fuer HTTP-Tests ohne echten Server
- KoSIT-Mock in Unit Tests

---

## Phasen-Plan

### Phase 1: Fundament (Woche 1-2)
- Projekt initialisieren (npm, TypeScript, Fastify, Zod, Docker)
- Fastify App Factory mit `fastify-type-provider-zod` einrichten
- Zod-Schemas anlegen (`env.schema.ts`, `conversion.schema.ts`, `validation.schema.ts`)
- Dockerfile erstellen (Multi-Stage Build, Distroless Runtime)
- K8s-Manifeste (`k8s/`) + Tiltfile fuer lokale Entwicklung auf Minikube
- Docker Compose als Fallback / CI-Umgebung
- `detection.service.ts` -- Format-Erkennung
- `xml-parser.service.ts` -- CII-XML parsen
- KoSIT-Validator als K8s-Deployment
- `validation.service.ts` -- KoSIT-HTTP-Integration
- `GET /health` + `POST /validate` Endpunkte (Request-Validierung via Zod)
- Unit Tests fuer Detection und Parsing
- Testdaten beschaffen

**Ergebnis:** Service laeuft auf Minikube via `tilt up`, akzeptiert XML, erkennt Format, validiert via KoSIT.

### Phase 2: ZUGFeRD PDF-Operationen (Woche 3)
- `@stafyniaksacha/facturx` integrieren (`extract()` + `generate()`)
- `zugferd-pdf.service.ts`
- `pdf-render.service.ts` -- pdfmake Integration
- `invoice.template.ts` -- pdfmake Dokumentdefinition fuer Rechnungslayout
- `POST /convert/xrechnung-to-zugferd` (nur CII-Input)
- `POST /convert/zugferd-to-xrechnung` (nur CII-Output)
- Integrationstests mit echten ZUGFeRD-PDFs

**Ergebnis:** CII-zu-ZUGFeRD und ZUGFeRD-zu-CII funktionieren.

### Phase 3: CII <-> UBL Transformation (Woche 4-5)
- SaxonJS + `xslt3` Build-Pipeline
- CII-to-UBL XSLT (SFTI Repository) -> SEF kompilieren
- UBL-to-CII XSLT schreiben (basierend auf EN 16931 Mapping-Tabelle) -> SEF
- `cii-ubl-transform.service.ts`
- Konvertierungs-Endpunkte um UBL erweitern
- Roundtrip-Tests

**Ergebnis:** Volle bidirektionale Konvertierung (CII/UBL <-> ZUGFeRD).

### Phase 4: Profil-Support + Schematron (Woche 6)
- ZUGFeRD-Profil-Erkennung und -Mapping
- Profilspezifische Validierung
- `node-schematron` Integration fuer In-Process EN16931 + XRechnung CIUS
- Offizielle Schematron-Dateien vendoren
- Profil-Auswahl im Konvertierungs-Endpunkt
- Tests aller 6 Profile

**Ergebnis:** Profilbewusste Konvertierung mit dualer Validierung.

### Phase 5: Haertung + Produktionsreife (Woche 7-8)
- Request-Logging (pino)
- OpenAPI/Swagger Docs (`@fastify/swagger` -- Schemas werden automatisch aus Zod generiert via `fastify-type-provider-zod`)
- Rate Limiting
- Graceful Shutdown
- CI/CD Pipeline (GitHub Actions)
- Lasttests
- Dokumentation

**Ergebnis:** Produktionsfaehiges Docker-Image, bereit fuer K8s-Deployment.

---

## Dockerfile: Multi-Stage Distroless Build

```dockerfile
# Build Stage
FROM node:22-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production Stage
FROM gcr.io/distroless/nodejs22-debian12
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/package.json ./
EXPOSE 3000
CMD ["dist/index.js"]
```

**Warum Distroless:**
- ~50MB Runtime-Image statt ~300MB+ mit `node:22-slim`
- Keine Shell, kein Paketmanager, keine System-Utilities -> minimale Angriffsflaeche
- Moeglich durch pdfmake-Entscheidung: keine nativen System-Dependencies (Puppeteer/Chrome haette Distroless ausgeschlossen)
- Debugging via `kubectl logs` + strukturiertes Logging (pino), nicht via `kubectl exec`

---

## Docker Compose (Fallback / CI)

```yaml
version: '3.8'
services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - KOSIT_VALIDATOR_URL=http://kosit-validator:8080
      - NODE_ENV=production
    depends_on:
      - kosit-validator

  kosit-validator:
    image: flexness/xrechnung-validator-docker:latest
    ports:
      - "8080:8080"
```

---

## Lokale Entwicklung: Tilt + Minikube

Primaere Entwicklungsumgebung laeuft auf lokalem K8s (Minikube), gesteuert durch Tilt. Das stellt sicher, dass die Entwicklungsumgebung dem Produktions-Deployment auf K8s entspricht.

### Voraussetzungen
- **Minikube** installiert und gestartet (`minikube start`)
- **Tilt** installiert (https://tilt.dev)
- **kubectl** konfiguriert gegen Minikube-Kontext

### K8s-Manifeste (`k8s/`)

**`k8s/deployment.yaml`** -- App Deployment + Service:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zwiebelfisch
  labels:
    app: zwiebelfisch
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zwiebelfisch
  template:
    metadata:
      labels:
        app: zwiebelfisch
    spec:
      containers:
        - name: api
          image: zwiebelfisch
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: zwiebelfisch-config
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 30
---
apiVersion: v1
kind: Service
metadata:
  name: zwiebelfisch
spec:
  selector:
    app: zwiebelfisch
  ports:
    - port: 3000
      targetPort: 3000
```

**`k8s/kosit-validator.yaml`** -- KoSIT Sidecar:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kosit-validator
  labels:
    app: kosit-validator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kosit-validator
  template:
    metadata:
      labels:
        app: kosit-validator
    spec:
      containers:
        - name: kosit-validator
          image: flexness/xrechnung-validator-docker:latest
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: kosit-validator
spec:
  selector:
    app: kosit-validator
  ports:
    - port: 8080
      targetPort: 8080
```

**`k8s/config.yaml`** -- ConfigMap:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: zwiebelfisch-config
data:
  KOSIT_VALIDATOR_URL: "http://kosit-validator:8080"
  NODE_ENV: "development"
  LOG_LEVEL: "debug"
```

### Tiltfile

```python
# -*- mode: Python -*-

# App-Image bauen mit Live-Update (Rebuild bei Code-Aenderungen)
docker_build(
    'zwiebelfisch',
    '.',
    dockerfile='Dockerfile',
    live_update=[
        sync('./src', '/app/src'),
        run('npm run build', trigger=['./src']),
    ],
)

# K8s-Manifeste laden
k8s_yaml([
    'k8s/config.yaml',
    'k8s/kosit-validator.yaml',
    'k8s/deployment.yaml',
])

# Port-Forwarding fuer lokalen Zugriff
k8s_resource('zwiebelfisch', port_forwards='3000:3000')
k8s_resource('kosit-validator', port_forwards='8080:8080')
```

### Workflow

```bash
minikube start
tilt up           # Startet alles, oeffnet Tilt UI im Browser
# Tilt beobachtet Datei-Aenderungen und rebuilt/redeployed automatisch
tilt down         # Raeumt auf
```

---

## Verifikation

1. **Unit Tests:** `npm test` -- alle Services einzeln getestet
2. **Integration Tests:** `npm run test:integration` -- mit KoSIT-Docker-Container
3. **Manueller Test:**
   - XRechnung CII-XML hochladen -> ZUGFeRD-PDF zurueckbekommen -> in PDF-Viewer oeffnen + XML-Anhang pruefen
   - ZUGFeRD-PDF hochladen -> XRechnung-XML zurueckbekommen -> gegen KoSIT validieren
4. **Roundtrip-Test:** XML -> PDF -> XML -> inhaltlicher Vergleich
5. **Validierungstest:** Invalides XML hochladen -> strukturierte Fehlermeldung pruefen
