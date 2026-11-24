# Helios MCP tools

<!-- Generated from the Helios MCP server by `npm run generate:client-kit`. Do not edit by hand. -->

Server version: `2.0.0`

This catalog lists what the Helios MCP server offers to AI assistants.
Assistants read the same descriptions, parameter schemas and output
schemas directly from the server when they connect, so you never need to
paste them anywhere.
Every tool acts only on the data of the company you logged in with; in the analytics tools, a super administrator may choose another company.

## Tools

| Tool | Title | Effect |
|---|---|---|
| [`describe_analytics_schema`](#describe_analytics_schema) | Descrever dados para análise | Read-only |
| [`run_analytics_query`](#run_analytics_query) | Executar consulta analítica | Read-only |
| [`create_client`](#create_client) | Cadastrar cliente | Write (additive) |
| [`update_client`](#update_client) | Atualizar cliente | Write (may change or delete existing data), idempotent |
| [`create_installation`](#create_installation) | Cadastrar números de instalação | Write (additive) |
| [`update_installation`](#update_installation) | Corrigir números da instalação | Write (may change or delete existing data), idempotent |
| [`create_consumer_unit`](#create_consumer_unit) | Cadastrar unidade consumidora | Write (additive) |
| [`update_consumer_unit`](#update_consumer_unit) | Atualizar unidade consumidora | Write (may change or delete existing data), idempotent |
| [`create_contact`](#create_contact) | Cadastrar contato | Write (additive) |
| [`update_contact`](#update_contact) | Atualizar contato | Write (may change or delete existing data), idempotent |
| [`delete_contact`](#delete_contact) | Excluir contato | Write (may change or delete existing data) |
| [`cancel_bank_slip`](#cancel_bank_slip) | Cancelar boleto | Write (may change or delete existing data), idempotent, acts outside Helios |

## Server instructions

The server sends these instructions to every assistant when it connects. Some assistants don't show them to their model, so each tool's description also carries the rules that tool depends on.

- Helios manages a Brazilian solar energy company's clients, CEMIG installations, consumer units (UCs), contacts, invoices, boletos, CEMIG energy bills, GD reports and generation units (UGs).
- Every tool acts only on the user's company; another company's uuid behaves like a missing one (NotFoundError), except that a super administrator may pass one from describe_analytics_schema's companies to run_analytics_query.
- Start with describe_analytics_schema: who you are, your company, now (Brasília) and, in its overview, verified lookup examples. Look records up with run_analytics_query from those examples: a client by CPF/CNPJ or name, what a number belongs to, a client's UCs and contacts, a contact by phone or e-mail, an invoice with its boletos and cancel_allowed, a CEMIG bill, the GD balance, a plant (UG). Resolve the record and take its uuid from a result before any write; right after a write, trust only a current result.
- Terms users mix up: "Cobrança" is the invoice (fatura, the Cobranças menu), but users also say it for one boleto. "Dar baixa" or "baixar" can mean a boleto cancel at the bank, a payment record (web app only: "Confirmar pagamento") or a boleto download (no tool does). "Conta de luz" or "conta de energia" is the CEMIG energy bill, not the invoice. "Instalação" is CEMIG's record of the installation and UC numbers, but users also say it for the UC, and a number may be a UC's or a plant's (UG, "usina"). Reads: take the likeliest meaning and say which. Writes: ask only when the meaning changes the record, tool or field written; always confirm the exact record before cancel_bank_slip or delete_contact.
- Today is the Brasília date (describe_analytics_schema's now, or current_date in SQL), never your own clock. Tool inputs take documents, CEPs and UC and installation numbers as digits only. Show values to users as: CPF 123.456.789-09, CNPJ 12.345.678/0001-90, UC number 3.012.345.678-90, installation number as its 10 digits, phone +55 (31) 99876-5432, CEP 30123-456, e-mail in lowercase, money R$ 1.234,56, dates 10/03/2026, timestamps 10/03/2026 14:30 (Brasília time as printed, never converted), periods FEV/2026.
- Invoices and boletos are read-only, except cancel_bank_slip, which cancels a boleto at its bank and can't be undone. Creates return ConflictError on duplicates. Money is BRL as exact decimal strings: quote it as stored, never recompute.
- Run one query at a time. For totals, rankings or trends, aggregate in SQL (a truncated result is incomplete) and say how current it is (dataAsOf).
- If analytics is unavailable (ServerError), lookups, and with them updates, deletes and cancels, can't be done here: say lookups are temporarily unavailable and the user can use Helios web. Never guess a uuid.
- A failed call returns isError with `<ErrorType>: <message>` (ValidationError, NotFoundError, ConflictError, ForbiddenError or ServerError); the pt-BR message can be shown as it is.
- Answer in Brazilian Portuguese unless the user writes in another language.

## Tool reference

### `describe_analytics_schema`

**Descrever dados para análise**

Effect: Read-only

Call it first in a conversation. It says who you are (your user, role and company; for a super administrator, every company, whose uuid run_analytics_query takes as companyUuid) and the current date and time in Brasília (now), the zone of every timestamp. It describes the read-only analytics views that run_analytics_query can use: what each view holds, its columns with types, units and allowed values, the aggregation rules that keep totals right, helper functions, limits and verified examples, both lookups of one record (a client, what a number belongs to, a contact, an invoice with its boletos) and analyses. Call it without arguments for the overview, with the rules and examples, then with views=[…] for only the columns of the views you need. Only administrators can use it (ForbiddenError otherwise).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `views` | array of `"company"`, `"clients"`, `"contacts"`, `"consumer_units_contacts"`, `"installations"`, `"consumer_units"`, `"generation_units"`, `"compensation_distributions"`, `"compensation_distribution_entries"`, `"invoices"`, `"extra_invoice_items"`, `"invoice_compensation_ratios"`, `"billings"`, `"energy_bills"`, `"gd_reports"`, `"notification_messages"` | no | Views to describe column by column. Omit for the overview of every view. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `dialect` | string |  |
| `timeZone` | string | Zone of every timestamp: Brasília (America/Sao_Paulo, UTC-03:00); dates have no zone. |
| `now` | string (date-time) | Current date and time in Brasília (America/Sao_Paulo, ISO 8601). Its date is today for the user: use it, never your own clock. |
| `you` | object | You: the connected user and their company. |
| `you.user` | object |  |
| `you.user.uuid` | string (uuid) |  |
| `you.user.name` | string |  |
| `you.user.role` | `"super_admin"`, `"admin"` | super_admin may also query every other company (companyUuid); admin only its own. |
| `you.company` | object | The company your queries read when you omit companyUuid. |
| `you.company.uuid` | string (uuid) |  |
| `you.company.name` | string |  |
| `you.company.cnpj` | string |  |
| `companies` | array of object | Optional. Overview for a super administrator only: every company, by name, whose uuid run_analytics_query takes as companyUuid. |
| `companies[].uuid` | string (uuid) |  |
| `companies[].name` | string |  |
| `companies[].cnpj` | string |  |
| `companies[].city` | string |  |
| `limits` | object |  |
| `limits.defaultMaxRows` | number |  |
| `limits.maxRows` | number |  |
| `limits.statementTimeoutSeconds` | number |  |
| `limits.maxSqlLength` | number |  |
| `limits.maxRowBytes` | number |  |
| `limits.maxResultChars` | number |  |
| `limits.maxStalenessSeconds` | number |  |
| `notes` | array of string | Optional. Overview only: the rules of the views and of the SQL guard. |
| `views` | array of object |  |
| `views[].name` | string |  |
| `views[].description` | string |  |
| `views[].columnNames` | array of string |  |
| `views[].columns` | array of object | Optional. |
| `views[].columns[].name` | string |  |
| `views[].columns[].type` | string |  |
| `views[].columns[].description` | string |  |
| `functions` | array of object | Optional. Overview only: the helper functions of the analytics schema. |
| `functions[].signature` | string |  |
| `functions[].description` | string |  |
| `examples` | array of object | Optional. Overview only: verified queries, from finding one record to analyses across many. |
| `examples[].question` | string |  |
| `examples[].sql` | string |  |

### `run_analytics_query`

**Executar consulta analítica**

Effect: Read-only

Run one read-only PostgreSQL 16 query (SELECT, or WITH … SELECT) over the Helios analytics views and return its rows. It is how you find records: look one up, with the uuids the write tools take, starting from describe_analytics_schema's lookup examples, and analyze many records: totals, counts, rankings, trends by period and comparisons. Call describe_analytics_schema first: only the views and columns it lists exist, and it gives the aggregation rules and verified examples. Every view holds only one company's rows, so never filter by company. Allowed: joins, CTEs, subqueries, GROUP BY/ROLLUP, window functions, FILTER, CASE, and aggregate, math, text, date/time, array and JSON functions. Not allowed: more than one statement, writes, DDL, SET, locking clauses, system functions and catalogs. Qualify a column only with the name or alias of a FROM item of the query or of an enclosing query. To use a set-returning function's columns with a prefix, name them: generate_series(…) AS g(day), then g.day; any other expression in FROM needs an alias, as in CAST('2026-01-01' AS date) AS d(day). Aggregate in SQL and return few rows: results stop at maxRows (default 200, max 1000) and at a size limit, and truncated=true means the rows are incomplete, so never total them yourself. Money is BRL as exact decimal strings: quote stored totals instead of recomputing them. Dates and times are Brasília time (-03). The text content shows CPF/CNPJ, UC numbers, phones and CEPs formatted for reading, and e-mails in lowercase (format in columns says which); structuredContent keeps the stored values, the form to filter with. Every result has dataAsOf and freshness: say how current an analysis is; right after a write, or before acting on a record, only a current result counts. Only administrators can use it (ForbiddenError otherwise). companyUuid: only super administrators may pass another company's uuid (from describe_analytics_schema); for anyone else it is NotFoundError. Run queries one at a time. Errors: ValidationError when the query is rejected or fails (message, hint and position included: fix it and retry), ServerError when analytics is unavailable or not enabled, its data is too far behind, or too many calls were made (retry later; meanwhile lookups are unavailable: point the user to Helios web, never guess a uuid).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sql` | string | yes | One read-only PostgreSQL SELECT (WITH … SELECT allowed) over the views describe_analytics_schema lists. No other statements. |
| `companyUuid` | string (uuid) | no | Company to query. Omit for your own company. Only super administrators may pass another company's uuid, taken from describe_analytics_schema's companies. |
| `maxRows` | integer | no | Maximum rows to return (default 200, max 1000). Aggregate in SQL instead of raising it. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `company` | object |  |
| `company.uuid` | string (uuid) |  |
| `company.name` | string |  |
| `columns` | array of object |  |
| `columns[].name` | string |  |
| `columns[].type` | string |  |
| `columns[].format` | `"document"`, `"consumer_unit_number"`, `"phone"`, `"zip_code"`, `"email"` | Optional. Set when the text content shows this column formatted for reading: document (CPF 123.456.789-09, CNPJ 12.345.678/0001-90), consumer_unit_number (1.234.567.890-12), phone (+55 (31) 99876-5432), zip_code (30123-456) or email (lowercase). structuredContent keeps the stored value (numbers as digits only): filter and join with that. |
| `rows` | array of array of any |  |
| `rowCount` | integer |  |
| `maxRows` | integer |  |
| `truncated` | boolean | True when more rows existed than were returned: the rows are incomplete. |
| `truncatedBy` | `"rows"`, `"size"` \| null |  |
| `dataAsOf` | string (date-time) | The data includes every change committed up to this moment (America/Sao_Paulo, ISO 8601). |
| `freshness` | `"current"`, `"behind"`, `"primary"` |  |
| `stalenessSeconds` | number \| null |  |
| `durationMs` | integer |  |

### `create_client`

**Cadastrar cliente**

Effect: Write (additive)

Create a client (individual or company) in the authenticated user's company. Document (CPF or CNPJ) must be digits only. Look the document up first: a duplicate document returns ConflictError. Returns the created client DTO. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Client legal name (individual full name or company name). |
| `document` | string | yes | Brazilian tax document, digits only. CPF = 11 digits (individuals), CNPJ = 14 digits (companies), with valid check digits. Reject anything formatted with dots/dashes. |
| `address1` | string | yes | Street address line, e.g. 'Rua das Flores, 123'. |
| `address2` | string | yes | Neighbourhood (bairro) of the client's address. Not an address complement: an apartment or suite goes in address1 after the number, separated by a comma, e.g. 'Rua das Flores, 123, Apto 101'. Required by domain rules — pass '-' if unknown. |
| `zipCode` | string | yes | Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `city` | string | yes | City name. |
| `state` | string | yes | Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `observation` | string \| null | no | Free-form notes (max 1000 chars). Optional. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `name` | string |  |
| `document` | string |  |
| `address1` | string |  |
| `address2` | string |  |
| `city` | string |  |
| `state` | string |  |
| `zipCode` | string |  |
| `observation` | string \| null |  |

### `update_client`

**Atualizar cliente**

Effect: Write (may change or delete existing data), idempotent

Update an existing client. Pass the client uuid plus the fields to change. The uuid must belong to your company; cross-company uuids return NotFoundError. Omitted fields are left unchanged. Returns the updated DTO. The client's name and document appear on the invoices and boletos of all of its UCs, so tell the user how many UCs the change reaches; a UC's CEMIG account holder (titular) is a UC field (update_consumer_unit). Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing client uuid to update. |
| `patch` | object | yes | Partial client fields to update. Only the keys provided are written; omitted keys are left unchanged. |
| `patch.name` | string | no | Client legal name (individual full name or company name). |
| `patch.document` | string | no | Brazilian tax document, digits only. CPF = 11 digits (individuals), CNPJ = 14 digits (companies), with valid check digits. Reject anything formatted with dots/dashes. |
| `patch.address1` | string | no | Street address line, e.g. 'Rua das Flores, 123'. |
| `patch.address2` | string | no | Neighbourhood (bairro) of the client's address. Not an address complement: an apartment or suite goes in address1 after the number, separated by a comma, e.g. 'Rua das Flores, 123, Apto 101'. Required by domain rules — pass '-' if unknown. |
| `patch.zipCode` | string | no | Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `patch.city` | string | no | City name. |
| `patch.state` | string | no | Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `patch.observation` | string \| null | no | Free-form notes (max 1000 chars). Optional. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `name` | string |  |
| `document` | string |  |
| `address1` | string |  |
| `address2` | string |  |
| `city` | string |  |
| `state` | string |  |
| `zipCode` | string |  |
| `observation` | string \| null |  |

### `create_installation`

**Cadastrar números de instalação**

Effect: Write (additive)

Create a bare installation record in the authenticated user's company: only CEMIG's two numbers. Provide installationNumber (10 digits, starts with 3), consumerUnitNumber (the UC number, 1-15 digits), or both — at least one is required. Returns the DTO. Users who say "cadastrar instalação" almost always mean a consumer unit (UC) with that number: use create_consumer_unit, which creates or reuses the installation in the same call. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `installationNumber` | string | no | CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `consumerUnitNumber` | string | no | CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `installationNumber` | string \| null |  |
| `consumerUnitNumber` | string \| null |  |

### `update_installation`

**Corrigir números da instalação**

Effect: Write (may change or delete existing data), idempotent

Correct the CEMIG numbers of an installation record by uuid: installationNumber (10 digits, starts with 3) and consumerUnitNumber (the UC number), its only fields. "Instalação" is CEMIG's record of the installation and UC numbers, but users also say it for the UC, and a number may be a UC's or a plant's (UG, "usina"). So the address, holder (titular), due day, discounts, notification channel or active flag "da instalação" belong to the UC: change them with update_consumer_unit. Say which of the two numbers changes, from what to what. The uuid must belong to your company; cross-company uuids return NotFoundError. Pass null in patch to clear a value, or omit a key to leave it unchanged. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing installation uuid to update. |
| `patch` | object | yes | Fields to update. Pass `null` to explicitly clear a value, or omit a key to leave it unchanged. |
| `patch.installationNumber` | string \| null | no |  |
| `patch.consumerUnitNumber` | string \| null | no |  |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `installationNumber` | string \| null |  |
| `consumerUnitNumber` | string \| null |  |

### `create_consumer_unit`

**Cadastrar unidade consumidora**

Effect: Write (additive)

Create a consumer unit (UC): what users mean by "cadastrar instalação", "nova UC" or "nova conta". The UC must be attached to both a client and an installation (the CEMIG record of its installation and UC numbers). For each, you may pass `existing` (uuid — must belong to your company), `create` (full data, will fail if a unique key collides), or `find_or_create` (lookup by document for clients / installationNumber or consumerUnitNumber for installations; create with `data` if not found). All writes happen inside a single transaction. If the resolved installation already has a consumer unit attached, returns ConflictError (1:1 constraint). If a generation unit (plant, "usina") sits on it, returns ConflictError and writes nothing: a consumer unit can't use a plant's installation, so tell the user the number is a plant's. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `client` | object | yes | How to resolve the client this consumer unit belongs to. `existing` references a known uuid; `create` always creates a new client (errors if document already exists); `find_or_create` first looks up by document — if found, reuses it; otherwise creates from `data`. |
| `client.mode` | `"existing"` | yes |  |
| `client.uuid` | string (uuid) | yes | Only when `mode` is `"existing"`. UUID of an existing client owned by your company. |
| `client.mode` | `"create"` | yes |  |
| `client.data` | object | yes | Only when `mode` is `"create"`. |
| `client.data.name` | string | yes | Only when `mode` is `"create"`. Client legal name (individual full name or company name). |
| `client.data.document` | string | yes | Only when `mode` is `"create"`. Brazilian tax document, digits only. CPF = 11 digits (individuals), CNPJ = 14 digits (companies), with valid check digits. Reject anything formatted with dots/dashes. |
| `client.data.address1` | string | yes | Only when `mode` is `"create"`. Street address line, e.g. 'Rua das Flores, 123'. |
| `client.data.address2` | string | yes | Only when `mode` is `"create"`. Neighbourhood (bairro) of the client's address. Not an address complement: an apartment or suite goes in address1 after the number, separated by a comma, e.g. 'Rua das Flores, 123, Apto 101'. Required by domain rules — pass '-' if unknown. |
| `client.data.zipCode` | string | yes | Only when `mode` is `"create"`. Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `client.data.city` | string | yes | Only when `mode` is `"create"`. City name. |
| `client.data.state` | string | yes | Only when `mode` is `"create"`. Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `client.data.observation` | string \| null | no | Only when `mode` is `"create"`. Free-form notes (max 1000 chars). Optional. |
| `client.mode` | `"find_or_create"` | yes |  |
| `client.lookup` | object | yes | Only when `mode` is `"find_or_create"`. Lookup by tax document, digits only. |
| `client.lookup.document` | string | yes | Only when `mode` is `"find_or_create"`. Brazilian tax document to match exactly, digits only: CPF (11) or CNPJ (14). |
| `client.data` | object | yes | Only when `mode` is `"find_or_create"`. |
| `client.data.name` | string | yes | Only when `mode` is `"find_or_create"`. Client legal name (individual full name or company name). |
| `client.data.document` | string | yes | Only when `mode` is `"find_or_create"`. Brazilian tax document, digits only. CPF = 11 digits (individuals), CNPJ = 14 digits (companies), with valid check digits. Reject anything formatted with dots/dashes. |
| `client.data.address1` | string | yes | Only when `mode` is `"find_or_create"`. Street address line, e.g. 'Rua das Flores, 123'. |
| `client.data.address2` | string | yes | Only when `mode` is `"find_or_create"`. Neighbourhood (bairro) of the client's address. Not an address complement: an apartment or suite goes in address1 after the number, separated by a comma, e.g. 'Rua das Flores, 123, Apto 101'. Required by domain rules — pass '-' if unknown. |
| `client.data.zipCode` | string | yes | Only when `mode` is `"find_or_create"`. Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `client.data.city` | string | yes | Only when `mode` is `"find_or_create"`. City name. |
| `client.data.state` | string | yes | Only when `mode` is `"find_or_create"`. Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `client.data.observation` | string \| null | no | Only when `mode` is `"find_or_create"`. Free-form notes (max 1000 chars). Optional. |
| `installation` | object | yes | How to resolve the installation this consumer unit belongs to. `existing` references a known uuid; `create` always creates a new installation; `find_or_create` looks up by installationNumber or consumerUnitNumber; if found, reuses it (errors with ConflictError if a consumer unit is already attached); otherwise creates from `data`. |
| `installation.mode` | `"existing"` | yes |  |
| `installation.uuid` | string (uuid) | yes | Only when `mode` is `"existing"`. |
| `installation.mode` | `"create"` | yes |  |
| `installation.data` | object | yes | Only when `mode` is `"create"`. |
| `installation.data.installationNumber` | string | no | Only when `mode` is `"create"`. CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `installation.data.consumerUnitNumber` | string | no | Only when `mode` is `"create"`. CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |
| `installation.mode` | `"find_or_create"` | yes |  |
| `installation.lookup` | object | yes | Only when `mode` is `"find_or_create"`. |
| `installation.lookup.installationNumber` | string | no | Only when `mode` is `"find_or_create"`. CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `installation.lookup.consumerUnitNumber` | string | no | Only when `mode` is `"find_or_create"`. CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |
| `installation.data` | object | yes | Only when `mode` is `"find_or_create"`. |
| `installation.data.installationNumber` | string | no | Only when `mode` is `"find_or_create"`. CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `installation.data.consumerUnitNumber` | string | no | Only when `mode` is `"find_or_create"`. CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |
| `consumerUnit` | object | yes |  |
| `consumerUnit.holderName` | string | yes | Account holder full legal name. |
| `consumerUnit.holderDocument` | string | yes | Account holder tax document, digits only. CPF (11) or CNPJ (14). |
| `consumerUnit.active` | boolean | yes | Whether the consumer unit is currently active and should be billed; the monthly invoice run skips an inactive one. Users say "desativar" or "cancelar a UC" for false. |
| `consumerUnit.dueDay` | integer | yes | Day of month payment is due (1-30), for the invoices generated from now on; existing invoices and their boletos keep their due date. |
| `consumerUnit.dueNextMonth` | boolean | yes | True if the due day refers to the month following the billing period. |
| `consumerUnit.discountPercentage` | number | yes | Recurring discount percentage applied to billing (0-100, up to 2 decimals). At 100 the monthly invoice run generates no invoice for the unit. |
| `consumerUnit.promptPaymentDiscount` | boolean | yes | Enables an extra discount when the customer pays before the due date. Users call it "desconto de adimplência"; the invoice PDF shows it as "Desconto Adicional". |
| `consumerUnit.promptPaymentDiscountPercentage` | number \| null | no | Prompt-payment discount percentage. Required (non-null) when promptPaymentDiscount is true. |
| `consumerUnit.includeCemigBilling` | boolean | yes | Whether the CEMIG bill is included in the consolidated invoice. |
| `consumerUnit.hasOwnGeneration` | boolean | yes | True if the consumer unit has its own generation. |
| `consumerUnit.notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` | yes | Channel for this consumer unit's invoice notifications (the invoice with its boleto, due-date reminders, overdue notices): email, WhatsApp or both. They go to every contact linked to the consumer unit (its contactLinks, primary or not), by each contact's email or phone; the holder receives them only if linked as a contact. |
| `consumerUnit.address1` | string | yes | Street address line. |
| `consumerUnit.address2` | string | yes | Neighbourhood (bairro), printed on invoice PDFs and sent to the bank as the boleto payer's district. Not an address complement: an apartment or suite goes in address1 after the number, separated by a comma, e.g. 'Rua das Flores, 123, Apto 101'. Pass '-' if unknown. |
| `consumerUnit.zipCode` | string | yes | Brazilian CEP, 8 digits, no dash. |
| `consumerUnit.city` | string | yes |  |
| `consumerUnit.state` | string | yes | Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `holderName` | string |  |
| `holderDocument` | string |  |
| `active` | boolean |  |
| `dueDay` | integer |  |
| `dueNextMonth` | boolean |  |
| `discountPercentage` | number |  |
| `promptPaymentDiscount` | boolean |  |
| `promptPaymentDiscountPercentage` | number \| null |  |
| `includeCemigBilling` | boolean |  |
| `hasOwnGeneration` | boolean |  |
| `notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` |  |
| `address1` | string |  |
| `address2` | string |  |
| `city` | string |  |
| `state` | string |  |
| `zipCode` | string |  |
| `installation` | object |  |
| `installation.uuid` | string (uuid) |  |
| `installation.installationNumber` | string \| null |  |
| `installation.consumerUnitNumber` | string \| null |  |
| `client` | object |  |
| `client.uuid` | string (uuid) |  |
| `client.name` | string |  |
| `client.document` | string |  |
| `contactLinks` | array of object |  |
| `contactLinks[].contact` | object |  |
| `contactLinks[].contact.uuid` | string (uuid) |  |
| `contactLinks[].contact.name` | string |  |
| `contactLinks[].contact.email` | string (email) \| null |  |
| `contactLinks[].contact.phone` | string |  |
| `contactLinks[].isPrimary` | boolean |  |

### `update_consumer_unit`

**Atualizar unidade consumidora**

Effect: Write (may change or delete existing data), idempotent

Update a consumer unit (UC), which users also call the "instalação", by `uuid` (must belong to your company; cross-company uuids return NotFoundError). Pass partial fields in `patch`. They are standing settings for the invoices generated from now on; one existing invoice's due date or discount changes only in the web app, so when the user doesn't say, ask "só desta fatura ou das próximas?". The installation and UC numbers belong to the installation record (update_installation), and the name and document on invoices and boletos to the client (update_client). To reassign to a different client or installation, set `clientUuid` / `installationUuid` — both must belong to your company. Reassigning to an installation a generation unit (plant) sits on returns ConflictError. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes |  |
| `patch` | object | yes |  |
| `patch.holderName` | string | no | Account holder full legal name. |
| `patch.holderDocument` | string | no | Account holder tax document, digits only. CPF (11) or CNPJ (14). |
| `patch.active` | boolean | no | Whether the consumer unit is currently active and should be billed; the monthly invoice run skips an inactive one. Users say "desativar" or "cancelar a UC" for false. |
| `patch.dueDay` | integer | no | Day of month payment is due (1-30), for the invoices generated from now on; existing invoices and their boletos keep their due date. |
| `patch.dueNextMonth` | boolean | no | True if the due day refers to the month following the billing period. |
| `patch.discountPercentage` | number | no | Recurring discount percentage applied to billing (0-100, up to 2 decimals). At 100 the monthly invoice run generates no invoice for the unit. |
| `patch.promptPaymentDiscount` | boolean | no | Enables an extra discount when the customer pays before the due date. Users call it "desconto de adimplência"; the invoice PDF shows it as "Desconto Adicional". |
| `patch.promptPaymentDiscountPercentage` | number \| null | no | Prompt-payment discount percentage. Required (non-null) when promptPaymentDiscount is true. |
| `patch.includeCemigBilling` | boolean | no | Whether the CEMIG bill is included in the consolidated invoice. |
| `patch.hasOwnGeneration` | boolean | no | True if the consumer unit has its own generation. |
| `patch.notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` | no | Channel for this consumer unit's invoice notifications (the invoice with its boleto, due-date reminders, overdue notices): email, WhatsApp or both. They go to every contact linked to the consumer unit (its contactLinks, primary or not), by each contact's email or phone; the holder receives them only if linked as a contact. |
| `patch.address1` | string | no | Street address line. |
| `patch.address2` | string | no | Neighbourhood (bairro), printed on invoice PDFs and sent to the bank as the boleto payer's district. Not an address complement: an apartment or suite goes in address1 after the number, separated by a comma, e.g. 'Rua das Flores, 123, Apto 101'. Pass '-' if unknown. |
| `patch.zipCode` | string | no | Brazilian CEP, 8 digits, no dash. |
| `patch.city` | string | no |  |
| `patch.state` | string | no | Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `clientUuid` | string (uuid) | no | Reassign the consumer unit to a different client. |
| `installationUuid` | string (uuid) | no | Reassign the consumer unit to a different installation. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `holderName` | string |  |
| `holderDocument` | string |  |
| `active` | boolean |  |
| `dueDay` | integer |  |
| `dueNextMonth` | boolean |  |
| `discountPercentage` | number |  |
| `promptPaymentDiscount` | boolean |  |
| `promptPaymentDiscountPercentage` | number \| null |  |
| `includeCemigBilling` | boolean |  |
| `hasOwnGeneration` | boolean |  |
| `notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` |  |
| `address1` | string |  |
| `address2` | string |  |
| `city` | string |  |
| `state` | string |  |
| `zipCode` | string |  |
| `installation` | object |  |
| `installation.uuid` | string (uuid) |  |
| `installation.installationNumber` | string \| null |  |
| `installation.consumerUnitNumber` | string \| null |  |
| `client` | object |  |
| `client.uuid` | string (uuid) |  |
| `client.name` | string |  |
| `client.document` | string |  |
| `contactLinks` | array of object |  |
| `contactLinks[].contact` | object |  |
| `contactLinks[].contact.uuid` | string (uuid) |  |
| `contactLinks[].contact.name` | string |  |
| `contactLinks[].contact.email` | string (email) \| null |  |
| `contactLinks[].contact.phone` | string |  |
| `contactLinks[].isPrimary` | boolean |  |

### `create_contact`

**Cadastrar contato**

Effect: Write (additive)

Create a contact in the authenticated user's company. Phone accepts Brazilian (+55) or Portuguese (+351) formats and is stored as digits only. Email is optional and stored lowercased. Returns the created contact DTO. ConflictError if email or phone already exists in the company. Optionally pass `consumerUnits: [{ uuid, isPrimary? }]` to link the new contact to one or more consumer units in the same transaction; NotFoundError if any uuid is outside the caller's company, ConflictError if `isPrimary: true` collides with the existing primary contact of that unit. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Contact full name (min 2 chars). |
| `email` | string (email) \| null | no | Contact email address. Empty strings are treated as null. Stored lowercased. |
| `phone` | string | yes | Contact phone number. Accepts Brazilian or Portuguese formats; stored as digits-only with country code. |
| `consumerUnits` | array of object | no | Optional consumer-unit associations created in the same transaction as the contact. Duplicated uuids are rejected. Default: `[]`. |
| `consumerUnits[].uuid` | string (uuid) | yes | Consumer unit uuid. Must belong to the caller's company. |
| `consumerUnits[].isPrimary` | boolean | no | Mark this contact as the primary contact for the consumer unit. At most one primary contact is allowed per unit. Default: `false`. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `name` | string |  |
| `email` | string (email) \| null |  |
| `phone` | string |  |

### `update_contact`

**Atualizar contato**

Effect: Write (may change or delete existing data), idempotent

Update an existing contact. Pass the contact uuid plus at least one of: `patch` (fields to change), `consumerUnitsToAdd` (`[{ uuid, isPrimary? }]` to link or upsert isPrimary), or `consumerUnitsToRemove` (uuids to unlink). The uuid must belong to your company; cross-company uuids return NotFoundError. Pass `email: null` to clear the email; omitted patch fields are left unchanged. ConflictError if the new email or phone collides with another contact in the company or if `isPrimary: true` collides with the existing primary contact of that unit. `consumerUnitsToAdd` and `consumerUnitsToRemove` must not reference the same uuid. Returns the updated contact DTO. A new phone or email applies to every UC the contact is linked to, so name them all. "Tirar o contato da UC" means unlinking it with consumerUnitsToRemove, never delete_contact. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing contact uuid to update. |
| `patch` | object | no | Partial contact fields to update. Pass `null` on email to clear it; omitted keys are left unchanged. Must contain at least one field. |
| `patch.name` | string | no | Contact full name (min 2 chars). |
| `patch.email` | string (email) \| null | no | Contact email address. Empty strings are treated as null. Stored lowercased. |
| `patch.phone` | string | no | Contact phone number. Accepts Brazilian or Portuguese formats; stored as digits-only with country code. |
| `consumerUnitsToAdd` | array of object | no | Consumer-unit links to add or upsert. Existing links are updated when `isPrimary` changes. Default: `[]`. |
| `consumerUnitsToAdd[].uuid` | string (uuid) | yes | Consumer unit uuid. Must belong to the caller's company. |
| `consumerUnitsToAdd[].isPrimary` | boolean | no | Mark this contact as the primary contact for the consumer unit. At most one primary contact is allowed per unit. Default: `false`. |
| `consumerUnitsToRemove` | array of string (uuid) | no | Consumer-unit uuids to unlink from this contact. Idempotent: unknown uuids are ignored. Default: `[]`. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `name` | string |  |
| `email` | string (email) \| null |  |
| `phone` | string |  |

### `delete_contact`

**Excluir contato**

Effect: Write (may change or delete existing data)

Delete a contact in the authenticated user's company, from every UC it is linked to. Users who say "remover o contato" of one UC usually mean unlinking it: use update_contact with consumerUnitsToRemove, and delete only when they want the contact gone from every UC. The uuid must belong to your company; cross-company or unknown uuids return NotFoundError. Cascades to consumer-unit links and all notification history (WhatsApp + Email) via FK ON DELETE CASCADE. Irreversible and non-idempotent: a successful call returns `{ uuid }`; a second call with the same uuid returns NotFoundError because the row no longer exists. Before calling it, look the record up and resolve which one it is: if several match, list them and let the user choose; never pick one yourself. Then tell the user the record kind, its identifier, each field from old to new (each value, for a new record) and the scope, such as every UC a contact is linked to, and wait for their yes. Example: «Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?» To check the write afterwards, only a run_analytics_query result whose freshness is current counts.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing contact uuid to delete. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |

### `cancel_bank_slip`

**Cancelar boleto**

Effect: Write (may change or delete existing data), idempotent, acts outside Helios

Cancel one boleto of an invoice at its bank and mark its billing canceled in Helios; the invoice stays with its amounts, and no payment is recorded. "Cobrança" is the invoice (fatura, the Cobranças menu), but users also say it for one boleto. "Dar baixa" or "baixar" can mean a boleto cancel at the bank, a payment record (web app only: "Confirmar pagamento") or a boleto download (no tool does). When the user asks to cancel a "cobrança", or says "dar baixa" or "baixar", first ask one numbered question built from the records found, unless they said "cancelar" or "cancela" with "boleto". Example: «Sobre a "cobrança" de FEV/2026 da UC 3.012.345.678-90, você quer: (1) cancelar no banco o boleto Sicoob de R$ 250,00 com vencimento 10/03/2026 — a fatura continua no Helios e isso não pode ser desfeito; ou (2) excluir a fatura, o que hoje só é possível no Helios web?» If the client paid this boleto, don't cancel it: report its status. If they paid another way, the web app records it (Confirmar pagamento) once the open boleto is canceled: cancel it only if they then confirm. Irreversible: a canceled boleto can't be paid or restored; a reissue gets a new barcode. Always confirm the exact boleto and wait for the user's yes: «Vou cancelar no Sicoob o **boleto** de R$ 1.250,00 com vencimento 10/03/2026 da fatura de FEV/2026 (UC 3.012.345.678-90, Maria Exemplo, CPF 123.456.789-09). A fatura continua no Helios e esse boleto não poderá mais ser pago. Confirma?» Take both uuids from run_analytics_query's lookup of the invoice and its boletos and pick only a boleto whose cancel_allowed is true (cancel_reason says why not). After a `canceled` or `already_canceled` outcome, look the invoice up again and report any other boleto still allowed to cancel (still payable) or `processing` or `canceling` (may become payable). A repeated call is safe: it returns `already_canceled` without contacting the bank. Errors: NotFoundError when the invoice or billing is missing, another company's, or not on that invoice; ConflictError when the billing's status forbids canceling (paid, payment clearing, returned, being issued or being canceled) or the bank refused or doesn't know the boleto, with the reason; nothing changed, so don't retry it unchanged. ServerError when the bank couldn't be reached or the result is unclear: look the boleto up again (only a current result counts) before retrying.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `invoiceUuid` | string (uuid) | yes | Invoice (Cobrança) whose boleto to cancel. |
| `billingUuid` | string (uuid) | yes | The boleto's billing uuid, from run_analytics_query's lookup of the invoice with its boletos; it must belong to invoiceUuid. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `outcome` | `"canceled"`, `"already_canceled"` | `canceled`: the bank canceled the boleto now. `already_canceled`: it had already been canceled, in Helios (no bank call) or at the bank; a repeated call returns this. |
| `billing` | object | The billing whose boleto was canceled. |
| `billing.uuid` | string (uuid) |  |
| `billing.status` | `"processing"`, `"created"`, `"paid"`, `"canceling"`, `"canceled"`, `"expired"`, `"returned"`, `"updating"`, `"updated"`, `"processing_payment"` | The billing's status after the call: `canceled`. |
