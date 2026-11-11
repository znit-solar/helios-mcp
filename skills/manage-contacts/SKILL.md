---
name: manage-contacts
description: Add, change, link, unlink or remove the contacts of Helios consumer units (UCs), including which contact is primary. Use when the user wants to update a phone or email, add someone who should receive invoices, change the primary contact, or remove a contact ("alterar contato", "trocar telefone", "contato principal").
---

# Manage contacts

Contacts belong to the company and are linked to consumer units (UCs). The
linked contacts of a UC receive its invoices and notifications, and each UC
has at most one primary contact.

The Helios tools document their own parameters, formats and errors. Read
their descriptions; this skill only gives the order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Never create, update or delete anything before the user confirms. Show the
  change (who, which UCs, old and new values) and wait for a clear "sim".
- Never guess or invent a uuid. Take uuids only from tool results in this
  conversation.

## Steps

1. **Find the contact and the UCs.** `search_contacts` by phone or email
   (most reliable) or by name; `search_consumer_units` for the UCs involved.
   To see who is linked to a UC, `search_contacts` by the UC's uuid.
2. **Pick the tool:**
   - new person: check by phone and email that they don't exist, then
     `create_contact`, linking the UCs in the same call;
   - existing person, new data or new links: `update_contact`. Linking an
     existing contact to another UC is an update, not a new contact;
   - stop sending to someone for one UC: unlink with `update_contact`;
   - remove someone entirely: `delete_contact`.
3. **Changing the primary contact:** a UC already having a primary contact
   makes the new one fail with `ConflictError`. Unset the current primary
   first, then set the new one, and tell the user both changes.
4. **Confirm, run, report** the result in pt-BR.

## Deleting

`delete_contact` is irreversible: it also deletes the contact's links and
its whole WhatsApp and email notification history. Prefer unlinking when
the person should only stop receiving one UC's invoices. Before deleting,
tell the user exactly this consequence and get an explicit confirmation for
that contact. A second delete of the same contact returns `NotFoundError`;
that means it is already gone.

## When a call fails

- `ValidationError`: show the message and ask for a corrected value.
- `ConflictError`: the email or phone belongs to another contact, or the UC
  already has a primary contact. Show the existing record and ask how to
  proceed.
- `NotFoundError`: search again; never retry with a guessed uuid.
- `ServerError` or a timeout: search to see whether the change happened
  before retrying.
