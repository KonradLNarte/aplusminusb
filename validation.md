# RESONANSIA MCP SERVER — GEN 2 SPEC

> Part of [Gen2 Spec](index.md)

## 7. VALIDATION SCENARIO: PETTSON

Concrete acceptance test. A holding company with three subsidiaries. All tests MUST pass for a generation to be considered complete.

### 7.1 Tenant structure

```yaml
pettson_holding:                  # parent tenant
  children:
    - taylor_events               # event production
    - mountain_cabins             # cabin rentals
    - nordic_tickets              # ticket sales
```

### 7.2 Entity types per tenant (as type nodes)

```yaml
taylor_events:   [lead, campaign, event, venue, contact]
mountain_cabins: [property, booking, guest, season]
nordic_tickets:  [match, ticket_batch, package, partner]
```

### 7.3 Cross-tenant edges

```yaml
- package(nordic_tickets)  --[includes]-->    property(mountain_cabins)
- campaign(taylor_events)  --[sells]-->       ticket_batch(nordic_tickets)
- lead(taylor_events)      --[also_booked]--> booking(mountain_cabins)
```

### 7.4 Grants for cross-tenant access

```yaml
# These grants are created as seed data, stored in grants table
- Grant(subject=taylor_events, object=nordic_tickets.packages, capability=READ)
- Grant(subject=taylor_events, object=mountain_cabins.properties, capability=READ)
- Grant(subject=mountain_cabins, object=nordic_tickets.packages, capability=READ)
- Grant(subject=nordic_tickets, object=mountain_cabins.properties, capability=TRAVERSE)
```

### 7.5 Agent roles and scopes

```yaml
sales_agent:
  scopes: [tenant:taylor_events:read, tenant:nordic_tickets:read, tenant:mountain_cabins:read, tenant:taylor_events:write]

content_agent:
  scopes: [tenant:taylor_events:nodes:campaign:write, tenant:taylor_events:nodes:campaign:read]

booking_agent:
  scopes: [tenant:mountain_cabins:write, tenant:nordic_tickets:read]

partner_travel_agency:
  scopes: [tenant:nordic_tickets:nodes:package:read, tenant:mountain_cabins:nodes:property:read]
  token_expiry: 30 days
```

### 7.6 Acceptance tests

Every test is a tool call with expected outcome. **All must pass.**

