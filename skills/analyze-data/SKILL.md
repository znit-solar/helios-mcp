---
name: analyze-data
description: Look up Helios records and answer analytical questions with read-only SQL, explained in Brazilian Portuguese. Finds a client, its consumer units (UCs), installations and contacts, whom a CPF/CNPJ, installation or UC number, phone or e-mail belongs to, an invoice (cobrança) and its boletos, a CEMIG energy bill, the GD credit balance or a generation unit; and computes totals, counts, rankings, monthly trends and comparisons across them. Use when the user asks about one record ("buscar cliente", "dados do cliente", "de quem é essa UC", "cobranças do cliente", "boleto está pago?", "conta de energia de FEV/2026", "saldo de créditos") or for a report, summary, ranking or total ("total pago em dezembro", "faturamento por mês", "inadimplência", "ranking de UCs", "evolução da compensação", "relatório", "análise", "comparar meses"), including for a company they name ("total pago pela empresa X").
---

# Look up and analyze data

Read-only workflow: nothing here writes, so it needs no confirmation. If
the user then asks to change something, follow the matching workflow
(`onboard-client`, `manage-contacts`, `cancel-bank-slip`), or resolve the
record and confirm the change with the user before any tool that creates,
updates or deletes data.

Every read goes through two tools: `describe_analytics_schema` says who the
user is and what data exists, with verified lookup and analysis examples,
and `run_analytics_query` runs one read-only SQL query. Only Helios
administrators can use them, and only super administrators can read a
company other than their own. The tools document their own parameters,
rules and limits; read them, this skill only gives the order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Never invent or estimate a value: every figure in the answer comes from a
  query result in this conversation. If the data can't answer, say so.
- Take uuids only from tool results, never guess them. Another company's
  uuid comes only from the company list of `describe_analytics_schema`.
- Write SQL only with the views and columns that `describe_analytics_schema`
  returned in this conversation, starting from its closest verified
  example. Never assume that a view or column exists.
- Do the math in SQL. Quote stored money and kWh; never recompute a stored
  total, and never add up the rows of a truncated result.
- Show values formatted as the server instructions say (CPF, CNPJ, UC
  number, phone, money, dates, periods), and uuids only when the user asks.
- In an analysis, prefer totals and counts to personal data (CPF, phones,
  e-mails, addresses), and list them only when the user asks. In a lookup,
  show only what the question needs.
- Run queries one at a time, never in parallel: combine conditions in one
  query instead. Never retry in a loop.
- Users' words don't always name the record (the server instructions list
  the terms they mix up). Take the likeliest reading and say which one you
  used: "está pago?" asks about a boleto's status, and "conta de luz" is the
  CEMIG energy bill, not the invoice.

## Steps

1. **Who and which company.** Call `describe_analytics_schema` without
   arguments once per conversation. It returns the user and their company,
   which every query reads unless the user names another; `now`, today's
   date and time in Brasília, for "hoje", "este mês" or "vencidos" (never
   your own clock); the views, notes and verified examples; and, for a
   super administrator, every company.
2. **A company named in the request.** If the user names a company or gives
   a CNPJ ("para a <empresa>"), resolve it before anything else:
   - If it is the user's own company (same CNPJ or name), use the default
     scope: don't pass `companyUuid`.
   - Otherwise, a super administrator's overview lists every company. Find
     it by CNPJ, or by the words of the name ignoring case and accents:
     legal names are often longer than the names people use.
     - One match: pass its uuid as `companyUuid` in every query for that
       question.
     - Several: list them (name, CNPJ, city) and ask which one.
     - None: the name may be one of the user's clients rather than a
       company in Helios. Look it up as a client (step 3) in the user's own
       company, or ask the user for the CNPJ.
   - Anyone else's overview has no company list, because only super
     administrators can read other companies: say so. Never answer with the
     user's own company's data instead, and never pass a `companyUuid` that
     didn't come from the list.
3. **Find one record**, when the question is about a specific client, UC,
   contact, invoice, bill or plant. Start from the most precise identifier
   the user gave and the matching lookup example of the overview, with the
   user's values in place of the sample ones:

   | The user gives or asks about | Lookup example |
   |---|---|
   | A CPF/CNPJ or a name | A client by CPF/CNPJ or name |
   | An installation or UC number | What a number belongs to: a UC and its client, a plant (UG) or an installation without a UC |
   | A phone or an e-mail | A contact by phone or e-mail, with its UCs |
   | A client's UCs and their contacts | A client's UCs with their contacts |
   | An invoice, its boletos, "está pago?" | A UC's invoice for a period, with its boletos |
   | "Conta de luz", "conta de energia" | A UC's CEMIG bill for a period |
   | "Saldo de créditos" | A UC's GD credit balance |
   | A plant, "usina" | A plant by name or number |

   Then:
   - Several records match: list them (name, CPF/CNPJ, city or UC number)
     and ask which one. None: say so, and suggest another identifier.
   - Fill in what the question needs with the other lookups, using the
     uuids just found. Don't ask the user for what a lookup can find.
   - Answer only what was asked. For an invoice, give each boleto's bank,
     amount, due date and status, and its payment date when paid; "está
     pago?" is answered by the boleto, not the invoice.
   - Freshness: when `freshness` is `current`, say nothing about it; when it
     is `behind`, say the data is as of `dataAsOf`. Before acting on the
     record (an update, a cancel), or right after a write in this
     conversation, only a `current` result counts: if it is `behind`, wait
     a moment and query again.
