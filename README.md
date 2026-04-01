# PP/DS MCP Server – Projektdokumentation

## Ziel

KI-gestützte Planungsauskunft direkt in SAP: Eine Custom Fiori App im S/4HANA PCE System schickt Planungsfragen an SAP AI Core, der über den MCP-Server auf PP/DS-Daten zugreift und die Antwort direkt zurück in die Fiori App liefert.

---

## Architektur (Zielzustand)

```
┌─────────────────────────────────┐
│       S/4HANA Private Cloud     │
│                                 │
│  ┌──────────────────────────┐   │
│  │  Custom Fiori App        │   │
│  │  (Input: Frage + Kontext)│   │
│  └────────────┬─────────────┘   │
│               │ OData/HTTP      │
│  ┌────────────▼─────────────┐   │
│  │  BTP Integration / ABAP  │   │
│  │  HTTP Outbound Call      │   │
│  └────────────┬─────────────┘   │
└───────────────┼─────────────────┘
                │ Cloud Connector / Private Link
                ▼
┌─────────────────────────────────┐
│          SAP BTP                │
│                                 │
│  ┌──────────────────────────┐   │
│  │  SAP AI Core             │   │
│  │  Orchestration Service   │   │
│  └────────────┬─────────────┘   │
│               │ MCP Tool Calls  │
│  ┌────────────▼─────────────┐   │
│  │  PP/DS MCP Server        │   │
│  │  (dieses Repo)           │   │
│  │  odata-mcp-proxy         │   │
│  └────────────┬─────────────┘   │
│               │ OData           │
└───────────────┼─────────────────┘
                │ Cloud Connector / Private Link
                ▼
┌─────────────────────────────────┐
│       S/4HANA Private Cloud     │
│                                 │
│  ┌──────────────────────────┐   │
│  │  PP/DS OData APIs        │   │
│  │  (Planned/Prod. Orders,  │   │
│  │   Work Centers, ...)     │   │
│  └──────────────────────────┘   │
└─────────────────────────────────┘
                │
                │ Antwort zurück durch denselben Pfad
                ▼
        Custom Fiori App Output
```

**Datenpfad:**
1. Fiori App sendet Frage + Kontextdaten (z.B. Planauftragsnummer, Werk) an AI Core
2. AI Core Orchestration versteht die Anfrage, wählt passende MCP-Tools
3. MCP-Server ruft die PP/DS OData APIs im S/4HANA PCE ab
4. Antwort läuft zurück: MCP → AI Core → Fiori App

---

## Komponenten

### 1. Custom Fiori App (S/4HANA PCE)
- Eingebettet in SAP-Transaktion oder eigenständige App
- Sendet strukturierten Kontext an AI Core (z.B. aktuell geöffneter Planauftrag, Werk, Zeitraum)
- Zeigt die AI-Antwort direkt in der UI an
- **Tech:** SAP UI5 / Fiori Elements, ABAP OData oder CAP Backend

### 2. SAP AI Core (BTP)
- Orchestration Service als zentrales AI-Brain
- Nimmt die Anfrage der Fiori App entgegen
- Entscheidet welche MCP-Tools aufgerufen werden
- Gibt formatierte Antwort zurück

### 3. PP/DS MCP Server (dieses Repo, BTP CF)
- Powered by `odata-mcp-proxy`
- Exposes PP/DS OData APIs als MCP-Tools
- Kein eigener Code — nur `api-config.json`
- Läuft auf BTP Cloud Foundry, nutzt Destination Service + Cloud Connector

### 4. S/4HANA PCE OData APIs
- Planaufträge (`API_PLANNED_ORDERS`)
- Fertigungsaufträge (`API_PRODUCTION_ORDERS_2_SRV`)
- Arbeitsplätze (`API_WORK_CENTER_SRV`)
- Erweiterbar via Custom CDS Views

---

## BTP Deployment (MCP Server)

### Voraussetzungen
- SAP BTP Subaccount (Cloud Foundry aktiviert)
- Cloud Connector konfiguriert mit Virtual Host für S/4HANA PCE
- BTP Destination `S4H_PCE` angelegt (OnPremise, Basic oder OAuth2SAML)
- Services: Destination, Connectivity, XSUAA

