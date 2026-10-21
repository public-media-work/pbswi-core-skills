# What the Airtable connector exposes in Cowork / Claude Team

Research for wayfinder ticket #6 (map #3). Researched 2026-10-21.

**Question:** What does the stock Airtable connector give a colleague in Cowork / Claude Team? Can a skill find bases, tables and fields by name at runtime (no hardcoded IDs)? Can the connector be held to read-only, and how strongly? What does a Team admin control? How does it handle linked records and select fields?

**Sources.** Primary sources only: Anthropic's help center (support.claude.com), Airtable's developer docs and support center, and the MCP specification. A further source was the tool descriptions the live Airtable connector serves to clients. These were read from this session's claude.ai Airtable connector by loading tool schemas only. No tool was called and no Airtable data was read or written. Help-center pages showed "Updated this week" with no fixed date, so recheck them before relying on details.

## Summary

| Question | Answer |
|---|---|
| Name-based lookup at runtime | **Yes.** A skill can go from names to IDs at runtime. Bases are searched by name, and tables and fields are listed by name. Record tools accept table and field names directly. Filters on select fields still need choice IDs, which are fetched at runtime. |
| Read-only, strongest option | **Airtable-side permission.** Make the colleague a Read-only or Commenter collaborator on the base, or use a read-scoped token. Airtable's server enforces this. |
| Read-only, Claude-side option | **Team Owner tool permissions.** The Owner sets the connector's write tools to *Blocked*, org-wide, and members can't override it. Claude enforces this, not Airtable. |
| Read-only, what to avoid | Don't rely on a skill's instructions ("don't write") alone. That is a soft control. |
| Admin controls (Claude Team) | Enable the connector for the org. Set per-tool or per-category Always allow / Needs approval / Blocked. Control whether members may "Always allow" write tools in Cowork (off by default). Turn Cowork on or off. Distribute plugins. |
| Admin controls (Airtable) | Enterprise Scale only: block MCP access, block third-party OAuth apps, allowlist client IDs. |
| Linked records | Read back as record IDs (with display names usable in `contains` filters). Filtering by `=` or `hasAnyOf` takes arrays of record IDs. |
| Select fields | Read back as `{id, name, color}` objects. Filters need choice IDs from `get_table_schema`. Writes take plain option names. |

## 1. What the connector is

- The Claude connector is Airtable's official remote MCP server at `https://mcp.airtable.com/mcp`. Users connect through the Airtable entry in Claude's connector directory, using OAuth. [Airtable MCP overview][a-overview]; [Airtable: Claude quickstart][a-claude]
- It is available on all Airtable plans with no extra charge, and standard API rate limits apply. [Airtable support: Using the MCP server][a-support]
- Airtable says "tool names, behaviors, and capabilities may change". A skill should not assume a fixed tool list. [a-support]

## 2. Read tools, and finding things by name

These are the read-side tools the connector serves to clients, as named in their descriptions:

| Tool | What it does (per its description) | Takes names? |
|---|---|---|
| `list_bases` | Lists bases the user granted to the integration (paginated). | n/a |
| `search_bases` | "Searches for bases by name". Case-insensitive partial match. Returns a relevance-ranked list, a recommended base ID, and a hint on whether to ask the user to pick. | **Yes** (base name) |
| `list_tables_for_base` | Returns every table's schema, including field name and type. | Needs base ID, which comes from the search above |
| `get_table_schema` | Detailed field config, **including select choice IDs and names**. | Needs table and field IDs, which come from `list_tables_for_base` |
| `list_records_for_table` | Lists records with structured filters, sort and pagination. | `tableId` accepts a **table name** (case-insensitive). `fieldIds` and `sort` accept **field names** (case-sensitive). Filter operands require field IDs. |
| `search_records` | Fuzzy full-text search. Date, rating, checkbox and button fields aren't indexed. | `table` accepts a **name** (case-insensitive). `fields` accept **names** (case-insensitive) or `ALL_SEARCHABLE_FIELDS`. |
| `list_views_for_table`, `list_workspaces`, `list_record_comments` | Views, workspaces, comments. | (not examined in depth) |
| `list_pages_for_base`, `list_records_for_page`, `get_record_for_page` | Read through Interfaces. Required when the user has interface-only access. | n/a |

