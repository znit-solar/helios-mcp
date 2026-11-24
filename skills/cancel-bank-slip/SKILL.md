---
name: cancel-bank-slip
description: Cancel a Helios boleto (bank slip) at the bank, after making sure a cancel is what the user wants, finding the invoice (cobrança) and its boletos, and confirming the exact boleto. Use when the user wants to cancel a boleto or a charge, or write a boleto off at the bank ("cancelar boleto", "cancelar cobrança", "baixar boleto", "dar baixa no boleto"). Does not issue or reissue boletos, record payments or delete invoices.
---

# Cancel a boleto

An invoice (cobrança) is Helios's monthly charge to a client. Each invoice
can have several boletos over time, issued through the company's bank.
Canceling asks that bank to write off (dar baixa) one boleto, so it can no
longer be paid. The invoice itself stays in Helios with its amounts
unchanged.

Records are looked up with `describe_analytics_schema`, whose overview has
verified lookup examples, and `run_analytics_query`, which runs them (the
`analyze-data` workflow). The Helios tools document their own parameters,
formats and errors. Read their descriptions; this skill only gives the
order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Canceling is irreversible. Never call `cancel_bank_slip` before the user
  confirms that specific boleto with a clear "sim".
- Never guess or invent a uuid. Take the invoice uuid and the boleto's
  billing uuid from the lookup of the invoice with its boletos, in this
  conversation; the boleto must be one of that invoice's boletos.
- If you have not called `describe_analytics_schema` in this conversation,
  call it first and tell the user which company you are acting for: the
  cancel goes to that company's bank.
- Before acting on a boleto's status, only a lookup result whose
  `freshness` is `current` counts: if it is `behind`, wait a moment and
  query again.
- No tool issues or reissues a boleto, records a payment, downloads a
  boleto or deletes an invoice: point the user to the Helios web app for
  those.
- Show values formatted as the server instructions say (money, dates, UC
  number, CPF/CNPJ), never the uuids.

## Steps

1. **Resolve the invoice and its boletos.** Find the UC first: by
   installation or UC number, with the lookup example for what a number
   belongs to; by CPF/CNPJ or name, with the client lookup example and
   then the one for a client's UCs. Then run the lookup example for a
   UC's invoice with its boletos, for the period (MMM/YYYY) the user named.
   It lists the boletos newest first, each with its status and whether it
   can be canceled now. If several UCs or invoices match, list them
   (period, UC, total, boleto status) and ask which one.
2. **Make sure it is a cancel.** Users say "cobrança" for the invoice and
   for one of its boletos, and "dar baixa" or "baixar" for a cancel, a
   payment or a download. Unless they said "cancelar" or "cancela" with
   "boleto", ask one numbered question built from what you found, such as:

   > Sobre a "cobrança" de FEV/2026 da UC 3.012.345.678-90, você quer: (1) cancelar no banco o boleto Sicoob de R$ 250,00 com vencimento 10/03/2026 — a fatura continua no Helios e isso não pode ser desfeito; ou (2) excluir a fatura, o que hoje só é possível no Helios web?

   Then, by the answer:
   - they want the boleto canceled at the bank: go on;
   - the client paid this boleto: don't cancel it; report its status;
   - the client paid another way (cash, transfer): the payment is recorded
     in the Helios web app ("Confirmar pagamento"), which first needs the
     open boleto canceled. Cancel it only if the user then confirms;
   - they want the file ("me manda o boleto", "baixar o PDF"): no tool
     does that; give the boleto's status and point to the Helios web app;
   - they want the invoice removed: only the Helios web app deletes
     invoices.
3. **Pick the boleto.** A boleto's `cancel_allowed` says whether
   `cancel_bank_slip` can cancel it now: offer only boletos where it is
   true. Show each boleto's status to the user as:

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

   If no boleto can be canceled, tell the user why for each boleto (or
   that the invoice has no boleto) and stop. Present each `cancel_reason`
   per boleto, for example `Boleto com vencimento 10/03/2026: <reason>`, so
   the user doesn't read it as being about the whole invoice. If more than
   one can be canceled, list them and ask which. A boleto's state can
   change between calls, so `cancel_bank_slip` has the final say: if its
   answer disagrees with the lookup, follow `cancel_bank_slip`.