### BTP Destination `S4H_PCE`

| Property | Wert |
|---|---|
| Name | `S4H_PCE` |
| Type | `HTTP` |
| URL | Virtual Host (Cloud Connector) |
| Authentication | `BasicAuthentication` oder `OAuth2SAMLBearerAssertion` |
| Proxy Type | `OnPremise` |

### Deployment

```bash
npm install -g mbt
mbt build
cf login -a https://api.cf.<region>.hana.ondemand.com
cf deploy mta_archives/ppds-mcp-server_1.0.0.mtar
```

---

## API Konfiguration (`api-config.json`)

Drei API-Gruppen, read-only:

| API | EntitySets | Use Case |
|---|---|---|
| `API_PLANNED_ORDERS` | PlannedOrder, PlannedOrderComponent, PlannedOrderCapacity | Planaufträge lesen, Komponenten, Kapazitätsbedarfe |
| `API_PRODUCTION_ORDERS_2_SRV` | ProductionOrder, ProductionOrderOperation | Fertigungsaufträge + Vorgänge |
| `API_WORK_CENTER_SRV` | WorkCenter | Arbeitsplätze / Ressourcen |

→ Details in `api-config.json`

---

## Beispiel-Interaktionen

Fiori App schickt an AI Core (mit Kontext Werk=1000, Material=FG-001):

> *„Welche Planaufträge für Material FG-001 laufen diese Woche?"*
→ AI Core → `PlannedOrder_list` mit $filter auf Material + Datum → Antwort in Fiori

> *„Auf welchem Arbeitsplatz wird Planauftrag 1234567 gefertigt?"*
→ AI Core → `PlannedOrderCapacity_get` → WorkCenter aus Ergebnis → Antwort

> *„Welche Komponenten fehlen für Planauftrag 1234567?"*
→ AI Core → `PlannedOrderComponent_list` → Abgleich mit Verfügbarkeit → Antwort

---

## Covered Use Cases (Phase 1 – Read-only)

| Frage | API / EntitySet |
|---|---|
| Welche Planaufträge laufen diese Woche für Werk X? | `PlannedOrder` |
| Details zu Planauftrag XXXXXXX? | `PlannedOrder` GET |
| Welche Komponenten braucht Planauftrag X? | `PlannedOrderComponent` |
| Kapazitätsbedarf für Planauftrag X? | `PlannedOrderCapacity` |
| Status Fertigungsauftrag X? | `ProductionOrder` GET |
| Welche Vorgänge hat Fertigungsauftrag X? | `ProductionOrderOperation` |
| Alle offenen Aufträge für Material Y? | `ProductionOrder` ($filter) |
| Welche Arbeitsplätze gibt es in Werk Z? | `WorkCenter` |

---

## Roadmap

- [x] MCP Server Konfiguration (api-config.json)
- [x] BTP Deployment Setup (mta.yaml, xs-security.json)
- [ ] BTP Deployment ausführen + testen
- [ ] AI Core Orchestration: MCP-Tools registrieren
- [ ] Custom Fiori App: AI Core Integration (Input/Output)
- [ ] Phase 2: Write-Operationen (Planauftrag ändern)
- [ ] Custom CDS Views für Lücken (aggregierte Kapazitätsauslastung, Alerts)

---

## Referenzen

- [odata-mcp-proxy GitHub](https://github.com/lemaiwo/odata-mcp-proxy)
- [odata-mcp-proxy Blog](https://community.sap.com/t5/technology-blog-posts-by-members/odata-mcp-proxy-introduction/ba-p/14348684)
- [AI Core MCP Server Blog](https://community.sap.com/t5/technology-blog-posts-by-members/ai-core-mcp-server-managing-sap-ai-core-with-ai-assistants/ba-p/14348695)
- [SAP API Hub – Planned Orders](https://api.sap.com/api/API_PLANNED_ORDERS)
- [SAP API Hub – Production Orders](https://api.sap.com/api/API_PRODUCTION_ORDERS_2_SRV)