```yaml
T01_semantic_cross_tenant_search:
  agent: sales_agent
  call: find_entities(query="football fans interested in cabin packages")
  expect: Returns leads from taylor_events matching semantic query on interest field
  validates: semantic search + RLS (cross-tenant edge discovery is via subsequent explore_graph)

T02_create_entity:
  agent: content_agent
  call: store_entity(entity_type="campaign", data={name: "Summer Festival 2026"})
  expect: Success. Returns entity_id, entity_type, version=1, epistemic="asserted", event_id.
  validates: entity creation + event emission + event lineage

T03_permission_denial:
  agent: content_agent
  call: find_entities(entity_types=["lead"])
  expect: DENIED or empty results (no lead read scope)
  validates: scope enforcement

T04_cross_tenant_traverse:
  agent: booking_agent
  call: explore_graph(start_id=PACKAGE_NODE, edge_types=["includes"], direction="outgoing")
  expect: Returns cabin properties from mountain_cabins tenant
  validates: cross-tenant edge traversal + grants table consultation + RLS

T05_partner_read_only:
  agent: partner_travel_agency
  call: store_entity(entity_type="package", data={...})
  expect: DENIED (read-only scope)
  validates: write protection on partner tokens

T06_audit_trail:
  agent: any
  precondition: Perform any mutation
  call: get_timeline(entity_id=MUTATED_ENTITY)
  expect: Timeline includes event with created_by = agent identity AND event_id matches entity's created_by_event
  validates: automatic event emission + event lineage + audit

T07_bitemporal_query:
  agent: sales_agent
  precondition: Lead exists, was updated on Feb 1, queried on Mar 1
  call: query_at_time(entity_id=LEAD_ID, valid_at="2026-01-15")
  expect: Returns lead data as it was on Jan 15 (before Feb 1 update)
  validates: bitemporal point-in-time query

T08_timeline:
  agent: sales_agent
  call: get_timeline(entity_id=LEAD_ID)
  expect: Chronological list: created → contacted → responded → booked. Each entry has event_id.
  validates: event aggregation + version history merge

T09_capture_thought:
  agent: sales_agent
  call: capture_thought(content="Met Johan at the cabin fair, he wants 20 tickets for Allsvenskan and a cabin for midsommar", source="manual")
  expect: Creates a lead node (epistemic="hypothesis"), finds-or-creates "Johan" contact, creates edges to ticket and cabin concepts
  validates: LLM extraction + entity linking + epistemic status + multi-step automation

T10_schema_discovery:
  agent: sales_agent
  call: get_schema(tenant_id=TAYLOR_EVENTS_ID)
  expect: Returns all entity types (as type nodes) with counts, example fields, and type_node_ids
  validates: schema introspection via type nodes

T11_deduplication:
  agent: sales_agent
  precondition: T09 has been run (Johan exists)
  call: capture_thought(content="Johan Eriksson called again about the midsommar cabin", source="manual")
  expect: Links to EXISTING Johan entity (not creating a duplicate). match_confidence returned.
  validates: entity deduplication in capture_thought

T12_epistemic_status:
  agent: sales_agent
  precondition: T09 has created Johan as hypothesis
  call: store_entity(entity_id=JOHAN_ID, data={name: "Johan Eriksson", phone: "070-123456"}, epistemic="confirmed")
  expect: Johan's epistemic status changes to "confirmed". Event emitted with intent_type "epistemic_change".
  validates: epistemic status transitions + event emission

T13_event_primacy:
  agent: any
  precondition: Multiple entities have been created
  call: Direct database query (not via MCP — test infrastructure only)
  expect: Every row in nodes table has a non-null created_by_event. Every referenced event exists in events table.
  validates: event lineage invariant

T14_verify_lineage:
  agent: any
  precondition: Multiple entities created via store_entity and capture_thought
  call: verify_lineage(entity_id=LEAD_ID)
  expect: is_valid=true, orphaned_facts=[], event_count > 0
  validates: event lineage integrity check via tool (not just DB-level)
```

### 7.8 Exact seed data (gen1 — Pettson scenario)

All UUIDs below are deterministic for reproducibility. Format: well-known prefixed UUIDs.

**IMPORTANT: Event generation for seed data.** Per INV-LINEAGE, every fact row requires a `created_by_event`. The seed script MUST:
1. Create one `entity_created` event per type node and entity node
2. Create one `edge_created` event per edge
3. Create one `grant_created` event per grant
4. All seed events use `created_by = METATYPE_ID` (bootstrap actor)
5. All seed events and fact rows are inserted within a single transaction