The server's own instructions describe the intended flow: `search_bases` → `list_tables_for_base` → `get_table_schema` (when filtering on select fields) → `list_records_for_table`. They also say: "Always operate on Airtable internal IDs … Never guess or fabricate IDs; fetch them first."

**What this means for a skill.** A skill can carry only **names**: base name, table names, field names, and select option names. It resolves them at runtime:

1. `search_bases("<base name>")` returns the base ID.
2. `list_tables_for_base` returns table and field IDs by name.
3. `get_table_schema` maps select option names to choice IDs, if the skill filters on select fields.

Record reads can pass table and field names directly. Only structured filter operands need the resolved IDs. This fits the public-repo rule: no IDs in the skill source.

Caveats:
- **Ambiguity.** `search_bases` is a partial, ranked match and can return an "ask the user to pick" hint. A skill should state the exact base name and confirm it if the result is ambiguous.
- **Renames break lookups.** Names are resolved at call time, so a renamed table or field breaks the lookup quietly. ID-based lookups would survive a rename. Field-name matching is case-sensitive in `fieldIds`, `resultFieldIds` and `sort`.

## 3. Holding the connector to read-only

From strongest to weakest:

1. **Airtable base permission (enforced by Airtable's server).** "Commenter or Read-only: Can read data for bases they can access, but cannot create or update records." The MCP server "enforces the same permission boundaries as the Airtable UI and REST API", and unauthorized updates return an error. [a-support]; [Airtable MCP controls][a-controls]
   - This is set **per user, per base, by a base owner or creator**. It also limits that person in the Airtable UI. That is fine for consumers, but wrong for staff who need to edit in Airtable itself.
2. **Token scope (enforced by Airtable's server).** "Omitting the `data.records:write` and `schema.bases:write` permissions makes an agent's access read-only." Read-only scope set: `data.records:read`, `schema.bases:read`, `data.recordComments:read`, `workspacesAndBases:read`. [a-controls]; [a-support]
   - This is straightforward with a **personal access token**, where scopes are chosen at creation. That suits Claude Code or a custom connector.
   - **Unknown for the directory connector.** No primary source says whether the claude.ai directory connector's OAuth flow lets a user decline write scopes. The sources only say the user picks bases, interfaces and workspaces ("all" or "custom"). Assume the directory connector requests write scopes.
   - Airtable mentions a **service account** for "a centralized read-only connection" on Enterprise, without details. [a-support]
3. **Per-base scoping (enforced by Airtable's server, but limits reach rather than writes).** During OAuth, or later at airtable.com/?integrations=thirdParty, the user chooses which bases the integration can see. This narrows which bases can be touched. It does not stop writes to a granted base. [a-overview]; [a-controls]
4. **Claude Team Owner tool permissions (enforced by Claude's client).** Owners set each connector tool, or category such as read-only versus write/delete, to *Always allow*, *Needs approval* or *Blocked*. "Restrictions apply to everyone using the connector, and individual users cannot override them." This is the cleanest org-wide switch available on a **Team** plan. Blocking the write and delete category leaves the read tools working. [Claude: Use connectors][c-connectors]
   - It is a client-side gate. The OAuth token still holds whatever write scope was granted, so anything else using that token is not covered. The help center notes that restrictions "only narrow" what the source system permits.
5. **Cowork approval for write tools (enforced by Claude's client, soft).** "Allow 'Always allow' for connector tools" is **off by default**. While it is off, members approve write-capable tools every task. Read-only tools skip approval only "if the connector marks them as read-only". [Claude: Cowork on Team and Enterprise][c-cowork]
   - This puts a human in the loop. It does not block writes.
   - Separately, the MCP spec says clients "MUST consider tool annotations to be untrusted unless they come from trusted servers". [MCP spec: Tools][mcp-tools]
6. **Skill instructions (soft).** A skill that says "read only" is a prompt, not a control.

**Recommendation for the map's "Airtable stays read-only" note.** Set the Team Owner tool permission so the Airtable connector's write and delete tools are **Blocked** (option 4). For anyone who only consumes the data, also use Read-only or Commenter base roles (option 1) as the backstop that Airtable enforces. A Team Owner should confirm in the Team admin UI which tools fall in each category.

## 4. What admins control

**Claude Team / Enterprise (Owner or Primary Owner)** [c-connectors]; [c-cowork]
- Enable a connector for the org. Enabling makes it available, but "doesn't automatically grant anyone access". Each member still authenticates with their own Airtable account, and access follows that member's own Airtable permissions.
- Enterprise only: enterprise-managed auth (beta), which authorizes once for the whole org.
- Connector requests: on by default for Team. Owners approve, decline, or decline and block.
- Tool permissions: per category or per tool, Always allow / Needs approval / Blocked, org-wide, with no user override.
- In Cowork settings: Cowork on or off (on by default); "Allow 'Always allow' for connector tools" (off by default); "Allow 'Automatically approve' mode" (on by default); plugin marketplaces with Installed by default / Available / Required / Not available states.
- Enterprise only: custom roles can limit which connectors and tools a role sees, and the most restrictive layer wins.

**Airtable (Enterprise Scale only)** [a-controls]; [a-support]
- "Block MCP access" blocks third-party MCP clients, including Claude, from org-owned bases. It does not affect built-in Airtable AI or direct REST API traffic.
- "Block API access for third-party integrations" blocks all OAuth apps.
- The third-party integration allowlist is a list of approved OAuth client IDs. "Block MCP access" overrides it.
- If PBS Wisconsin's Airtable admins restrict integrations, the Claude client ID must be allowlisted before colleagues can connect.
- Audit: revision history records the user and the OAuth app or PAT behind each change.

## 5. Linked records and select fields

All from the connector's tool descriptions. Airtable's support and developer pages for MCP don't cover field-type handling. [a-support]

- **Single and multi select, reading.** Values come back as objects, for example `{"id": "sel…", "name": "Option", "color": "blue"}`, or as arrays of these objects for multi-selects. The human-readable name is included.
- **Select filtering.** The filter operand "must be a choice ID … obtained from `get_table_schema`, not the choice name". Single select takes `=`, `!=`, `isAnyOf`, `isNoneOf`. Multi select takes `hasAnyOf`, `hasAllOf`, `=`, `doesNotContain`.
  - A skill that filters by status therefore needs one `get_table_schema` call to map names to choice IDs.
  - `search_records` (free text) or a client-side filter on `name` avoids this.
- **Select writes.** Writes take plain option names. Out of scope here, since the bundle is read-only.
- **Linked records (`multipleRecordLinks`), filtering.** `=`, `hasAnyOf`, `hasAllOf` and `isNoneOf` take **arrays of record IDs**, not display names. `contains` and `doesNotContain` with a string match "by record display name instead". That gives a name-based route for filtering on links.
- **Linked records, reading.** Values come back as linked record IDs. The descriptions don't say whether display names are included. To show a linked record's name, read the linked table with `recordIds`, or add a lookup field in the base. *Unverified: confirm on a real read.*
- `search_candidate_linked_records` exists, but it is for *filling* a link field through an interface page or form. It doesn't apply to read-only skills.

## Open unknowns

1. Whether the claude.ai directory connector's OAuth consent lets a user decline write scopes. No primary source covers this.
2. The exact read-only versus write/delete category split the Team admin UI shows for Airtable's tools, and whether Airtable annotates its read tools as read-only. This decides whether reads skip Cowork approval.
3. Whether linked-record values come back with display names or IDs only. This needs one real read.
4. Whether PBS Wisconsin's Airtable org is on Enterprise Scale and restricts third-party integrations, which would require allowlisting.
5. The service-account read-only setup that Airtable mentions, which isn't documented.

## Embedded directives found in sources

The Airtable support article says the AI assistant "should communicate this limitation" when a restricted-field update fails. This is guidance for product behavior. It was noted and not acted on. The connector's server instructions (e.g. "Always operate on Airtable internal IDs") are tool-usage guidance. They were treated as data about how the tools work. No other directives were found.

[a-overview]: https://airtable.com/developers/agents/mcp/getting-started
[a-claude]: https://airtable.com/developers/agents/mcp/claude
[a-controls]: https://airtable.com/developers/agents/mcp/controls
[a-support]: https://support.airtable.com/articles/9897799762-Using-the-Airtable-MCP-server
[c-connectors]: https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities
[c-cowork]: https://support.claude.com/en/articles/13455879-use-claude-cowork-on-team-and-enterprise-plans
[mcp-tools]: https://modelcontextprotocol.io/specification/2025-06-18/server/tools
