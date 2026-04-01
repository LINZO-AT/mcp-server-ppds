# PP/DS MCP Server – Projektdokumentation

## Ziel

MCP-Server der PP/DS-relevante OData APIs aus einem S/4HANA Private Cloud (PCE) System exposed — als Grundlage für einen AI-gestützten Planungs-Chatbot via SAP AI Core.

---

## Architektur

```
S/4HANA PCE
    ↓ Cloud Connector
BTP Destination Service (S4H_PCE)
    ↓
PP/DS MCP Server (BTP Cloud Foundry)
  └─ odata-mcp-proxy npm package
  └─ api-config.json (PP/DS EntitySets)
    ↓
AI Core Orchestration Service
    ↓
Custom App / Chatbot
```

**Basis:** [`odata-mcp-proxy`](https://github.com/lemaiwo/odata-mcp-proxy) – kein eigener Code, nur JSON-Konfiguration.

---

## Phase 1 – Lokal testen (ohne BTP)

Ziel: MCP-Server lokal starten, mit Claude Desktop verbinden und PP/DS-Fragen testen.

### Voraussetzungen
- Node.js 20+
- Claude Desktop installiert
- Zugang zum S/4HANA PCE System (VPN o.ä.)

### Setup

```bash
git clone https://github.com/lemaiwo/odata-mcp-proxy
cd odata-mcp-proxy
npm install
```

`.env` Datei anlegen (für lokale Credentials ohne BTP Destination Service):

```env
S4H_PCE_URL=https://<your-s4h-host>/
S4H_PCE_USERNAME=<user>
S4H_PCE_PASSWORD=<password>
```

> Naming Convention: `<DESTINATION_NAME>_URL`, `<DESTINATION_NAME>_USERNAME`, etc.

`api-config.json` → siehe Abschnitt "API Konfiguration"

Server starten:
```bash
npm start
```

### Claude Desktop verbinden

`~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):

```json
{
  "mcpServers": {
    "ppds": {
      "command": "node",
      "args": ["/path/to/odata-mcp-proxy/src/index.js", "--transport", "stdio"],
      "env": {
        "S4H_PCE_URL": "https://<your-s4h-host>/",
        "S4H_PCE_USERNAME": "<user>",
        "S4H_PCE_PASSWORD": "<password>"
      }
    }
  }
}
```

Claude Desktop neu starten → MCP-Tools erscheinen automatisch.

**Testfragen:**
- *"Welche Planaufträge für Werk 1000 laufen diese Woche?"*
- *"Zeig mir alle offenen Fertigungsaufträge für Material XYZ"*
- *"Welche Komponenten hat Planauftrag 1234567?"*

---

## Phase 2 – BTP Deployment

### Voraussetzungen
- SAP BTP Subaccount (Cloud Foundry aktiviert)
- Cloud Connector konfiguriert mit Virtual Host für S/4HANA PCE
- BTP Destination angelegt (Name: `S4H_PCE`)
- Services: Destination Service, XSUAA (Authorization & Trust), optional Connectivity Service

### BTP Destination konfigurieren

| Property | Wert |
|---|---|
| Name | `S4H_PCE` |
| Type | `HTTP` |
| URL | Virtual Host des Cloud Connectors |
| Authentication | `BasicAuthentication` oder `OAuth2SAMLBearerAssertion` |
| Proxy Type | `OnPremise` |

> Für PCE empfohlen: `OAuth2SAMLBearerAssertion` (SSO-fähig, kein Passwort im Destination)

### Projektstruktur

```
ppds-mcp-server/
├── api-config.json       ← PP/DS EntitySet-Definitionen
├── package.json          ← nur odata-mcp-proxy als Dependency
├── mta.yaml              ← BTP Multi-Target-Application Descriptor
└── xs-security.json      ← XSUAA Rollen (MCPViewer, MCPEditor)
```

### `package.json`

```json
{
  "name": "ppds-mcp-server",
  "version": "1.0.0",
  "scripts": {
    "start": "odata-mcp-proxy"
  },
  "dependencies": {
    "odata-mcp-proxy": "latest"
  }
}
```

### `mta.yaml`

```yaml
_schema-version: "3.1"
ID: ppds-mcp-server
version: 1.0.0

modules:
  - name: ppds-mcp-server
    type: nodejs
    path: .
    parameters:
      memory: 256M
      disk-quota: 512M
    requires:
      - name: ppds-destination-service
      - name: ppds-xsuaa
      - name: ppds-connectivity

resources:
  - name: ppds-destination-service
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite

  - name: ppds-xsuaa
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

  - name: ppds-connectivity
    type: org.cloudfoundry.managed-service
    parameters:
      service: connectivity
      service-plan: lite
```

### `xs-security.json`

```json
{
  "xsappname": "ppds-mcp-server",
  "tenant-mode": "dedicated",
  "scopes": [
    { "name": "$XSAPPNAME.MCPViewer", "description": "Read access to PP/DS MCP tools" }
  ],
  "role-templates": [
    {
      "name": "MCPViewer",
      "description": "Read-only PP/DS planning data",
      "scope-references": ["$XSAPPNAME.MCPViewer"]
    }
  ]
}
```

### Deployment

```bash
npm install -g mbt @sap/cf-tools
mbt build
cf login -a https://api.cf.<region>.hana.ondemand.com
cf deploy mta_archives/ppds-mcp-server_1.0.0.mtar
```

---

## API Konfiguration (`api-config.json`)

```json
{
  "server": {
    "name": "ppds-mcp-server",
    "version": "1.0.0",
    "description": "PP/DS Planning Assistant – S/4HANA PCE"
  },
  "apis": [
    {
      "name": "planned-orders",
      "destination": "S4H_PCE",
      "pathPrefix": "/sap/opu/odata/sap/API_PLANNED_ORDERS",
      "csrfProtected": true,
      "entitySets": [
        {
          "entitySet": "PlannedOrder",
          "urlPath": "A_PlannedOrder",
          "description": "PP/DS Planned Orders – Header data with dates, quantities, material",
          "keys": [{ "name": "PlannedOrder", "type": "string" }],
          "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false },
          "filterableProperties": [
            "Material", "MRPPlant", "PlannedOrderType",
            "PlannedOrderOpeningDate", "ProductionStartDate", "ProductionEndDate",
            "PlannedOrderPlannedStartDate", "PlannedOrderPlannedEndDate"
          ]
        },
        {
          "entitySet": "PlannedOrderComponent",
          "urlPath": "A_PlannedOrderComponent",
          "description": "Components (BOM items) of planned orders",
          "keys": [
            { "name": "PlannedOrder", "type": "string" },
            { "name": "Reservation", "type": "string" },
            { "name": "ReservationItem", "type": "string" }
          ],
          "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false },
          "filterableProperties": ["PlannedOrder", "Material", "StorageLocation"]
        },
        {
          "entitySet": "PlannedOrderCapacity",
          "urlPath": "A_PlannedOrderCapacity",
          "description": "Capacity requirements of planned orders",
          "keys": [
            { "name": "PlannedOrder", "type": "string" },
            { "name": "OperationInternalID", "type": "string" }
          ],
          "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false },
          "filterableProperties": ["PlannedOrder", "WorkCenter", "CapacityRequirementStartDate"]
        }
      ]
    },
    {
      "name": "production-orders",
      "destination": "S4H_PCE",
      "pathPrefix": "/sap/opu/odata/sap/API_PRODUCTION_ORDERS_2_SRV",
      "csrfProtected": true,
      "entitySets": [
        {
          "entitySet": "ProductionOrder",
          "urlPath": "A_ProductionOrder",
          "description": "Manufacturing / Production Orders – status, dates, quantities",
          "keys": [{ "name": "ManufacturingOrder", "type": "string" }],
          "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false },
          "filterableProperties": [
            "Material", "ProductionPlant", "ManufacturingOrderType",
            "MfgOrderPlannedStartDate", "MfgOrderPlannedEndDate",
            "ManufacturingOrderStatus"
          ]
        },
        {
          "entitySet": "ProductionOrderOperation",
          "urlPath": "A_ProductionOrderOperation",
          "description": "Operations / routing steps of production orders",
          "keys": [
            { "name": "ManufacturingOrder", "type": "string" },
            { "name": "ManufacturingOrderOperation", "type": "string" }
          ],
          "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false },
          "filterableProperties": ["ManufacturingOrder", "WorkCenter", "OpActualStartDate"]
        }
      ]
    },
    {
      "name": "work-centers",
      "destination": "S4H_PCE",
      "pathPrefix": "/sap/opu/odata/sap/API_WORK_CENTER_SRV",
      "csrfProtected": true,
      "entitySets": [
        {
          "entitySet": "WorkCenter",
          "urlPath": "A_WorkCenter",
          "description": "Work centers / resources",
          "keys": [
            { "name": "WorkCenterInternalID", "type": "string" }
          ],
          "operations": { "list": true, "get": true, "create": false, "update": false, "delete": false },
          "filterableProperties": ["WorkCenter", "WorkCenterTypeCode", "Plant"]
        }
      ]
    }
  ]
}
```

---

## Covered Use Cases (Read-only)

| Frage | API / EntitySet |
|---|---|
| Welche Planaufträge laufen diese Woche für Werk X? | `PlannedOrder` ($filter auf Datum + Werk) |
| Details zu Planauftrag 1234567? | `PlannedOrder` GET |
| Welche Komponenten braucht Planauftrag X? | `PlannedOrderComponent` |
| Kapazitätsbelastung für Planauftrag X? | `PlannedOrderCapacity` |
| Status Fertigungsauftrag X? | `ProductionOrder` GET |
| Welche Vorgänge hat Fertigungsauftrag X? | `ProductionOrderOperation` |
| Alle offenen Aufträge für Material Y? | `ProductionOrder` ($filter auf Material + Status) |
| Welche Arbeitsplätze gibt es in Werk Z? | `WorkCenter` |

---

## Nächste Schritte / Offen

- [ ] Phase 1: Lokal testen mit Claude Desktop
- [ ] BTP Subaccount + Cloud Connector Setup prüfen
- [ ] BTP Destination `S4H_PCE` anlegen + testen
- [ ] Phase 2: BTP CF Deployment
- [ ] AI Core Integration (Phase 3)
- [ ] Custom CDS Views für Lücken (Kapazitätsauslastung aggregiert, Alerts)

---

## Referenzen

- [odata-mcp-proxy GitHub](https://github.com/lemaiwo/odata-mcp-proxy)
- [odata-mcp-proxy Blog (Intro)](https://community.sap.com/t5/technology-blog-posts-by-members/odata-mcp-proxy-introduction/ba-p/14348684)
- [AI Core MCP Server Blog](https://community.sap.com/t5/technology-blog-posts-by-members/ai-core-mcp-server-managing-sap-ai-core-with-ai-assistants/ba-p/14348695)
- [SAP API Business Hub – Planned Orders](https://api.sap.com/api/API_PLANNED_ORDERS)
- [SAP API Business Hub – Production Orders](https://api.sap.com/api/API_PRODUCTION_ORDERS_2_SRV)