4. **Analyze**, for totals, rankings, trends or comparisons:
   1. *Pin down the question:* metric, period, grouping and filters. Turn
      dates into an explicit range: "1 a 31 de dezembro de 2025" is
      2025-12-01 to 2025-12-31, both included, or on a timestamp column
      from 2025-12-01 up to, not including, 2026-01-01. "Total pago" is
      what was paid in the period, by payment date; "faturado" is what was
      invoiced, by billing period. Ask one short question only when the
      ambiguity would change the answer; otherwise state your assumption.
   2. *Learn the data:* call `describe_analytics_schema` with `views` for the
      columns of the views you need. Start from the closest verified
      example, and follow the notes and view descriptions. The usual traps:
      - an invoice can have several boletos over time (canceled, then
        reissued): take payments from the boletos and invoiced amounts from
        the invoices, as the descriptions say;
      - a CEMIG energy bill can be stored once per generation unit: count
        each bill once, as the notes say;
      - in GD reports, consumption, compensation and credit balances belong
        to the UC and repeat on every plant's row: take the largest value
        per UC, month and time band, never add them across plants; injected
        energy belongs to each plant: add plants and report months,
        counting each report line once;
      - periods are month labels: use the date columns to sort and filter
        ranges.
   3. *Query:* call `run_analytics_query` with one SELECT that does the
      whole computation (a CTE per step is fine), with explicit join keys
      and ORDER BY, returning few rows. Pass `companyUuid` only for another
      company (step 2). Follow the SQL rules in the tool description and
      the notes; on a `ValidationError`, fix the SQL as its message says.
   4. *Check the result:* if `truncated` is true the rows are incomplete:
      never add them up yourself; aggregate in SQL and query again. A total
      that comes back null means no rows matched: say so. Check that joins
      didn't multiply rows when a total looks too large, that the result's
      `company` is the one you meant, and treat the current month as
      partial.
   5. *Answer:* lead with the answer and the company it is about, then a
      short table (at most about 15 rows) with formatted money, kWh and
      periods, then the assumptions and filters you used. Always say how
      current the data is, from `dataAsOf`: "Dados atualizados até
      05/01/2026 14:05 (horário de Brasília)". If `freshness` is `behind`,
      add that the most recent changes may be missing. Show the SQL only
      when the user asks.

## When a call fails

The error text starts with its type; the message after it is in pt-BR and
can be shown to the user as it is.

- `ValidationError`: the query was rejected or failed. Read the message,
  hint and position, fix the SQL against what `describe_analytics_schema`
  returned, and retry, up to 3 times; then tell the user what failed.
- `ForbiddenError`: only Helios administrators can look data up or run
  analyses here. Tell the user and stop; the Helios web app shows the same
  data.
- `NotFoundError` about the company: the company wasn't found or isn't
  available to this user, because only super administrators can read other
  companies. Say so; don't retry with another uuid.
- `ServerError`: analytics is unavailable or not enabled, its data is too
  far behind, or too many queries were made. Tell the user that lookups and
  analyses are temporarily unavailable, to try again in a few minutes (or
  after the wait the message gives), and that the Helios web app shows the
  same data. Never answer from memory, and never guess a uuid.

## Examples

"O boleto de FEV/2026 da UC 3.012.345.678-90 já foi pago?":

1. `describe_analytics_schema` → the user's company, and the lookup example
   for a UC's invoice with its boletos.
2. `run_analytics_query` with that example, the UC number and FEV/2026 →
   the invoice and its boletos, newest first.
3. Answer per boleto: bank, amount, due date and status, such as "Boleto
   Sicoob de R$ 250,00 com vencimento 10/03/2026: pago em 08/03/2026."

"Me dê o total pago entre 1 e 31 de dezembro de 2025 para a Energia
Fictícia Ltda":

1. `describe_analytics_schema` → the user is a super administrator of
   another company, and the company list has "ENERGIA FICTÍCIA GERAÇÃO
   SOLAR LTDA"; keep its uuid. Had it found none: look "Energia Fictícia"
   up as a client of the user's own company, then ask for the CNPJ.
2. `describe_analytics_schema` with the boleto view → the columns for the
   amount paid and the payment date, and the verified example closest to
   "total pago" in a date range.
3. `run_analytics_query` with that `companyUuid`: one row with the sum of
   the amounts paid with a payment date from 2025-12-01 to 2025-12-31, and
   the number of those boletos.
4. Answer: "A Energia Fictícia Geração Solar Ltda recebeu R$ … em …
   boletos pagos entre 1 e 31 de dezembro de 2025. Dados atualizados até …
   (horário de Brasília)."
