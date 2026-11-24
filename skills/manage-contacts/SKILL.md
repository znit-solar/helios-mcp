---
name: manage-contacts
description: Add, change, link, unlink or remove the contacts of Helios consumer units (UCs), including which contact is primary, after finding the contact and every UC it is linked to. Use when the user wants to update a phone or e-mail, add someone who should receive invoices, change the primary contact, take someone off a UC, or remove a contact ("alterar contato", "trocar telefone", "contato principal", "tirar o contato da UC", "remover contato").
---

# Manage contacts

Contacts belong to the company and are linked to consumer units (UCs). The
linked contacts of a UC receive its invoices and notifications, through
the UC's notification channel, and each UC has at most one primary
contact.

Records are looked up with `describe_analytics_schema`, whose overview has
verified lookup examples, and `run_analytics_query`, which runs them (the
`analyze-data` workflow). The Helios tools document their own parameters,
formats and errors. Read their descriptions; this skill only gives the
order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Never create, update or delete anything before the user confirms. Show
  the change and wait for a clear "sim".
- Never guess or invent a uuid. Take uuids only from tool results in this
  conversation.
- If you have not called `describe_analytics_schema` in this conversation,
  call it first.
- Run lookups one at a time, never in parallel.
- Show values formatted as the server instructions say (phone, e-mail, UC
  number).

## Steps

1. **Resolve the contact and its UCs** (read-only):
   - by phone or e-mail: the contact lookup example, with both in one
     query. It returns the contact with every UC it is linked to;
   - by client (CPF/CNPJ or name): the client lookup example, then the one
     for a client's UCs with their contacts;
   - by installation or UC number: the lookup example for what a number
     belongs to, then the client's UCs with their contacts.

   If several contacts match, list them (name, phone, e-mail, UCs) and ask
   which one; never pick one yourself.
2. **Decide what changes, and with which tool:**
   - a new person: check by phone and e-mail (step 1) that they don't
     exist, then `create_contact`, linking the UCs in the same call;
   - an existing person's new phone, e-mail or name: `update_contact`. The
     change reaches every UC the contact is linked to, so name them all;
   - an existing person who should also receive another UC's invoices:
     link them with `update_contact`, not a new contact;
   - stop sending one UC's invoices to someone ("tirar da UC", "parar de
     receber"): unlink them from that UC with `update_contact`, never
     `delete_contact`;
   - remove someone from every UC and from Helios: `delete_contact` (see
     Deleting);
   - "só por WhatsApp" or "só por e-mail" is the UC's notification channel,
     a UC setting changed with `update_consumer_unit`, not a contact
     change.

   If the user's words fit two of these, such as unlinking or deleting,
   ask which one before going on.
3. **Changing the primary contact:** a UC already having a primary contact
   makes the new one fail with `ConflictError`. Unset the current primary
   first, then set the new one, and tell the user both changes.
4. **Confirm the change.** Tell the user the record, each field from old to
   new, and the scope (every UC the change reaches), then wait for a clear
   "sim". For example:

   > Vou alterar o **contato** Maria Exemplo: telefone de +55 (31) 99876-5432 para +55 (31) 98888-7777. Ela recebe as faturas das UCs 3.012.345.678-90 e 3.098.765.432-10, e a mudança vale para as duas. Confirma?
5. **Run and report** the result in pt-BR. To check it, look the contact up
   again: right after a write, only a result whose `freshness` is
   `current` shows it.

## Deleting

`delete_contact` is irreversible: it also deletes the contact's links and
its whole WhatsApp and e-mail notification history. Prefer unlinking when
the person should only stop receiving one UC's invoices. Before deleting,
tell the user exactly this consequence and get an explicit confirmation
for that contact. A second delete of the same contact returns
`NotFoundError`; that means it is already gone.

## When a call fails

- `ValidationError`: show the message and ask for a corrected value.
- `ConflictError`: the e-mail or phone belongs to another contact, or the
  UC already has a primary contact. Look the existing record up, show it
  and ask how to proceed.
- `NotFoundError`: look the record up again; never retry with a guessed
  uuid.
- `ServerError` or a timeout from a write: look the contact up (only a
  `current` result counts) to see whether the change happened before
  retrying.
- `ServerError` from `run_analytics_query`: lookups are temporarily
  unavailable. Don't change anything you couldn't look up; tell the user to
  try again in a few minutes or to use the Helios web app.