```yaml
# ═══ WELL-KNOWN UUIDs ═══
METATYPE_ID:      "00000000-0000-7000-0000-000000000001"
SYSTEM_TENANT_ID: "00000000-0000-7000-0000-000000000000"
ACTOR_TYPE_ID:    "00000000-0000-7000-0000-000000000002"

# ═══ TENANTS ═══
tenants:
  - tenant_id: "10000000-0000-7000-0000-000000000001"
    name: "Pettson Holding AB"
    parent_id: null
    config: { "plan": "enterprise", "is_holding": true }

  - tenant_id: "10000000-0000-7000-0000-000000000002"
    name: "Taylor Events AB"
    parent_id: "10000000-0000-7000-0000-000000000001"
    config: { "plan": "pro" }

  - tenant_id: "10000000-0000-7000-0000-000000000003"
    name: "Mountain Cabins AB"
    parent_id: "10000000-0000-7000-0000-000000000001"
    config: { "plan": "pro" }

  - tenant_id: "10000000-0000-7000-0000-000000000004"
    name: "Nordic Tickets AB"
    parent_id: "10000000-0000-7000-0000-000000000001"
    config: { "plan": "pro" }

# ═══ TYPE NODES (per tenant, all type_node_id = METATYPE_ID) ═══

# Taylor Events types
type_nodes_taylor:
  - node_id: "20000000-0000-7000-0001-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor_events
    data:
      name: "lead"
      kind: "entity_type"
      description: "A potential customer or sales opportunity"
      label_schema:
        type: object
        required: [name]
        properties:
          name: { type: string }
          email: { type: string, format: email }
          phone: { type: string }
          interest: { type: string }
          source: { type: string }
          status: { type: string, enum: [new, contacted, qualified, converted, lost] }

  - node_id: "20000000-0000-7000-0001-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    data:
      name: "campaign"
      kind: "entity_type"
      description: "A marketing or sales campaign"
      label_schema:
        type: object
        required: [name]
        properties:
          name: { type: string }
          type: { type: string, enum: [digital, event, print, partnership] }
          budget: { type: number }
          start_date: { type: string, format: date }
          end_date: { type: string, format: date }

  - node_id: "20000000-0000-7000-0001-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    data:
      name: "event"
      kind: "entity_type"
      description: "A scheduled event (concert, show, match)"
      label_schema:
        type: object
        required: [name, date]
        properties:
          name: { type: string }
          date: { type: string, format: date }
          venue: { type: string }
          capacity: { type: integer }

  - node_id: "20000000-0000-7000-0001-000000000004"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    data:
      name: "venue"
      kind: "entity_type"
      description: "A physical location for events"
      label_schema:
        type: object
        required: [name]
        properties:
          name: { type: string }
          city: { type: string }
          capacity: { type: integer }

  - node_id: "20000000-0000-7000-0001-000000000005"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    data:
      name: "contact"
      kind: "entity_type"
      description: "A person or organization contact"
      label_schema:
        type: object
        required: [name]
        properties:
          name: { type: string }
          email: { type: string }
          phone: { type: string }
          role: { type: string }

# Mountain Cabins types
type_nodes_mountain:
  - node_id: "20000000-0000-7000-0002-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    data:
      name: "property"
      kind: "entity_type"
      description: "A rental cabin or property"
      label_schema:
        type: object
        required: [name, location]
        properties:
          name: { type: string }
          location: { type: string }
          beds: { type: integer }
          price_per_night: { type: number }
          amenities: { type: array, items: { type: string } }

  - node_id: "20000000-0000-7000-0002-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    data:
      name: "booking"
      kind: "entity_type"
      description: "A cabin reservation"
      label_schema:
        type: object
        required: [check_in, check_out]
        properties:
          check_in: { type: string, format: date }
          check_out: { type: string, format: date }
          guests: { type: integer }
          status: { type: string, enum: [pending, confirmed, cancelled] }
          total_price: { type: number }

  - node_id: "20000000-0000-7000-0002-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    data:
      name: "guest"
      kind: "entity_type"
      description: "A cabin guest"
      label_schema:
        type: object
        required: [name]
        properties:
          name: { type: string }
          email: { type: string }
          phone: { type: string }

  - node_id: "20000000-0000-7000-0002-000000000004"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    data:
      name: "season"
      kind: "entity_type"
      description: "A pricing season"
      label_schema:
        type: object
        required: [name, start_date, end_date]
        properties:
          name: { type: string }
          start_date: { type: string, format: date }
          end_date: { type: string, format: date }
          price_multiplier: { type: number }

# Nordic Tickets types
type_nodes_nordic:
  - node_id: "20000000-0000-7000-0003-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    data:
      name: "match"
      kind: "entity_type"
      description: "A sports match or event"
      label_schema:
        type: object
        required: [name, date]
        properties:
          name: { type: string }
          date: { type: string, format: date }
          sport: { type: string }
          venue: { type: string }
          teams: { type: array, items: { type: string } }

  - node_id: "20000000-0000-7000-0003-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    data:
      name: "ticket_batch"
      kind: "entity_type"
      description: "A batch of tickets for sale"
      label_schema:
        type: object
        required: [quantity, price_per_ticket]
        properties:
          quantity: { type: integer }
          price_per_ticket: { type: number }
          category: { type: string, enum: [standard, vip, corporate] }
          sold: { type: integer, default: 0 }

  - node_id: "20000000-0000-7000-0003-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    data:
      name: "package"
      kind: "entity_type"
      description: "A combined package (tickets + accommodation)"
      label_schema:
        type: object
        required: [name, price]
        properties:
          name: { type: string }
          price: { type: number }
          description: { type: string }
          includes_accommodation: { type: boolean }

  - node_id: "20000000-0000-7000-0003-000000000004"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    data:
      name: "partner"
      kind: "entity_type"
      description: "An external distribution partner"
      label_schema:
        type: object
        required: [name]
        properties:
          name: { type: string }
          type: { type: string, enum: [travel_agency, corporate, reseller] }
          commission_pct: { type: number }

# Edge type nodes (shared across tenants via system tenant)
edge_type_nodes:
  - node_id: "20000000-0000-7000-0000-000000000E01"
    tenant_id: "00000000-0000-7000-0000-000000000000"  # system tenant
    data:
      name: "includes"
      kind: "edge_type"
      description: "Package includes property/ticket"

  - node_id: "20000000-0000-7000-0000-000000000E02"
    tenant_id: "00000000-0000-7000-0000-000000000000"
    data:
      name: "sells"
      kind: "edge_type"
      description: "Campaign sells ticket batch"

  - node_id: "20000000-0000-7000-0000-000000000E03"
    tenant_id: "00000000-0000-7000-0000-000000000000"
    data:
      name: "also_booked"
      kind: "edge_type"
      description: "Lead also booked a cabin"

  - node_id: "20000000-0000-7000-0000-000000000E04"
    tenant_id: "00000000-0000-7000-0000-000000000000"
    data:
      name: "booked_at"
      kind: "edge_type"
      description: "Booking is at a property"

  - node_id: "20000000-0000-7000-0000-000000000E05"
    tenant_id: "00000000-0000-7000-0000-000000000000"
    data:
      name: "for_match"
      kind: "edge_type"
      description: "Ticket batch is for a match"

  - node_id: "20000000-0000-7000-0000-000000000E06"
    tenant_id: "00000000-0000-7000-0000-000000000000"
    data:
      name: "contacted_via"
      kind: "edge_type"
      description: "Lead was contacted via campaign"

# ═══ SAMPLE ENTITIES (≥3 per tenant) ═══

entities_taylor:
  - node_id: "30000000-0000-7000-0001-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    type_node_id: "20000000-0000-7000-0001-000000000001"  # lead
    valid_from: "2026-01-01T00:00:00Z"  # backdated for T07 bitemporal test
    data:
      name: "Erik Lindström"
      email: "erik@example.com"
      phone: "070-111-2222"
      interest: "football and cabin packages"
      source: "website"
      status: "qualified"
    epistemic: "asserted"

  - node_id: "30000000-0000-7000-0001-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    type_node_id: "20000000-0000-7000-0001-000000000002"  # campaign
    data:
      name: "Summer Festival 2026"
      type: "event"
      budget: 50000
      start_date: "2026-06-01"
      end_date: "2026-08-31"
    epistemic: "asserted"

  - node_id: "30000000-0000-7000-0001-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000002"
    type_node_id: "20000000-0000-7000-0001-000000000003"  # event
    data:
      name: "Allsvenskan Semifinal"
      date: "2026-07-15"
      venue: "Gamla Ullevi"
      capacity: 18000
    epistemic: "confirmed"

entities_mountain:
  - node_id: "30000000-0000-7000-0002-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    type_node_id: "20000000-0000-7000-0002-000000000001"  # property
    data:
      name: "Fjällstugan Björnen"
      location: "Åre"
      beds: 8
      price_per_night: 2500
      amenities: ["sauna", "fireplace", "ski-in/ski-out"]
    epistemic: "confirmed"

  - node_id: "30000000-0000-7000-0002-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    type_node_id: "20000000-0000-7000-0002-000000000002"  # booking
    data:
      check_in: "2026-06-19"
      check_out: "2026-06-22"
      guests: 4
      status: "confirmed"
      total_price: 7500
    epistemic: "confirmed"

  - node_id: "30000000-0000-7000-0002-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000003"
    type_node_id: "20000000-0000-7000-0002-000000000003"  # guest
    data:
      name: "Erik Lindström"
      email: "erik@example.com"
      phone: "070-111-2222"
    epistemic: "asserted"

entities_nordic:
  - node_id: "30000000-0000-7000-0003-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    type_node_id: "20000000-0000-7000-0003-000000000001"  # match
    data:
      name: "Allsvenskan: IFK vs AIK"
      date: "2026-07-15"
      sport: "football"
      venue: "Gamla Ullevi"
      teams: ["IFK Göteborg", "AIK"]
    epistemic: "confirmed"

  - node_id: "30000000-0000-7000-0003-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    type_node_id: "20000000-0000-7000-0003-000000000002"  # ticket_batch
    data:
      quantity: 500
      price_per_ticket: 350
      category: "standard"
      sold: 127
    epistemic: "confirmed"

  - node_id: "30000000-0000-7000-0003-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000004"
    type_node_id: "20000000-0000-7000-0003-000000000003"  # package
    data:
      name: "Midsommar Football & Cabin"
      price: 8900
      description: "2 match tickets + 3 nights cabin near Åre"
      includes_accommodation: true
    epistemic: "asserted"

# ═══ CROSS-TENANT EDGES ═══

cross_tenant_edges:
  - edge_id: "40000000-0000-7000-0000-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000004"  # nordic_tickets owns
    type_node_id: "20000000-0000-7000-0000-000000000E01"  # includes
    source_id: "30000000-0000-7000-0003-000000000003"   # package (nordic)
    target_id: "30000000-0000-7000-0002-000000000001"   # property (mountain)

  - edge_id: "40000000-0000-7000-0000-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor owns
    type_node_id: "20000000-0000-7000-0000-000000000E02"  # sells
    source_id: "30000000-0000-7000-0001-000000000002"   # campaign (taylor)
    target_id: "30000000-0000-7000-0003-000000000002"   # ticket_batch (nordic)

  - edge_id: "40000000-0000-7000-0000-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor owns
    type_node_id: "20000000-0000-7000-0000-000000000E03"  # also_booked
    source_id: "30000000-0000-7000-0001-000000000001"   # lead (taylor)
    target_id: "30000000-0000-7000-0002-000000000002"   # booking (mountain)

# ═══ INTRA-TENANT EDGES ═══

intra_tenant_edges:
  - edge_id: "40000000-0000-7000-0000-000000000010"
    tenant_id: "10000000-0000-7000-0000-000000000003"  # mountain
    type_node_id: "20000000-0000-7000-0000-000000000E04"  # booked_at
    source_id: "30000000-0000-7000-0002-000000000002"   # booking
    target_id: "30000000-0000-7000-0002-000000000001"   # property

  - edge_id: "40000000-0000-7000-0000-000000000011"
    tenant_id: "10000000-0000-7000-0000-000000000004"  # nordic
    type_node_id: "20000000-0000-7000-0000-000000000E05"  # for_match
    source_id: "30000000-0000-7000-0003-000000000002"   # ticket_batch
    target_id: "30000000-0000-7000-0003-000000000001"   # match

  - edge_id: "40000000-0000-7000-0000-000000000012"
    tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor
    type_node_id: "20000000-0000-7000-0000-000000000E06"  # contacted_via
    source_id: "30000000-0000-7000-0001-000000000001"   # lead
    target_id: "30000000-0000-7000-0001-000000000002"   # campaign

# ═══ GRANTS ═══

grants:
  - grant_id: "50000000-0000-7000-0000-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000004"  # nordic issues
    subject_tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor receives
    object_node_id: "30000000-0000-7000-0003-000000000003"  # package node
    capability: "READ"

  - grant_id: "50000000-0000-7000-0000-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000003"  # mountain issues
    subject_tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor receives
    object_node_id: "30000000-0000-7000-0002-000000000001"  # property node
    capability: "READ"

  - grant_id: "50000000-0000-7000-0000-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000004"  # nordic issues
    subject_tenant_id: "10000000-0000-7000-0000-000000000003"  # mountain receives
    object_node_id: "30000000-0000-7000-0003-000000000003"  # package node
    capability: "READ"

  - grant_id: "50000000-0000-7000-0000-000000000004"
    tenant_id: "10000000-0000-7000-0000-000000000003"  # mountain issues
    subject_tenant_id: "10000000-0000-7000-0000-000000000004"  # nordic receives
    object_node_id: "30000000-0000-7000-0002-000000000001"  # property node
    capability: "TRAVERSE"

# ═══ ACTOR NODES ═══

actor_nodes:
  - node_id: "60000000-0000-7000-0000-000000000001"
    tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor
    type_node_id: "00000000-0000-7000-0000-000000000002"  # actor type
    data:
      name: "Sales Agent"
      actor_type: "agent"
      external_id: "sales-agent-001"
      purpose: "Manages leads and cross-tenant searches"
      scopes: ["tenant:10000000-0000-7000-0000-000000000002:read", "tenant:10000000-0000-7000-0000-000000000003:read", "tenant:10000000-0000-7000-0000-000000000004:read", "tenant:10000000-0000-7000-0000-000000000002:write"]

  - node_id: "60000000-0000-7000-0000-000000000002"
    tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor
    type_node_id: "00000000-0000-7000-0000-000000000002"
    data:
      name: "Content Agent"
      actor_type: "agent"
      external_id: "content-agent-001"
      purpose: "Manages campaigns"
      scopes: ["tenant:10000000-0000-7000-0000-000000000002:nodes:campaign:write", "tenant:10000000-0000-7000-0000-000000000002:nodes:campaign:read"]

  - node_id: "60000000-0000-7000-0000-000000000003"
    tenant_id: "10000000-0000-7000-0000-000000000003"  # mountain
    type_node_id: "00000000-0000-7000-0000-000000000002"
    data:
      name: "Booking Agent"
      actor_type: "agent"
      external_id: "booking-agent-001"
      purpose: "Manages bookings and properties"
      scopes: ["tenant:10000000-0000-7000-0000-000000000003:write", "tenant:10000000-0000-7000-0000-000000000004:read"]

  - node_id: "60000000-0000-7000-0000-000000000004"
    tenant_id: "10000000-0000-7000-0000-000000000004"  # nordic
    type_node_id: "00000000-0000-7000-0000-000000000002"
    data:
      name: "Partner Travel Agency"
      actor_type: "agent"
      external_id: "partner-travel-001"
      purpose: "External partner with read-only access"
      scopes: ["tenant:10000000-0000-7000-0000-000000000004:nodes:package:read", "tenant:10000000-0000-7000-0000-000000000003:nodes:property:read"]

# ═══ AGENT JWT TOKENS ═══
# These are the JWT claims for each agent. Implementation generates signed JWTs from these.

agent_tokens:
  sales_agent:
    sub: "sales-agent-001"
    iss: "resonansia-admin"
    aud: "resonansia-mcp"
    tenant_ids: ["10000000-0000-7000-0000-000000000002", "10000000-0000-7000-0000-000000000003", "10000000-0000-7000-0000-000000000004"]
    scopes: ["tenant:10000000-0000-7000-0000-000000000002:read", "tenant:10000000-0000-7000-0000-000000000003:read", "tenant:10000000-0000-7000-0000-000000000004:read", "tenant:10000000-0000-7000-0000-000000000002:write"]
    exp: "+1h"

  content_agent:
    sub: "content-agent-001"
    iss: "resonansia-admin"
    aud: "resonansia-mcp"
    tenant_ids: ["10000000-0000-7000-0000-000000000002"]
    scopes: ["tenant:10000000-0000-7000-0000-000000000002:nodes:campaign:write", "tenant:10000000-0000-7000-0000-000000000002:nodes:campaign:read"]
    exp: "+1h"

  booking_agent:
    sub: "booking-agent-001"
    iss: "resonansia-admin"
    aud: "resonansia-mcp"
    tenant_ids: ["10000000-0000-7000-0000-000000000003", "10000000-0000-7000-0000-000000000004"]
    scopes: ["tenant:10000000-0000-7000-0000-000000000003:write", "tenant:10000000-0000-7000-0000-000000000004:read"]
    exp: "+1h"

  partner_travel_agency:
    sub: "partner-travel-001"
    iss: "resonansia-admin"
    aud: "resonansia-mcp"
    tenant_ids: ["10000000-0000-7000-0000-000000000004", "10000000-0000-7000-0000-000000000003"]
    scopes: ["tenant:10000000-0000-7000-0000-000000000004:nodes:package:read", "tenant:10000000-0000-7000-0000-000000000003:nodes:property:read"]
    exp: "+30d"
```