4. **Confirm the exact boleto.** Name the bank, the amount, the due date,
   the invoice's period, the UC and the client, say what happens and what
   doesn't, and wait for a clear "sim" for this boleto:

   > Vou cancelar no Sicoob o **boleto** de R$ 1.250,00 com vencimento 10/03/2026 da fatura de FEV/2026 (UC 3.012.345.678-90, Maria Exemplo, CPF 123.456.789-09). A fatura continua no Helios e esse boleto não poderá mais ser pago. Confirma?

   When it matters, add that a boleto issued later for this invoice gets a
   new barcode, so the client must not use the old one, and that a paid
   boleto, or one whose payment is clearing, can't be canceled.
5. **Cancel** with `cancel_bank_slip`, passing the invoice uuid and the
   boleto's billing uuid.
6. **Report** the result's `outcome` in pt-BR:
   - `canceled`: the bank canceled the boleto;
   - `already_canceled`: the boleto was already canceled, by an earlier
     attempt or by someone else. Nothing else changed.

   Then look the invoice up again, until the result is `current`, and
   check its other boletos:
   - one that can still be canceled can still be paid: tell the user.
     Cancel it too only if the user asks, going through steps 4 to 6 for
     it;
   - one with status `processing` is still being registered at the bank
     and may become payable, and one with status `canceling` can be paid
     until the bank confirms the cancel: mention it and suggest checking
     it again later.

To cancel several boletos, list them all and get one confirmation for the
list. Then cancel them one at a time and report each `outcome` as in
step 6. In a confirmed batch, skip step 6's lookup after each cancel, and
run it once per invoice involved, after the last cancel. Stop at the first
`ServerError` and check it as below before going on.

## When a call fails

The error text starts with its type; the message after it is in pt-BR and
can be shown to the user as it is. Recognize the cases below by their
meaning: the exact wording may change.

- `ValidationError`: an argument has the wrong format. Take both uuids
  from the lookup of the invoice with its boletos and retry.
- `NotFoundError`: the invoice or the boleto doesn't exist in this
  company, or the boleto isn't one of that invoice's. Look the invoice up
  again and take both uuids from it; never retry with a guessed uuid.
- `ConflictError`: nothing was canceled, and repeating the call now would
  fail the same way. Show the message and act on what it says:
  - **the boleto's status forbids it** (paid, payment clearing, returned,
    or a cancel already in progress): if a cancel is in progress, look the
    boleto up later to see whether it reached `canceled`;
  - **still being registered at the bank** ("ainda está sendo registrado
    no banco"): wait a few minutes, look the invoice up again, and retry
    only if the user still wants it and the boleto can now be canceled;
  - **not found at the bank** ("não foi encontrado no banco"): tell the
    user to check this boleto with the bank;
  - **in settlement or already settled by the bank** ("em liquidação ou
    já liquidado"): the client has probably paid it, even if Helios
    doesn't show the payment yet; tell the user;
  - **refused by the bank** ("O banco recusou o cancelamento"): show the
    bank's answer as it is; the user should check with the bank.
- `ServerError` or a timeout from `cancel_bank_slip`: the outcome may be
  unknown, and the bank may have canceled the boleto. Look the invoice up
  again before anything else, until the result is `current`:
  - `canceled`: the cancel worked; tell the user, and do step 6's check
    of the invoice's other boletos;
  - `canceling`: still in progress, so check again later;
  - `paid`, `processing_payment` or `returned`: the client paid while the
    cancel was running. Don't retry; tell the user to check the boleto
    with the bank;
  - still open, and it can be canceled: Helios recorded no cancel. If the
    message is about the company's bank settings ("configuração
    bancária"), the request never left Helios and those settings need
    fixing: tell the user, and retry only when they say the settings were
    fixed. Any other message (the bank was unavailable, or the
    communication failed, possibly after the request was sent) means the
    bank may have acted: tell the user, and retry `cancel_bank_slip` only
    with their OK. The retry is safe: it cancels the boleto, or answers
    `already_canceled` or one of the refusals above if the bank had
    already canceled it.
- `ServerError` from `run_analytics_query`: lookups are temporarily
  unavailable, so the boleto can't be resolved or checked here. Don't
  cancel anything; tell the user to try again in a few minutes or to use
  the Helios web app.
