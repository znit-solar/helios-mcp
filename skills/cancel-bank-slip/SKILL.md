---
name: cancel-bank-slip
description: Cancel a Helios boleto (bank slip) at the bank, after finding its invoice (cobrança), showing the boleto to the user and getting their confirmation. Use when the user wants to cancel a boleto or a charge, or write a boleto off at the bank ("cancelar boleto", "cancelar cobrança", "baixar boleto", "dar baixa no boleto"). Does not issue or reissue boletos.
---

# Cancel a boleto

An invoice (cobrança) is Helios's monthly charge to a client. Each invoice
can have several boletos over time, issued through the company's bank.
Canceling asks that bank to write off (dar baixa) one boleto, so it can no
longer be paid. The invoice itself stays in Helios with its amounts
unchanged.

The Helios tools document their own parameters, formats and errors. Read
their descriptions; this skill only gives the order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Canceling is irreversible. Never call `cancel_bank_slip` before the user
  confirms that specific boleto with a clear "sim".
- Never guess or invent a uuid. Take the invoice uuid and the boleto's
  billing uuid from the `get_invoice` result in this conversation; the
  boleto must be one of that invoice's boletos.
- If you have not called `get_current_user` in this conversation, call it
  first and tell the user which company you are acting for: the cancel
  goes to that company's bank.
- "Cancelar a cobrança" usually means its boleto: this workflow cancels
  only the boleto. "Baixar o boleto" can also mean downloading it; if it
  is unclear, ask whether the user wants to cancel it at the bank.
- No tool issues or reissues a boleto. If the client still has to pay,
  tell the user to issue a new boleto in the Helios web app.

## Steps

1. **Find the invoice.** Start from the most precise identifier the user
   gave: `search_clients` by CPF/CNPJ or name, `search_consumer_units` by
   installation or UC number, `search_contacts` by phone or email. Then
   `search_invoices` by the client's or the UC's uuid, with the period
   (`MMM/YYYY`) if the user named one. If several invoices match, list them
   (period, UC, total, boleto status) and ask which one.
2. **Pick the boleto.** Call `get_invoice` and look at its boletos, newest
   first. Each boleto's `cancel.allowed` says whether `cancel_bank_slip`
   can cancel it now: offer only boletos where it is true. Show each
   boleto's status to the user as:

   | Status | Show as |
   |---|---|
   | `created` | Em aberto |
   | `updated` | Atualizado |
   | `updating` | Atualizando |
   | `expired` | Vencido |
   | `processing` | Registrando |
   | `paid` | Pago |
   | `processing_payment` | Compensando |
   | `returned` | Estornado |
   | `canceling` | Cancelando |
   | `canceled` | Cancelado |

   If no boleto has `cancel.allowed: true`, tell the user why for each
   boleto (or that the invoice has no boleto) and stop. If more than one
   has, list them and ask which. Whenever you relay a `cancel.reason`,
   present it per boleto, for example
   `Boleto com vencimento <dueDate>: <reason>`, so the user doesn't read
   it as being about the whole invoice. A boleto's state can change
   between calls, so `cancel_bank_slip` has the final say: if its answer
   disagrees with `get_invoice`, follow `cancel_bank_slip`.
3. **Show the boleto:** client, UC, invoice period, bank, amount, due
   date and status. Quote the amount and dates as returned, without
   recomputing them.
4. **Confirm.** Tell the user that:
   - the cancel can't be undone, and the boleto can no longer be paid;
   - a boleto issued later for this invoice gets a new barcode, so the
     client must not use the old one;
   - a paid boleto, or one whose payment is clearing, can't be canceled.

   Then wait for a clear "sim" for this boleto.
5. **Cancel** with `cancel_bank_slip`, passing the invoice uuid and the
   boleto's billing uuid.
6. **Report** the result's `outcome` in pt-BR:
   - `canceled`: the bank canceled the boleto;
   - `already_canceled`: the boleto was already canceled, by an earlier
     attempt or by someone else. Nothing else changed.

   Then call `get_invoice` again and check the invoice's other boletos:
   - one with `cancel.allowed: true` can still be paid: tell the user.
     Cancel it too only if the user asks, going through steps 3 to 6 for
     it;
   - one with status `processing` is still being registered at the bank
     and may become payable, and one with status `canceling` can be paid
     until the bank confirms the cancel: mention it and suggest checking
     it again later with `get_invoice`.

To cancel several boletos, list them all and get one confirmation for the
list. Then cancel them one at a time and report each `outcome` as in
step 6. In a confirmed batch, skip step 6's `get_invoice` check after
each cancel, and run it once per invoice involved, after the last cancel.
Stop at the first `ServerError` and check it as below before going on.

## When a call fails

The error text starts with its type; the message after it is in pt-BR and
can be shown to the user as it is. Recognize the cases below by their
meaning: the exact wording may change.

- `ValidationError`: an argument has the wrong format. Take both uuids
  from `get_invoice` and retry.
- `NotFoundError`: the invoice or the boleto doesn't exist in this
  company, or the boleto isn't one of that invoice's. Call `get_invoice`
  again and take both uuids from it; never retry with a guessed uuid.
- `ConflictError`: nothing was canceled, and repeating the call now would
  fail the same way. Show the message and act on what it says:
  - **the boleto's status forbids it** (paid, payment clearing, returned,
    or a cancel already in progress): if a cancel is in progress, check
    later with `get_invoice` whether the boleto reached `canceled`;
  - **still being registered at the bank** ("ainda está sendo registrado
    no banco"): wait a few minutes, call `get_invoice` again, and retry
    only if the user still wants it and the boleto's `cancel.allowed` is
    now true;
  - **not found at the bank** ("não foi encontrado no banco"): tell the
    user to check this boleto with the bank;
  - **in settlement or already settled by the bank** ("em liquidação ou
    já liquidado"): the client has probably paid it, even if Helios
    doesn't show the payment yet; tell the user;
  - **refused by the bank** ("O banco recusou o cancelamento"): show the
    bank's answer as it is; the user should check with the bank.
- `ServerError` or a timeout: the outcome may be unknown, and the bank may
  have canceled the boleto. Call `get_invoice` before anything else:
  - `canceled`: the cancel worked; tell the user, and do step 6's check
    of the invoice's other boletos;
  - `canceling`: still in progress, so check again later;
  - `paid`, `processing_payment` or `returned`: the client paid while the
    cancel was running. Don't retry; tell the user to check the boleto
    with the bank;
  - still open (`cancel.allowed: true`): Helios recorded no cancel. If
    the message is about the company's bank settings ("configuração
    bancária"), the request never left Helios and those settings need
    fixing: tell the user, and retry only when they say the settings were
    fixed. Any other message (the bank was unavailable, or the
    communication failed, possibly after the request was sent) means the
    bank may have acted: tell the user, and retry `cancel_bank_slip` only
    with their OK. The retry is safe: it cancels the boleto, or answers
    `already_canceled` or one of the refusals above if the bank had
    already canceled it.
