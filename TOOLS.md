# Helios MCP tools

<!-- Generated from the Helios MCP server by `npm run generate:client-kit`. Do not edit by hand. -->

Server version: `1.2.0`

This catalog lists what the Helios MCP server offers to AI assistants.
Assistants read the same descriptions, parameter schemas and output
schemas directly from the server when they connect, so you never need to
paste them anywhere.
Every tool acts only on the data of the company you logged in with; in the analytics tools, a super administrator may choose another company.

## Tools

| Tool | Title | Effect |
|---|---|---|
| [`get_current_user`](#get_current_user) | Usuário conectado | Read-only |
| [`search_companies`](#search_companies) | Buscar empresas | Read-only |
| [`create_client`](#create_client) | Cadastrar cliente | Write (additive) |
| [`update_client`](#update_client) | Atualizar cliente | Write (may change or delete existing data), idempotent |
| [`search_clients`](#search_clients) | Buscar clientes | Read-only |
| [`create_installation`](#create_installation) | Cadastrar instalação | Write (additive) |
| [`update_installation`](#update_installation) | Atualizar instalação | Write (may change or delete existing data), idempotent |
| [`search_installations`](#search_installations) | Buscar instalações | Read-only |
| [`create_consumer_unit`](#create_consumer_unit) | Cadastrar unidade consumidora | Write (additive) |
| [`update_consumer_unit`](#update_consumer_unit) | Atualizar unidade consumidora | Write (may change or delete existing data), idempotent |
| [`search_consumer_units`](#search_consumer_units) | Buscar unidades consumidoras | Read-only |
| [`create_contact`](#create_contact) | Cadastrar contato | Write (additive) |
| [`update_contact`](#update_contact) | Atualizar contato | Write (may change or delete existing data), idempotent |
| [`delete_contact`](#delete_contact) | Excluir contato | Write (may change or delete existing data) |
| [`search_contacts`](#search_contacts) | Buscar contatos | Read-only |
| [`search_invoices`](#search_invoices) | Buscar cobranças | Read-only |
| [`get_invoice`](#get_invoice) | Detalhar cobrança | Read-only |
| [`cancel_bank_slip`](#cancel_bank_slip) | Cancelar boleto | Write (may change or delete existing data), idempotent, acts outside Helios |
| [`search_energy_bills`](#search_energy_bills) | Buscar contas de energia | Read-only |
| [`search_gd_reports`](#search_gd_reports) | Buscar relatórios GD | Read-only |
| [`search_generation_units`](#search_generation_units) | Buscar unidades geradoras | Read-only |
| [`get_generation_unit`](#get_generation_unit) | Detalhar unidade geradora | Read-only |
| [`describe_analytics_schema`](#describe_analytics_schema) | Descrever dados para análise | Read-only |
| [`run_analytics_query`](#run_analytics_query) | Executar consulta analítica | Read-only |

## Server instructions

Every assistant receives these instructions when it connects:

- Helios is the management system of a Brazilian solar energy company: clients, CEMIG installations, consumer units (UCs) and their contacts, with their billing and CEMIG data and read-only SQL analytics over them.
- Every tool operates within the authenticated user's company. A uuid from another company behaves exactly like a missing one (NotFoundError). The one exception: a super administrator may find other companies with search_companies and pass one's uuid to run_analytics_query.
- Call get_current_user first to confirm the connection and which company you are acting for.
- Search before creating: look clients up by document and installations by installation or consumer unit number. Create tools return ConflictError on duplicates.
- Formats: documents (CPF 11 digits, CNPJ 14 digits) and ZIP codes are digits only; installation numbers are 10 digits starting with 3; state codes are 2 uppercase letters.
- Billing data: invoices (Cobranças) and their boletos are read-only, except cancel_bank_slip, which cancels a boleto at its bank and can't be undone: call it only after the user explicitly confirms. Periods are MMM/YYYY with pt-BR months (FEV/2026); money is BRL as exact decimal strings, to quote as stored, never to recompute.
- Read-only CEMIG data: generation units (UGs), imported energy bills (contas de energia) and GD reports (generation, compensation and credit balance in kWh) per installation and period.
- Analytics (administrators only): for totals, rankings, trends or comparisons across many records, call describe_analytics_schema, then run_analytics_query with one read-only SELECT, one call at a time. When the user names the company to analyze, find its uuid with search_companies first. Aggregate in SQL; a truncated result is incomplete. Say how current the data is (dataAsOf). For one specific record, use the search_ and get_ tools.
- A failed call returns isError with the text `<ErrorType>: <message>`, where ErrorType is ValidationError, NotFoundError, ConflictError, ForbiddenError or ServerError. Messages are in Brazilian Portuguese and can be shown to the user as they are.
- Helios users are Brazilian: answer them in Brazilian Portuguese unless they write in another language.

## Tool reference

### `get_current_user`

**Usuário conectado**

Effect: Read-only

Return the authenticated Helios user and the company every other tool is scoped to. Call it first in a session to confirm the connection works and which company you are acting for.

No parameters.

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `user` | object |  |
| `user.uuid` | string (uuid) |  |
| `user.name` | string |  |
| `user.email` | string |  |
| `company` | object |  |
| `company.uuid` | string (uuid) |  |
| `company.name` | string |  |
| `company.cnpj` | string |  |

### `search_companies`

**Buscar empresas**

Effect: Read-only

Find Helios companies by name or CNPJ and return their uuids, for run_analytics_query's companyUuid when the user names a company instead of meaning their own. Super administrators see every company; other users see only their own (isCurrentUserCompany marks it). Name matching ignores case and accents, and every word given must appear in the company's legal name, in any order. If nothing matches, retry with the most distinctive single word or with the CNPJ. Calls count toward the analytics rate limits (ServerError: retry later). Returns an array, possibly empty.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | no | Words of the company name, in any order (each must appear in the legal name); case and accents are ignored. |
| `cnpj` | string | no | Exact CNPJ, 14 digits. |
| `limit` | integer | no | Maximum companies to return (default 20). |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].name` | string |  |
| `[].cnpj` | string |  |
| `[].city` | string |  |
| `[].state` | string |  |
| `[].isCurrentUserCompany` | boolean |  |

### `create_client`

**Cadastrar cliente**

Effect: Write (additive)

Create a client (individual or company) in the authenticated user's company. Document (CPF or CNPJ) must be digits only. Search by document first: a duplicate document returns ConflictError. Returns the created client DTO.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Client legal name (individual full name or company name). |
| `document` | string | yes | Brazilian tax document, digits only. CPF = 11 digits (individuals), CNPJ = 14 digits (companies), with valid check digits. Reject anything formatted with dots/dashes. |
| `address1` | string | yes | Street address line, e.g. 'Rua das Flores, 123'. |
| `address2` | string | yes | Address complement (apartment, suite, building). Required by domain rules — pass '-' if no complement is known. |
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

Update an existing client. Pass the client uuid plus the fields to change. The uuid must belong to your company; cross-company uuids return NotFoundError. Omitted fields are left unchanged. Returns the updated DTO.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing client uuid to update. |
| `patch` | object | yes | Partial client fields to update. Only the keys provided are written; omitted keys are left unchanged. |
| `patch.name` | string | no | Client legal name (individual full name or company name). |
| `patch.document` | string | no | Brazilian tax document, digits only. CPF = 11 digits (individuals), CNPJ = 14 digits (companies), with valid check digits. Reject anything formatted with dots/dashes. |
| `patch.address1` | string | no | Street address line, e.g. 'Rua das Flores, 123'. |
| `patch.address2` | string | no | Address complement (apartment, suite, building). Required by domain rules — pass '-' if no complement is known. |
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

### `search_clients`

**Buscar clientes**

Effect: Read-only

Search clients in the authenticated user's company. Combine filters to narrow the result; `document` is the most reliable lookup. Name is a substring match and may return many rows. Returns an array (possibly empty); cross-company uuids never appear in results.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | no | Exact uuid match. When set, returns at most one row. |
| `document` | string | no | Exact document match (digits only). Use this for an unambiguous lookup. |
| `name` | string | no | Substring match on client name. May return many rows — pair with `document` when possible. |
| `city` | string | no |  |
| `state` | string | no | Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `zipCode` | string | no | Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `limit` | integer | no |  |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].name` | string |  |
| `[].document` | string |  |
| `[].address1` | string |  |
| `[].address2` | string |  |
| `[].city` | string |  |
| `[].state` | string |  |
| `[].zipCode` | string |  |
| `[].observation` | string \| null |  |

### `create_installation`

**Cadastrar instalação**

Effect: Write (additive)

Create an installation in the authenticated user's company. Provide installationNumber (10 digits, starts with 3), consumerUnitNumber (1-15 digits), or both — at least one is required. Returns the DTO.

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

**Atualizar instalação**

Effect: Write (may change or delete existing data), idempotent

Update an installation by uuid. The uuid must belong to your company; cross-company uuids return NotFoundError. Pass null in patch to clear a value, or omit a key to leave it unchanged.

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

### `search_installations`

**Buscar instalações**

Effect: Read-only

Search installations in the authenticated user's company by uuid, installationNumber, or consumerUnitNumber. Returns an array (possibly empty); cross-company uuids never appear in results.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | no |  |
| `installationNumber` | string | no | CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `consumerUnitNumber` | string | no | CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |
| `limit` | integer | no |  |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].installationNumber` | string \| null |  |
| `[].consumerUnitNumber` | string \| null |  |

### `create_consumer_unit`

**Cadastrar unidade consumidora**

Effect: Write (additive)

Create a consumer unit (CU). The CU must be attached to both a client and an installation. For each, you may pass `existing` (uuid — must belong to your company), `create` (full data, will fail if a unique key collides), or `find_or_create` (lookup by document for clients / installationNumber or consumerUnitNumber for installations; create with `data` if not found). All writes happen inside a single transaction. If the resolved installation already has a consumer unit attached, returns ConflictError (1:1 constraint).

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
| `client.data.address2` | string | yes | Only when `mode` is `"create"`. Address complement (apartment, suite, building). Required by domain rules — pass '-' if no complement is known. |
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
| `client.data.address2` | string | yes | Only when `mode` is `"find_or_create"`. Address complement (apartment, suite, building). Required by domain rules — pass '-' if no complement is known. |
| `client.data.zipCode` | string | yes | Only when `mode` is `"find_or_create"`. Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `client.data.city` | string | yes | Only when `mode` is `"find_or_create"`. City name. |
| `client.data.state` | string | yes | Only when `mode` is `"find_or_create"`. Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `client.data.observation` | string \| null | no | Only when `mode` is `"find_or_create"`. Free-form notes (max 1000 chars). Optional. |
| `installation` | object | yes | How to resolve the installation this consumer unit belongs to. `existing` references a known uuid; `create` always creates a new installation; `find_or_create` looks up by installationNumber or consumerUnitNumber; if found, reuses it (errors with ConflictError if a CU is already attached); otherwise creates from `data`. |
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
| `consumerUnit.active` | boolean | yes | Whether the consumer unit is currently active and should be billed. |
| `consumerUnit.dueDay` | integer | yes | Day of month payment is due (1-30). |
| `consumerUnit.dueNextMonth` | boolean | yes | True if the due day refers to the month following the billing period. |
| `consumerUnit.discountPercentage` | number | yes | Recurring discount percentage applied to billing (0-100, up to 2 decimals). |
| `consumerUnit.promptPaymentDiscount` | boolean | yes | Enables an extra discount when the customer pays before the due date. |
| `consumerUnit.promptPaymentDiscountPercentage` | number \| null | no | Prompt-payment discount percentage. Required (non-null) when promptPaymentDiscount is true. |
| `consumerUnit.includeCemigBilling` | boolean | yes | Whether the CEMIG bill is included in the consolidated invoice. |
| `consumerUnit.hasOwnGeneration` | boolean | yes | True if the consumer unit has its own generation. |
| `consumerUnit.notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` | yes | How invoices/notifications are delivered to the holder. |
| `consumerUnit.address1` | string | yes | Street address line. |
| `consumerUnit.address2` | string | yes | Address complement (apartment, suite, building). Pass '-' if unknown. |
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

Update a consumer unit by `uuid` (must belong to your company; cross-company uuids return NotFoundError). Pass partial fields in `patch`. To reassign to a different client or installation, set `clientUuid` / `installationUuid` — both must belong to your company.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes |  |
| `patch` | object | yes |  |
| `patch.holderName` | string | no | Account holder full legal name. |
| `patch.holderDocument` | string | no | Account holder tax document, digits only. CPF (11) or CNPJ (14). |
| `patch.active` | boolean | no | Whether the consumer unit is currently active and should be billed. |
| `patch.dueDay` | integer | no | Day of month payment is due (1-30). |
| `patch.dueNextMonth` | boolean | no | True if the due day refers to the month following the billing period. |
| `patch.discountPercentage` | number | no | Recurring discount percentage applied to billing (0-100, up to 2 decimals). |
| `patch.promptPaymentDiscount` | boolean | no | Enables an extra discount when the customer pays before the due date. |
| `patch.promptPaymentDiscountPercentage` | number \| null | no | Prompt-payment discount percentage. Required (non-null) when promptPaymentDiscount is true. |
| `patch.includeCemigBilling` | boolean | no | Whether the CEMIG bill is included in the consolidated invoice. |
| `patch.hasOwnGeneration` | boolean | no | True if the consumer unit has its own generation. |
| `patch.notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` | no | How invoices/notifications are delivered to the holder. |
| `patch.address1` | string | no | Street address line. |
| `patch.address2` | string | no | Address complement (apartment, suite, building). Pass '-' if unknown. |
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

### `search_consumer_units`

**Buscar unidades consumidoras**

Effect: Read-only

Search consumer units in the authenticated user's company. Combine filters to narrow results: installationNumber, consumerUnitNumber, holderDocument, or clientDocument are exact matches. clientName, holderName, city, address1, address2 are case-insensitive substring matches. Cross-company uuids never match. Returns an array (possibly empty).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | no |  |
| `installationNumber` | string | no | CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `consumerUnitNumber` | string | no | CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |
| `clientName` | string | no |  |
| `clientDocument` | string | no | Brazilian tax document to match exactly, digits only: CPF (11) or CNPJ (14). |
| `holderName` | string | no |  |
| `holderDocument` | string | no | Brazilian tax document to match exactly, digits only: CPF (11) or CNPJ (14). |
| `city` | string | no |  |
| `state` | string | no | Two-letter Brazilian state code, uppercase. Currently the system only stores MG. |
| `zipCode` | string | no | Brazilian CEP, 8 digits, no dash. Example: 30123456. |
| `address1` | string | no |  |
| `address2` | string | no |  |
| `limit` | integer | no |  |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].holderName` | string |  |
| `[].holderDocument` | string |  |
| `[].active` | boolean |  |
| `[].dueDay` | integer |  |
| `[].dueNextMonth` | boolean |  |
| `[].discountPercentage` | number |  |
| `[].promptPaymentDiscount` | boolean |  |
| `[].promptPaymentDiscountPercentage` | number \| null |  |
| `[].includeCemigBilling` | boolean |  |
| `[].hasOwnGeneration` | boolean |  |
| `[].notificationChannel` | `"email"`, `"whatsapp"`, `"email_and_whatsapp"` |  |
| `[].address1` | string |  |
| `[].address2` | string |  |
| `[].city` | string |  |
| `[].state` | string |  |
| `[].zipCode` | string |  |
| `[].installation` | object |  |
| `[].installation.uuid` | string (uuid) |  |
| `[].installation.installationNumber` | string \| null |  |
| `[].installation.consumerUnitNumber` | string \| null |  |
| `[].client` | object |  |
| `[].client.uuid` | string (uuid) |  |
| `[].client.name` | string |  |
| `[].client.document` | string |  |
| `[].contactLinks` | array of object |  |
| `[].contactLinks[].contact` | object |  |
| `[].contactLinks[].contact.uuid` | string (uuid) |  |
| `[].contactLinks[].contact.name` | string |  |
| `[].contactLinks[].contact.email` | string (email) \| null |  |
| `[].contactLinks[].contact.phone` | string |  |
| `[].contactLinks[].isPrimary` | boolean |  |

### `create_contact`

**Cadastrar contato**

Effect: Write (additive)

Create a contact in the authenticated user's company. Phone accepts Brazilian (+55) or Portuguese (+351) formats and is stored as digits only. Email is optional and stored lowercased. Returns the created contact DTO. ConflictError if email or phone already exists in the company. Optionally pass `consumerUnits: [{ uuid, isPrimary? }]` to link the new contact to one or more consumer units in the same transaction; NotFoundError if any uuid is outside the caller's company, ConflictError if `isPrimary: true` collides with the existing primary contact of that unit.

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

Update an existing contact. Pass the contact uuid plus at least one of: `patch` (fields to change), `consumerUnitsToAdd` (`[{ uuid, isPrimary? }]` to link or upsert isPrimary), or `consumerUnitsToRemove` (uuids to unlink). The uuid must belong to your company; cross-company uuids return NotFoundError. Pass `email: null` to clear the email; omitted patch fields are left unchanged. ConflictError if the new email or phone collides with another contact in the company or if `isPrimary: true` collides with the existing primary contact of that unit. `consumerUnitsToAdd` and `consumerUnitsToRemove` must not reference the same uuid. Returns the updated contact DTO.

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

Delete a contact in the authenticated user's company. The uuid must belong to your company; cross-company or unknown uuids return NotFoundError. Cascades to consumer-unit links and all notification history (WhatsApp + Email) via FK ON DELETE CASCADE. Irreversible and non-idempotent: a successful call returns `{ uuid }`; a second call with the same uuid returns NotFoundError because the row no longer exists.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing contact uuid to delete. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |

### `search_contacts`

**Buscar contatos**

Effect: Read-only

Search contacts in the authenticated user's company. Combine filters to narrow the result; `email` and `phone` are the most reliable lookups (phone is matched as digits-only). Name is a substring match and may return many rows. Pass `consumerUnitUuid` to restrict the result to contacts linked to that consumer unit via the join table. Returns an array (possibly empty); cross-company uuids never appear in results.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | no | Exact uuid match. When set, returns at most one row. |
| `email` | string | no | Exact email match (case-insensitive). Use this for an unambiguous lookup. |
| `phone` | string | no | Phone lookup. Digits-only substring match against the stored unformatted phone. |
| `name` | string | no | Substring match on contact name. May return many rows — pair with `email` or `phone` when possible. |
| `consumerUnitUuid` | string (uuid) | no | Restrict results to contacts linked to this consumer unit via the consumer-unit/contact join table. Cross-company uuids return no rows. |
| `limit` | integer | no | Default: `50`. |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].name` | string |  |
| `[].email` | string (email) \| null |  |
| `[].phone` | string |  |

### `search_invoices`

**Buscar cobranças**

Effect: Read-only

Search invoices (Cobranças, Helios's monthly charge to a client) in the authenticated user's company, newest period first. Filter by consumer unit uuid, client uuid, period (MMM/YYYY, e.g. FEV/2026) or status: the latest boleto's status, or `unbilled` for invoices without a boleto. To start from an installation or UC number, find the consumer unit with search_consumer_units first. Returns an array (possibly empty); another company's uuids match nothing.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `consumerUnitUuid` | string (uuid) | no | Only invoices of this consumer unit (UC). |
| `clientUuid` | string (uuid) | no | Only invoices billed to this client. |
| `period` | string | no | Billing period MMM/YYYY with an uppercase pt-BR month abbreviation (JAN FEV MAR ABR MAI JUN JUL AGO SET OUT NOV DEZ). Example: FEV/2026. |
| `status` | `"processing"`, `"created"`, `"paid"`, `"canceling"`, `"canceled"`, `"expired"`, `"returned"`, `"updating"`, `"updated"`, `"processing_payment"`, `"unbilled"` | no | Status of the latest boleto, or `unbilled` for invoices without any boleto. |
| `limit` | integer | no | Maximum rows to return, 1-100 (default 50). |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].period` | string | Billing period, MMM/YYYY. |
| `[].dueDate` | string | Due date, YYYY-MM-DD. |
| `[].clientUuid` | string (uuid) |  |
| `[].clientName` | string |  |
| `[].consumerUnitUuid` | string (uuid) |  |
| `[].holderName` | string | Consumer unit holder name. |
| `[].installationNumber` | string \| null |  |
| `[].consumerUnitNumber` | string \| null |  |
| `[].totalBrl` | string | Invoice total as stored, BRL as an exact decimal string ("123.45"). |
| `[].billingStatus` | `"processing"`, `"created"`, `"paid"`, `"canceling"`, `"canceled"`, `"expired"`, `"returned"`, `"updating"`, `"updated"`, `"processing_payment"` \| null | Status of the most recently issued boleto (by creation time), or null when no boleto was ever issued. |

### `get_invoice`

**Detalhar cobrança**

Effect: Read-only

Get one invoice by uuid with its stored line amounts (compensation, discounts, CEMIG bill, surplus B wire, extra items, total), the generation units that compensated it and every boleto issued for it, newest first. Each boleto in `billings` carries `cancel`: whether cancel_bank_slip can cancel it now and, if not, the reason in Brazilian Portuguese. Amounts are BRL decimal strings exactly as stored; don't recompute them. Read-only: it never issues or syncs boletos. A missing or another company's uuid returns NotFoundError.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Invoice uuid, e.g. from search_invoices. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `period` | string | Billing period, MMM/YYYY. |
| `dueDate` | string | Due date, YYYY-MM-DD. |
| `client` | object |  |
| `client.uuid` | string (uuid) |  |
| `client.name` | string |  |
| `client.document` | string | CPF or CNPJ, digits only. |
| `consumerUnit` | object |  |
| `consumerUnit.uuid` | string (uuid) |  |
| `consumerUnit.holderName` | string |  |
| `consumerUnit.installationUuid` | string (uuid) |  |
| `consumerUnit.installationNumber` | string \| null |  |
| `consumerUnit.consumerUnitNumber` | string \| null |  |
| `tariff` | string | Tariff in BRL per kWh, 8 decimal places. |
| `consumptionKwh` | string |  |
| `compensationKwh` | string |  |
| `balanceKwh` | string |  |
| `balancePeriod` | string | Period the kWh balance refers to. |
| `discountPercentage` | string |  |
| `discountBrl` | string | Discount on the compensation amount, BRL as an exact decimal string ("123.45"). |
| `promptPaymentDiscountPercentage` | string |  |
| `promptPaymentDiscountBrl` | string | Discount granted when paid by the due date, already excluded from totalBrl, BRL as an exact decimal string ("123.45"). |
| `compensationBrl` | string | Compensated energy amount, BRL as an exact decimal string ("123.45"). |
| `cemigBillBrl` | string | CEMIG bill amount, BRL as an exact decimal string ("123.45"). |
| `includeCemigBill` | boolean | Whether cemigBillBrl is charged as part of totalBrl. |
| `surplusBWireBrl` | string | Surplus B wire amount, subtracted from the total, BRL as an exact decimal string ("123.45"). |
| `totalBrl` | string | Invoice total as stored, BRL as an exact decimal string ("123.45"). |
| `extraItems` | array of object |  |
| `extraItems[].uuid` | string (uuid) |  |
| `extraItems[].name` | string |  |
| `extraItems[].amountBrl` | string | Item amount (negative for a credit), BRL as an exact decimal string ("123.45"). |
| `generationUnits` | array of object |  |
| `generationUnits[].uuid` | string (uuid) |  |
| `generationUnits[].name` | string |  |
| `generationUnits[].compensationKwh` | string | kWh this generation unit compensated on the invoice. |
| `generationUnits[].surplusBWireBrl` | string | Surplus B wire amount attributed to this unit, BRL as an exact decimal string ("123.45"). |
| `billingStatus` | `"processing"`, `"created"`, `"paid"`, `"canceling"`, `"canceled"`, `"expired"`, `"returned"`, `"updating"`, `"updated"`, `"processing_payment"` \| null | Status of the most recently issued boleto (by creation time), or null when no boleto was ever issued. |
| `billings` | array of object | Every boleto issued for the invoice, newest first. |
| `billings[].uuid` | string (uuid) |  |
| `billings[].bank` | `"btg"`, `"cora"`, `"sicoob"` |  |
| `billings[].status` | `"processing"`, `"created"`, `"paid"`, `"canceling"`, `"canceled"`, `"expired"`, `"returned"`, `"updating"`, `"updated"`, `"processing_payment"` |  |
| `billings[].amount` | string | Amount due when paid by the due date, BRL as an exact decimal string ("123.45"). |
| `billings[].amountPaid` | string | Amount paid so far, BRL as an exact decimal string ("123.45"). |
| `billings[].paidAt` | string \| null | Payment date, YYYY-MM-DD. |
| `billings[].dueDate` | string | Boleto due date, YYYY-MM-DD. |
| `billings[].manuallyCreated` | boolean |  |
| `billings[].createdAt` | string | When the boleto was issued (timestamp). |
| `billings[].cancel` | object | Whether cancel_bank_slip can cancel this boleto now, judged from its status. When it can't, reason says why, in Brazilian Portuguese. |
| `billings[].cancel.allowed` | boolean | Whether cancel_bank_slip can cancel this boleto now. |
| `billings[].cancel.reason` | string \| null | Why it can't, in Brazilian Portuguese, to show the user; null when allowed. |

### `cancel_bank_slip`

**Cancelar boleto**

Effect: Write (may change or delete existing data), idempotent, acts outside Helios

Cancel an invoice's boleto at its bank and mark its billing canceled in Helios. Irreversible: a canceled boleto can't be paid or restored, and a boleto issued again for the invoice gets a new barcode and payment line. Only call it after the user explicitly confirmed this cancellation: show the invoice, the boleto's bank, amount and due date, and ask first. Take both uuids from get_invoice (`billings`, newest first) and pick only a boleto whose `cancel.allowed` is true; `cancel.reason` says why another can't be canceled. After a `canceled` or `already_canceled` outcome, call get_invoice again and tell the user about any other boleto of the invoice whose `cancel.allowed` is still true, since it can still be paid, or whose status is `processing` or `canceling`, since it may still be or become payable. A billing that is already canceled returns outcome `already_canceled` without contacting the bank, so a repeated call is safe. Errors: NotFoundError when the invoice or billing is missing, belongs to another company, or the billing isn't on that invoice; ConflictError when the billing's status forbids canceling (paid, payment clearing, returned, still being issued or being canceled) or the bank refused or doesn't know the boleto, with the reason in the message; nothing changed, so don't retry it unchanged. ServerError when the bank couldn't be reached, was unavailable or the result is unclear: call get_invoice again and check the billing's status before retrying.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `invoiceUuid` | string (uuid) | yes | Invoice (Cobrança) whose boleto to cancel. |
| `billingUuid` | string (uuid) | yes | The boleto's billing uuid, from get_invoice's `billings`; it must belong to invoiceUuid. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `outcome` | `"canceled"`, `"already_canceled"` | `canceled`: the bank canceled the boleto now. `already_canceled`: it had already been canceled, in Helios (no bank call) or at the bank; a repeated call returns this. |
| `billing` | object | The billing whose boleto was canceled. |
| `billing.uuid` | string (uuid) |  |
| `billing.status` | `"processing"`, `"created"`, `"paid"`, `"canceling"`, `"canceled"`, `"expired"`, `"returned"`, `"updating"`, `"updated"`, `"processing_payment"` | The billing's status after the call: `canceled`. |

### `search_energy_bills`

**Buscar contas de energia**

Effect: Read-only

Search imported CEMIG energy bills (contas de energia) in the authenticated user's company, newest period first. Filter by installation uuid, consumer unit uuid (its installation's bills), generation unit uuid, installation number or period (MMM/YYYY). A bill imported from several generation units' exports appears once per generation unit (see generationUnitUuid). Amounts are BRL and kWh decimal strings exactly as imported; don't recompute them. Returns an array (possibly empty); another company's uuids match nothing.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `installationUuid` | string (uuid) | no | Only bills of this installation. |
| `consumerUnitUuid` | string (uuid) | no | Only bills of this consumer unit's installation. |
| `generationUnitUuid` | string (uuid) | no | Only bills imported from this generation unit's export. |
| `installationNumber` | string | no | Exact CEMIG installation number printed on the bill, 10 digits starting with 3. |
| `period` | string | no | Billing period MMM/YYYY with an uppercase pt-BR month abbreviation (JAN FEV MAR ABR MAI JUN JUL AGO SET OUT NOV DEZ). Example: FEV/2026. |
| `limit` | integer | no | Maximum rows to return, 1-100 (default 50). |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].period` | string | Billing period, MMM/YYYY. |
| `[].installationUuid` | string (uuid) | Installation the bill belongs to. |
| `[].installationNumber` | string |  |
| `[].consumerUnitNumber` | string \| null |  |
| `[].generationUnitUuid` | string (uuid) | Generation unit whose CEMIG export the bill was imported from. |
| `[].readingStart` | string | YYYY-MM-DD. |
| `[].readingEnd` | string | YYYY-MM-DD. |
| `[].nextReadingDate` | string | YYYY-MM-DD. |
| `[].dueDate` | string | CEMIG due date, YYYY-MM-DD. |
| `[].total` | string | Bill total excluding compensation values, BRL as an exact decimal string ("123.45"). |
| `[].tariff` | string \| null | Tariff in BRL per kWh, 8 decimal places. |
| `[].installationCategory` | string |  |
| `[].installationClass` | `"Monofásico"`, `"Bifásico"`, `"Trifásico"` |  |
| `[].installationSubClass1` | string |  |
| `[].installationSubClass2` | string \| null |  |
| `[].tariffModel` | string |  |
| `[].availabilityTotalCost` | string \| null | Availability (minimum) charge including PIS/COFINS and ICMS, BRL as an exact decimal string; null when the bill has none. |
| `[].availabilityPisCof` | string \| null | PIS/COFINS on the availability charge, BRL as an exact decimal string; null when the bill has none. |
| `[].availabilityIcms` | string \| null | ICMS on the availability charge, BRL as an exact decimal string; null when the bill has none. |
| `[].consumptionKwh` | string \| null | Total metered consumption, kWh as a decimal string; null when the bill has none. |
| `[].notCompensatedConsumptionKwh` | string \| null | Consumption not compensated by distributed generation, kWh as a decimal string; null when the bill has none. |
| `[].icmsAndPisCofExemptCompensationKwh` | string \| null | Energy exempt from ICMS and PIS/COFINS ("En. isenta"), kWh as a decimal string; null when the bill has none. |
| `[].icmsAndPisCofExemptCompensationBrl` | string \| null | ICMS and PIS/COFINS exempt energy ("En. isenta"), BRL as an exact decimal string; null when the bill has none. |
| `[].icmsExemptCompensationKwh` | string \| null | Energy compensated by distributed generation other than own generation ("En. SCEE"), kWh as a decimal string; null when the bill has none. |
| `[].icmsExemptCompensationBrl` | string \| null | SCEE compensated energy ("En. SCEE"), BRL as an exact decimal string; null when the bill has none. |
| `[].icmsExemptCompensationPisCof` | string \| null | PIS/COFINS on the SCEE energy, BRL as an exact decimal string; null when the bill has none. |
| `[].gdICompensationKwh` | string \| null | GD I compensation, kWh as a decimal string; null when the bill has none. |
| `[].gdICompensationBrl` | string \| null | GD I compensation (zero or negative, a credit), BRL as an exact decimal string; null when the bill has none. |
| `[].gdIICompensationKwh` | string \| null | GD II compensation, kWh as a decimal string; null when the bill has none. |
| `[].gdIICompensationBrl` | string \| null | GD II compensation (zero or negative, a credit), BRL as an exact decimal string; null when the bill has none. |
| `[].additionalCompensationKwh` | string \| null | Additional compensation, kWh as a decimal string; null when the bill has none. |
| `[].additionalCompensationBrl` | string \| null | Additional compensation (zero or negative, a credit), BRL as an exact decimal string; null when the bill has none. |
| `[].currentBalanceKwh` | string \| null | Credit balance after this bill, kWh as a decimal string; null when the bill has none. |

### `search_gd_reports`

**Buscar relatórios GD**

Effect: Read-only

Search imported CEMIG GD reports (Relatório GD) in the authenticated user's company: per-period generation, injected, compensated and balance kWh. Each row is one installation for one period, use period and tariff time band (FORAP, PONTA, NOTUR) in one generation unit's report; the generator's own row usually has a gdMode ending in '-Geradora', the installations it credits '-Recebedora'. Filter by generation unit uuid, installation uuid, consumer unit uuid (its installation's rows), installation number, period (MMM/YYYY) or time band. Values are kWh decimal strings exactly as imported. Returns an array (possibly empty), newest period first; another company's uuids match nothing.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `generationUnitUuid` | string (uuid) | no | Only rows of this generation unit's report: its own generation and every installation it credits. |
| `installationUuid` | string (uuid) | no | Only rows describing this installation. |
| `consumerUnitUuid` | string (uuid) | no | Only rows describing this consumer unit's installation. |
| `installationNumber` | string | no | Exact CEMIG installation number in the report, 10 digits starting with 3. |
| `period` | string | no | Billing period MMM/YYYY with an uppercase pt-BR month abbreviation (JAN FEV MAR ABR MAI JUN JUL AGO SET OUT NOV DEZ). Example: FEV/2026. |
| `postHour` | `"FORAP"`, `"PONTA"`, `"NOTUR"` | no | Only this tariff time band: FORAP, PONTA or NOTUR. |
| `limit` | integer | no | Maximum rows to return, 1-100 (default 50). |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].generationUnitUuid` | string (uuid) | Generation unit whose GD report the row was imported from. |
| `[].installationUuid` | string (uuid) | Installation the row describes. |
| `[].installationNumber` | string |  |
| `[].consumerUnitNumber` | string \| null |  |
| `[].period` | string | Report period, MMM/YYYY. |
| `[].usePeriod` | string | Period whose credits the row compensates, MMM/YYYY. |
| `[].gdMode` | `"Auto Consumo-Recebedora"`, `"Auto Consumo-Geradora"`, `"Ger. Compartilhada-Geradora"`, `"Ger. Compartilhada-Recebedora"`, `""` | CEMIG GD mode: the generator ("-Geradora") or a recipient ("-Recebedora"). |
| `[].postHour` | `"FORAP"`, `"PONTA"`, `"NOTUR"` | Tariff time band (posto horário): FORAP, PONTA or NOTUR. |
| `[].quota` | string | Share of the generation unit's credits assigned to the installation, percent (0-100), 5 decimal places. |
| `[].consumptionKwh` | string | Energy consumed by the installation, kWh as a decimal string (8 decimal places). |
| `[].generationKwh` | string | Energy generated by the installation, kWh as a decimal string (8 decimal places). |
| `[].compensationKwh` | string | Energy compensated in the period, kWh as a decimal string (8 decimal places). |
| `[].additionalCompensationKwh` | string \| null | Additional compensated energy (GD II), kWh as a decimal string; null when absent. |
| `[].previousBalanceKwh` | string | Credit balance carried over from the last period, kWh as a decimal string (8 decimal places). |
| `[].transferredKwh` | string | Energy transferred to other installations, kWh as a decimal string (8 decimal places). |
| `[].injectedKwh` | string | Energy injected or received from other installations, kWh as a decimal string (8 decimal places). |
| `[].currentBalanceKwh` | string | Credit balance at the end of the period, kWh as a decimal string (8 decimal places). |
| `[].expiredBalanceKwh` | string | Credit balance that expired in the period, kWh as a decimal string (8 decimal places). |
| `[].nextBalanceExpKwh` | string | Credit balance that expires next, kWh as a decimal string (8 decimal places). |
| `[].nextBalanceExpPeriod` | string \| null | Period when nextBalanceExpKwh expires, MMM/YYYY. |

### `search_generation_units`

**Buscar unidades geradoras**

Effect: Read-only

Search generation units (Unidades Geradoras, the plants whose output is credited to consumer units) in the authenticated user's company, ordered by name. Filter by name (substring, ignoring case and accents), the plant's installation number or GD regime (GD_I, GD_II). Use the returned uuid with search_gd_reports or search_energy_bills. Returns an array (possibly empty).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `name` | string | no | Substring match on the generation unit name, ignoring case and accents. |
| `installationNumber` | string | no | Exact CEMIG installation number of the plant, 10 digits starting with 3. |
| `gdType` | `"GD_I"`, `"GD_II"` | no | Only this GD regime: GD_I or GD_II. |
| `limit` | integer | no | Maximum rows to return, 1-100 (default 50). |

Returns `structuredContent.result` (array of object). Its fields:

| Field | Type | Description |
|---|---|---|
| `[].uuid` | string (uuid) |  |
| `[].name` | string |  |
| `[].gdType` | `"GD_I"`, `"GD_II"` | Regulatory GD regime: GD_I or GD_II. |
| `[].administrationFeePercentage` | string | Administration fee, percent as a decimal string with 2 places ("12.00"). |
| `[].installationUuid` | string (uuid) | The plant's CEMIG installation. |
| `[].installationNumber` | string \| null |  |
| `[].consumerUnitNumber` | string \| null |  |

### `get_generation_unit`

**Detalhar unidade geradora**

Effect: Read-only

Get one generation unit by uuid with its GD regime, administration fee and plant installation. A missing or another company's uuid returns NotFoundError.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Generation unit uuid, e.g. from search_generation_units or an energy bill or GD report row. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `uuid` | string (uuid) |  |
| `name` | string |  |
| `gdType` | `"GD_I"`, `"GD_II"` | Regulatory GD regime: GD_I or GD_II. |
| `administrationFeePercentage` | string | Administration fee, percent as a decimal string with 2 places ("12.00"). |
| `installationUuid` | string (uuid) | The plant's CEMIG installation. |
| `installationNumber` | string \| null |  |
| `consumerUnitNumber` | string \| null |  |

### `describe_analytics_schema`

**Descrever dados para análise**

Effect: Read-only

Describe the read-only analytics views that run_analytics_query can use: what each view holds, its columns with types, units and allowed values, the aggregation rules that keep totals right, helper functions, limits and verified example queries. Call it once before writing SQL: without arguments for the overview, then with views=[…] for the columns of the views you need. Only administrators can use it (ForbiddenError otherwise).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `views` | array of `"company"`, `"clients"`, `"contacts"`, `"consumer_units_contacts"`, `"installations"`, `"consumer_units"`, `"generation_units"`, `"compensation_distributions"`, `"compensation_distribution_entries"`, `"invoices"`, `"extra_invoice_items"`, `"invoice_compensation_ratios"`, `"billings"`, `"energy_bills"`, `"gd_reports"`, `"notification_messages"` | no | Views to describe column by column. Omit for the overview of every view. |

Returns `structuredContent.result` (object). Its fields:

| Field | Type | Description |
|---|---|---|
| `dialect` | string |  |
| `timeZone` | string |  |
| `limits` | object |  |
| `limits.defaultMaxRows` | number |  |
| `limits.maxRows` | number |  |
| `limits.statementTimeoutSeconds` | number |  |
| `limits.maxSqlLength` | number |  |
| `limits.maxRowBytes` | number |  |
| `limits.maxResultChars` | number |  |
| `limits.maxStalenessSeconds` | number |  |
| `notes` | array of string |  |
| `views` | array of object |  |
| `views[].name` | string |  |
| `views[].description` | string |  |
| `views[].columnNames` | array of string |  |
| `views[].columns` | array of object | Optional. |
| `views[].columns[].name` | string |  |
| `views[].columns[].type` | string |  |
| `views[].columns[].description` | string |  |
| `functions` | array of object |  |
| `functions[].signature` | string |  |
| `functions[].description` | string |  |
| `examples` | array of object |  |
| `examples[].question` | string |  |
| `examples[].sql` | string |  |

### `run_analytics_query`

**Executar consulta analítica**

Effect: Read-only

Run one read-only PostgreSQL 16 query (SELECT, or WITH … SELECT) over the Helios analytics views and return its rows. Use it for analysis across many records: totals, counts, averages, rankings, trends by period, comparisons and shares. For one specific client, UC or invoice, use the search_ and get_ tools. Call describe_analytics_schema first: only the views and columns it lists exist, and it gives the aggregation rules and verified examples. Every view holds only one company's rows, so never filter by company. Allowed: joins, CTEs, subqueries, GROUP BY/ROLLUP, window functions, FILTER, CASE, and aggregate, math, text, date/time, array and JSON functions. Not allowed: more than one statement, writes, DDL, SET, locking clauses, system functions and catalogs. Qualify a column only with the name or alias of a FROM item of the query or of an enclosing query. To use a set-returning function's columns with a prefix, name them: generate_series(…) AS g(day), then g.day; any other expression in FROM needs an alias, as in CAST('2026-01-01' AS date) AS d(day). Aggregate in SQL and return few rows: results stop at maxRows (default 200, max 1000) and at a size limit, and truncated=true means the rows are incomplete, so never total them yourself. Money is BRL as exact decimal strings: quote stored totals instead of recomputing them. Every result has dataAsOf and freshness: say how current the data is. Only administrators can use it (ForbiddenError otherwise). companyUuid: only super administrators may pass another company's uuid (from search_companies); for anyone else a company other than their own is NotFoundError. Calls are rate limited per user; run queries one at a time. Errors: ValidationError when the query is rejected or fails (message, hint and position included: fix it and retry), ServerError when analytics is unavailable or not enabled, its data is too far behind, or too many calls were made (retry later).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `sql` | string | yes | One read-only PostgreSQL SELECT (WITH … SELECT allowed) over the views describe_analytics_schema lists. No other statements. |
| `companyUuid` | string (uuid) | no | Company to query. Omit for your own company. Only super administrators may pass another company's uuid, taken from search_companies. |
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
| `rows` | array of array of any |  |
| `rowCount` | integer |  |
| `maxRows` | integer |  |
| `truncated` | boolean | True when more rows existed than were returned: the rows are incomplete. |
| `truncatedBy` | `"rows"`, `"size"` \| null |  |
| `dataAsOf` | string | The data includes every change committed up to this moment (America/Sao_Paulo, ISO 8601). |
| `freshness` | `"current"`, `"behind"`, `"primary"` |  |
| `stalenessSeconds` | number \| null |  |
| `durationMs` | integer |  |
