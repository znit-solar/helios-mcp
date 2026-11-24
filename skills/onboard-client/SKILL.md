---
name: onboard-client
description: Register a new Helios client end to end, from the client record to its CEMIG installation, consumer unit (UC) and contacts, after checking that the installation or UC number is free and whether the client already exists. Use when the user wants to add, register or onboard a client, a new UC for a client, or a new energy account ("cadastrar cliente", "cadastrar UC", "cadastrar instalação", "nova unidade consumidora", "nova conta").
---

# Onboard a client

Helios models a client like this: a **client** (CPF or CNPJ) owns
**consumer units** (UCs); each UC is attached to exactly one CEMIG
**installation**; **contacts** are linked to UCs and receive invoices and
notifications. Onboarding creates whichever of these don't exist yet.

Records are looked up with `describe_analytics_schema`, whose overview has
verified lookup examples, and `run_analytics_query`, which runs them (the
`analyze-data` workflow). The Helios tools document their own parameters,
formats and errors. Read their descriptions; this skill only gives the
order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Never create, update or delete anything before the user confirms. Show
  what you are about to write and wait for a clear "sim".
- Resolve before creating: duplicates are refused with `ConflictError`, and
  an existing record is reused rather than recreated.
- Never guess or invent a uuid. Take uuids only from tool results in this
  conversation.
- If you have not called `describe_analytics_schema` in this conversation,
  call it first and tell the user which company you are acting for.
- Run lookups one at a time, never in parallel.
- Show values formatted as the server instructions say (CPF, CNPJ, UC
  number, phone, CEP).

## Steps

1. **Collect the data.** Ask only for what is missing:
   - client: name and CPF/CNPJ, address;
   - installation: the CEMIG installation number and/or the UC number;
   - UC billing settings: holder name and document, due day, discount,
     notification channel and the other required fields of
     `create_consumer_unit`;
   - contacts: name, phone, optional e-mail, and which one is the primary
     contact.

   A CEMIG energy bill usually has the client, installation and UC data;
   offer to read it if the user has one. Users who say "cadastrar
   instalação" or "nova conta" almost always mean a new UC.
2. **Resolve what already exists** (read-only, no confirmation needed):
   - **The number, first.** Look the installation or UC number up with the
     lookup example for what a number belongs to:
     - a UC already uses it: stop. Tell the user whose UC it is (client and
       UC number) and ask how to proceed: an installation holds only one
       UC;
     - it is a plant's (a generation unit, "usina"): stop. Tell the user
       the number belongs to a plant, which a UC can't use, and ask them to
       check the number;
     - an installation without a UC: reuse it;
     - nothing: the installation will be created.
   - **The client.** Look the CPF/CNPJ up with the client lookup example.
     If the client exists, reuse it, and check that its name matches what
     the user gave.
   - **The contacts.** For each contact, look the phone and the e-mail up
     in one query with the contact lookup example. An existing contact is
     linked to the new UC, not created again.
3. **Confirm the plan.** In one message, name each record, say what will
   be reused and what will be created, with its values, and wait for a
   clear "sim". For example:

   > Vou cadastrar a **UC** 3.012.345.678-90 (instalação 3000945591) da cliente **Maria Exemplo** (CPF 123.456.789-09, já cadastrada), com vencimento todo dia 10, e vincular a ela o **contato** já cadastrado de telefone +55 (31) 99876-5432, como principal. Confirma?
4. **Create the client, installation and UC.** Prefer a single
   `create_consumer_unit` call: it can reference the client and
   installation you found and create the missing ones in the same
   transaction, so a failure leaves nothing half-registered. Use
   `create_client` and `create_installation` on their own only when the
   user wants just that record.
5. **Add the contacts.** For a new contact, `create_contact` linked to the
   new UC. For a contact that already exists, link it with
   `update_contact` instead of creating a duplicate. Only one primary
   contact per UC.
6. **Report.** List what was created and what was reused, in pt-BR. To
   check the result, look the records up again: right after a write, only
   a result whose `freshness` is `current` shows it.

## When a call fails

The error text starts with its type; the message after it is in pt-BR and
can be shown to the user.

- `ValidationError`: a value has the wrong format. Show the message, ask
  the user for the corrected value, then retry.
- `ConflictError`: the record already exists (document, installation, UC,
  e-mail or phone, or a second primary contact), or the installation is a
  plant's. For a duplicate, look the record up, show it to the user and
  offer to reuse it. For a plant's installation, tell the user the number
  belongs to a plant and ask them to check it; don't retry.
- `NotFoundError`: a uuid is unknown in this company. Look the record up
  again by document or number; never retry with a guessed uuid.
- `ServerError` or a timeout from a create: don't retry blindly, since the
  first call may have succeeded. Look the record up first (only a
  `current` result counts); retry only if it is missing, and tell the user
  if the error persists.
- `ServerError` from `run_analytics_query`: lookups are temporarily
  unavailable. Don't create anything you couldn't check first; tell the
  user to try again in a few minutes or to use the Helios web app.