### 7.9 Exact test assertions (gen1)

```yaml
T01_semantic_cross_tenant_search:
  agent: sales_agent
  request:
    tool: find_entities
    params:
      query: "football fans interested in cabin packages"
      tenant_id: "10000000-0000-7000-0000-000000000002"  # taylor_events
      limit: 10
  expected_response:
    results:
      - entity_id: "30000000-0000-7000-0001-000000000001"  # Erik Lindström
        entity_type: "lead"
        data: { name: "Erik Lindström", interest: "football and cabin packages" }
        similarity: ">0.7"  # semantic match threshold
        epistemic: "asserted"
    total_count: ">=1"
  notes: "Erik's interest field contains 'football' and 'cabin packages'. Cross-tenant edges to booking and tickets should be discoverable via subsequent explore_graph."

T02_create_entity:
  agent: content_agent
  request:
    tool: store_entity
    params:
      entity_type: "campaign"
      data: { name: "Summer Festival 2026" }
      tenant_id: "10000000-0000-7000-0000-000000000002"
  expected_response:
    entity_id: "non-null UUID"
    entity_type: "campaign"
    data: { name: "Summer Festival 2026" }
    epistemic: "asserted"
    valid_from: "~now()"
    event_id: "non-null UUID"
    previous_version_id: null
  post_conditions:
    - "Event exists in events table with intent_type='entity_created'"
    - "Node exists in nodes table with is_deleted=false, valid_to='infinity'"
    - "Event.created_by = content_agent actor node_id"

T03_permission_denial:
  agent: content_agent
  request:
    tool: find_entities
    params:
      entity_types: ["lead"]
      tenant_id: "10000000-0000-7000-0000-000000000002"
  expected_response:
    error_code: "AUTH_DENIED"
    message: "contains 'lead' and 'read'"
  notes: "content_agent has campaign read/write only, no lead scope"

T04_cross_tenant_traverse:
  agent: booking_agent
  request:
    tool: explore_graph
    params:
      start_id: "30000000-0000-7000-0003-000000000003"  # package (nordic)
      edge_types: ["includes"]
      direction: "outgoing"
  expected_response:
    center:
      entity_id: "30000000-0000-7000-0003-000000000003"
      entity_type: "package"
    connections:
      - edge_type: "includes"
        direction: "outgoing"
        entity:
          entity_id: "30000000-0000-7000-0002-000000000001"  # cabin property
          entity_type: "property"
          tenant_id: "10000000-0000-7000-0000-000000000003"
        depth: 1
  notes: "Requires grant: nordic→mountain TRAVERSE on property node"

T05_partner_read_only:
  agent: partner_travel_agency
  request:
    tool: store_entity
    params:
      entity_type: "package"
      data: { name: "Unauthorized Package" }
      tenant_id: "10000000-0000-7000-0000-000000000004"
  expected_response:
    error_code: "AUTH_DENIED"
    message: "contains 'write'"
  notes: "partner has read-only scope"

T06_audit_trail:
  agent: sales_agent
  precondition: "Run store_entity first to create a mutation"
  step_1:
    tool: store_entity
    params:
      entity_type: "lead"
      data: { name: "Test Lead Audit", source: "test" }
      tenant_id: "10000000-0000-7000-0000-000000000002"
    capture: MUTATED_ENTITY_ID
  step_2:
    tool: get_timeline
    params:
      entity_id: "$MUTATED_ENTITY_ID"
  expected_response:
    entity:
      entity_id: "$MUTATED_ENTITY_ID"
    timeline:
      - type: "version_change"
        event_id: "non-null UUID"
        created_by: "60000000-0000-7000-0000-000000000001"  # sales_agent actor

T07_bitemporal_query:
  agent: sales_agent
  precondition: |
    Lead 30000000-0000-7000-0001-000000000001 (Erik) was created in seed data.
    Then updated on 2026-02-01 with new interest.
    Query at 2026-01-15 should return original data.
  setup:
    tool: store_entity
    params:
      entity_id: "30000000-0000-7000-0001-000000000001"
      entity_type: "lead"
      data: { name: "Erik Lindström", interest: "football, cabin, VIP packages", status: "qualified" }
      valid_from: "2026-02-01T00:00:00Z"
  request:
    tool: query_at_time
    params:
      entity_id: "30000000-0000-7000-0001-000000000001"
      valid_at: "2026-01-15T00:00:00Z"
  expected_response:
    entity_id: "30000000-0000-7000-0001-000000000001"
    entity_type: "lead"
    data: { name: "Erik Lindström", interest: "football and cabin packages" }  # original
    valid_from: "<=2026-01-15"
    valid_to: "2026-02-01T00:00:00Z"

T08_timeline:
  agent: sales_agent
  request:
    tool: get_timeline
    params:
      entity_id: "30000000-0000-7000-0001-000000000001"  # Erik (after T07 update)
  expected_response:
    timeline:
      - type: "version_change"
        data: { contains: "entity_created" }
        event_id: "non-null"
      - type: "version_change"
        data: { contains: "entity_updated" }
        event_id: "non-null"

T09_capture_thought:
  agent: sales_agent
  request:
    tool: capture_thought
    params:
      content: "Met Johan at the cabin fair, he wants 20 tickets for Allsvenskan and a cabin for midsommar"
      source: "manual"
      tenant_id: "10000000-0000-7000-0000-000000000002"
  expected_response:
    created_node:
      entity_type: "lead" OR "contact"
      data: { name: "contains 'Johan'" }
      epistemic: "hypothesis"
    extracted_entities: "length >= 1"
    action_items: "length >= 0"
  post_conditions:
    - "Event with intent_type='thought_captured' exists"
    - "Node with epistemic='hypothesis' created"
    - "At least one edge created linking Johan to relevant entities"

T10_schema_discovery:
  agent: sales_agent
  request:
    tool: get_schema
    params:
      tenant_id: "10000000-0000-7000-0000-000000000002"
  expected_response:
    entity_types:
      - name: "lead"
        node_count: ">=1"
        type_node_id: "20000000-0000-7000-0001-000000000001"
      - name: "campaign"
        node_count: ">=1"
        type_node_id: "20000000-0000-7000-0001-000000000002"
      - name: "event"
        type_node_id: "20000000-0000-7000-0001-000000000003"
      - name: "venue"
        type_node_id: "20000000-0000-7000-0001-000000000004"
      - name: "contact"
        type_node_id: "20000000-0000-7000-0001-000000000005"

T11_deduplication:
  agent: sales_agent
  precondition: "T09 has been run (Johan exists as hypothesis)"
  request:
    tool: capture_thought
    params:
      content: "Johan Eriksson called again about the midsommar cabin"
      source: "manual"
      tenant_id: "10000000-0000-7000-0000-000000000002"
  expected_response:
    extracted_entities:
      - entity_type: "contact" OR "lead"
        is_new: false
        match_confidence: ">0.85"
  notes: "MUST link to existing Johan entity, NOT create duplicate"

T12_epistemic_status:
  agent: sales_agent
  precondition: "T09 created Johan as hypothesis"
  setup: "Capture JOHAN_ID from T09 response"
  request:
    tool: store_entity
    params:
      entity_id: "$JOHAN_ID"
      entity_type: "contact"
      data: { name: "Johan Eriksson", phone: "070-123456" }
      epistemic: "confirmed"
  expected_response:
    entity_id: "$JOHAN_ID"
    epistemic: "confirmed"
    event_id: "non-null"
  post_conditions:
    - "Event with intent_type='epistemic_change' exists"
    - "Node version with epistemic='confirmed' is current"

T13_event_primacy:
  type: "database assertion (not MCP tool call)"
  query: |
    SELECT n.node_id, n.created_by_event
    FROM nodes n
    LEFT JOIN events e ON n.created_by_event = e.event_id
    WHERE e.event_id IS NULL
  expected: "Zero rows (no orphaned facts)"

T14_verify_lineage:
  agent: sales_agent
  request:
    tool: verify_lineage
    params:
      entity_id: "30000000-0000-7000-0001-000000000001"  # Erik lead
  expected_response:
    entity_id: "30000000-0000-7000-0001-000000000001"
    is_valid: true
    event_count: ">=1"
    orphaned_facts: []
```

### 7.7 Spec-level validation (for spec generations)

A spec generation is validated not by running code but by checking:

1. **Completeness**: Zero unresolved [DECIDE:genN] or [RESEARCH:genN] markers for the current generation.
2. **Precision**: Every schema definition, tool signature, auth rule, and test assertion is concrete enough that a coding agent needs zero design decisions to implement it. Test: can you extract a single unambiguous implementation from the spec text?
3. **Consistency**: No contradictions between sections. Every entity type referenced in validation tests exists in the schema. Every tool referenced in tests exists in the tool interface. Every scope referenced in agent roles exists in the auth model.
4. **Traceability**: Every resolved decision in the DECISION LOG has a corresponding spec text update (spec_section_updated field is non-empty).
5. **Handoff readiness**: The implementation brief (deliverable_6) contains enough information for a coding agent to start work without asking clarifying questions.

---

