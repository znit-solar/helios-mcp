---
name: onboard-client
description: Register a new Helios client end to end, from the client record to its CEMIG installation, consumer unit (UC) and contacts. Use when the user wants to add, register or onboard a client, a new UC for a client, or a new energy account ("cadastrar cliente", "cadastrar UC", "nova unidade consumidora").
---

# Onboard a client

Helios models a client like this: a **client** (CPF or CNPJ) owns
**consumer units** (UCs); each UC is attached to exactly one CEMIG
**installation**; **contacts** are linked to UCs and receive invoices and
notifications. Onboarding creates whichever of these don't exist yet.

The Helios tools document their own parameters, formats and errors. Read
their descriptions; this skill only gives the order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Never create, update or delete anything before the user confirms. Show what
  you are about to write (a short summary in pt-BR, with formatted CPF/CNPJ
  and CEP) and wait for a clear "sim".
- Search before creating. Duplicates are refused with `ConflictError`, and an
  existing record is reused rather than recreated.
- Never guess or invent a uuid. Take uuids only from tool results in this
  conversation.
- If you have not called `get_current_user` in this conversation, call it
  first and tell the user which company you are acting for.

## Steps

1. **Collect the data.** Ask only for what is missing:
   - client: name and CPF/CNPJ, address;
   - installation: the CEMIG installation number and/or the UC number;
   - UC billing settings: holder name and document, due day, discount,
     notification channel and the other required fields of
     `create_consumer_unit`;
   - contacts: name, phone, optional email, and which one is the primary
     contact.
   A CEMIG energy bill usually has the client, installation and UC data;
   offer to read it if the user has one.
2. **Look for what already exists** (read-only, no confirmation needed):
   - `search_clients` by document;
   - `search_installations` by installation number or UC number;
   - `search_consumer_units` by installation number or UC number. If the
     installation already has a UC, stop: tell the user who it belongs to
     and ask how to proceed. An installation holds only one UC.
   - `search_contacts` by phone and by email for each contact.
3. **Confirm the plan.** Summarize what will be reused and what will be
   created, then ask for confirmation.
4. **Create the client, installation and UC.** Prefer a single
   `create_consumer_unit` call: it can reference the client and installation
   you found and create the missing ones in the same transaction, so a
   failure leaves nothing half-registered. Use `create_client` and
   `create_installation` on their own only when the user wants just that
   record.
5. **Add the contacts.** For a new contact, `create_contact` linked to the
   new UC. For a contact that already exists, link it with `update_contact`
   instead of creating a duplicate. Only one primary contact per UC.
6. **Report.** List what was created and what was reused, in pt-BR.

## When a call fails

The error text starts with its type; the message after it is in pt-BR and
can be shown to the user.

- `ValidationError`: a value has the wrong format. Show the message, ask
  the user for the corrected value, then retry.
- `ConflictError`: the record already exists (document, installation, UC,
  email or phone, or a second primary contact). Search for it, show it to
  the user and offer to reuse it.
- `NotFoundError`: a uuid is unknown in this company. Search again by
  document or number; never retry with a guessed uuid.
- `ServerError` or a timeout: don't retry a create blindly, since the
  first call may have succeeded. Search first; retry only if the record is
  missing, and tell the user if the error persists.
