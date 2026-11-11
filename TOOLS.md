# Helios MCP tools

<!-- Generated from the Helios MCP server by `npm run generate:client-kit`. Do not edit by hand. -->

Server version: `1.0.0`

This catalog lists what the Helios MCP server offers to AI assistants.
Assistants read the same descriptions and parameter schemas directly from
the server when they connect, so you never need to paste them anywhere.
Every tool acts only on the data of the company you logged in with.

## Tools

| Tool | Title | Effect |
|---|---|---|
| [`get_current_user`](#get_current_user) | Usuário conectado | Read-only |
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
| [`search_energy_bills`](#search_energy_bills) | Buscar contas de energia | Read-only |
| [`search_gd_reports`](#search_gd_reports) | Buscar relatórios GD | Read-only |
| [`search_generation_units`](#search_generation_units) | Buscar unidades geradoras | Read-only |
| [`get_generation_unit`](#get_generation_unit) | Detalhar unidade geradora | Read-only |

## Server instructions

Every assistant receives these instructions when it connects:

- Helios is the management system of a Brazilian solar energy company: clients, CEMIG installations, consumer units (UCs) and their contacts.
- Every tool operates within the authenticated user's company. A uuid from another company behaves exactly like a missing one (NotFoundError).
- Call get_current_user first to confirm the connection and which company you are acting for.
- Search before creating: look clients up by document and installations by installation or consumer unit number. Create tools return ConflictError on duplicates.
- Formats: documents (CPF 11 digits, CNPJ 14 digits) and ZIP codes are digits only; installation numbers are 10 digits starting with 3; state codes are 2 uppercase letters.
- Read-only billing data: invoices (Cobranças) and their boletos. Periods are MMM/YYYY with pt-BR months (FEV/2026); money is BRL as exact decimal strings, to quote as stored, never to recompute.
- Read-only CEMIG data: generation units (UGs), imported energy bills (contas de energia) and GD reports (generation, compensation and credit balance in kWh) per installation and period.
- A failed call returns isError with the text `<ErrorType>: <message>`, where ErrorType is ValidationError, NotFoundError, ConflictError or ServerError. Messages are in Brazilian Portuguese and can be shown to the user as they are.
- Helios users are Brazilian: answer them in Brazilian Portuguese unless they write in another language.

## Tool reference

### `get_current_user`

**Usuário conectado**

Effect: Read-only

Return the authenticated Helios user and the company every other tool is scoped to. Call it first in a session to confirm the connection works and which company you are acting for.

No parameters.

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

### `create_installation`

**Cadastrar instalação**

Effect: Write (additive)

Create an installation in the authenticated user's company. Provide installationNumber (10 digits, starts with 3), consumerUnitNumber (1-15 digits), or both — at least one is required. Returns the DTO.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `installationNumber` | string | no | CEMIG installation number, exactly 10 digits, must start with '3'. Example: 3000945591. |
| `consumerUnitNumber` | string | no | CEMIG consumer-unit number, digits only, up to 15 digits. Often shown lpad'd to 15 in UIs but stored without padding. |

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

### `delete_contact`

**Excluir contato**

Effect: Write (may change or delete existing data)

Delete a contact in the authenticated user's company. The uuid must belong to your company; cross-company or unknown uuids return NotFoundError. Cascades to consumer-unit links and all notification history (WhatsApp + Email) via FK ON DELETE CASCADE. Irreversible and non-idempotent: a successful call returns `{ uuid }`; a second call with the same uuid returns NotFoundError because the row no longer exists.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Existing contact uuid to delete. |

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

### `get_invoice`

**Detalhar cobrança**

Effect: Read-only

Get one invoice by uuid with its stored line amounts (compensation, discounts, CEMIG bill, surplus B wire, extra items, total), the generation units that compensated it and every boleto issued for it, newest first. Amounts are BRL decimal strings exactly as stored; don't recompute them. Read-only: it never issues or syncs boletos. A missing or another company's uuid returns NotFoundError.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Invoice uuid, e.g. from search_invoices. |

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

### `get_generation_unit`

**Detalhar unidade geradora**

Effect: Read-only

Get one generation unit by uuid with its GD regime, administration fee and plant installation. A missing or another company's uuid returns NotFoundError.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `uuid` | string (uuid) | yes | Generation unit uuid, e.g. from search_generation_units or an energy bill or GD report row. |
