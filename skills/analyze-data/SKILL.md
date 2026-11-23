---
name: analyze-data
description: Answer analytical questions about Helios data with read-only SQL — totals, counts, averages, rankings, monthly trends, comparisons and shares across clients, consumer units (UCs), invoices (cobranças), boletos, CEMIG energy bills and GD reports — and explain the result in Brazilian Portuguese, saying how current the data is. For Helios administrators. Use when the user asks for a report, summary, ranking, total or insight ("total pago em dezembro", "faturamento por mês", "inadimplência", "ranking de UCs", "evolução da compensação", "relatório", "análise", "comparar meses"), including for a company they name ("total pago pela empresa X"). For one specific client or UC, use look-up-client.
---

# Analyze data

Read-only workflow: nothing here writes, so it needs no confirmation. Only
Helios administrators can run analyses, and only super administrators can
analyze a company other than their own.

The Helios tools document their own parameters, rules and limits, and
`describe_analytics_schema` returns the data that exists. Read them; this
skill only gives the order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Never invent or estimate a number: every figure in the answer comes from a
  query result in this conversation. If the tools can't answer, say so.
- Take uuids only from tool results, never guess them. Another company's
  uuid comes only from `search_companies`.
- Write SQL only with the views and columns that `describe_analytics_schema`
  returned in this conversation. Never assume that a view or column exists.
- Do the math in SQL. Quote money and kWh exactly as returned, formatted for
  the user (R$ 1.234,56); never recompute a stored total, and never add up
  the rows of a truncated result.
- Prefer totals and counts over listing personal data (CPF, phones, e-mails,
  addresses); list them only when the user asks.
- Run queries one at a time, never in parallel, and never retry in a loop.

## Steps

1. **Who and which company.** Call `get_current_user` once per conversation.
   Its company (name and CNPJ) is the one every query reads unless the user
   names another.
2. **A company named in the request.** If the user names a company or gives
   a CNPJ ("para a <empresa>"), resolve it before anything else:
   - If it is clearly the user's own company (same CNPJ or name as
     `get_current_user` returned), use the default scope: don't pass
     `companyUuid`.
   - Otherwise call `search_companies` with the CNPJ (digits only) if the
     user gave one, or else with the name as the user wrote it. Every word
     must appear in the company's registered legal name, which is often
     longer than the name people use.
     - One match: if `isCurrentUserCompany` is true, it is the user's own
       company (default scope). If not, pass its uuid as `companyUuid` in
       every query for that question.
     - Several: list them (name, formatted CNPJ, city and state) and ask
       which one.
     - None: retry once with the most distinctive single word of the name
       (drop words like "energia", "solar", "geração", "ltda", "de"). If
       that finds several, list them. If it still finds none, the name may
       be one of the user's clients rather than a company in Helios: check
       `search_clients`, and for a client, query the user's own company
       filtered by that client. Otherwise ask the user for the CNPJ and
       search by it.
   - Only super administrators can analyze another company. For anyone else,
     `search_companies` returns only their own company, and
     `run_analytics_query` answers `NotFoundError` for another company's
     uuid. So when the named company can't be found, say so and add that
     only super administrators can analyze other companies. Never answer
     with the user's own company's data instead, and never query another
     company without a uuid from `search_companies`.
3. **Pin down the question.** Metric, period, grouping and filters. Turn
   dates into an explicit range: "1 a 31 de dezembro de 2025" is 2025-12-01
   to 2025-12-31, both included, or on a timestamp column from 2025-12-01
   up to, not including, 2026-01-01. "Total pago" is what was paid in the
   period, by payment date; "faturado" is what was invoiced, by billing
   period. Ask one short question only when the ambiguity would change the
   answer; otherwise state your assumption.
4. **Learn the data.** Call `describe_analytics_schema` without arguments
   for the overview, then with `views` for the columns of the views you
   need. Start from the closest verified example it returns, and follow its
   notes and view descriptions. The usual traps:
   - an invoice can have several boletos over time (canceled, then
     reissued): take payments from the boletos and invoiced amounts from the
     invoices, as the descriptions say;
   - a CEMIG energy bill can be stored once per generation unit: count each
     bill once, as the notes say;
   - in GD reports, consumption, compensation and credit balances belong to
     the UC and repeat on every plant's row: take the largest value per UC,
     month and time band, never add them across plants; injected energy
     belongs to each plant: add plants and report months, counting each
     report line once;
   - periods are month labels: use the date columns to sort and filter
     ranges.
5. **Query.** Call `run_analytics_query` with one SELECT that does the whole
   computation (a CTE per step is fine), with explicit join keys and
   ORDER BY, returning few rows. Pass `companyUuid` only for another company
   (step 2). Follow the SQL rules in the tool description and the notes; on
   a `ValidationError`, fix the SQL as its message says.
6. **Check the result.** If `truncated` is true the rows are incomplete:
   never add them up yourself; aggregate in SQL and query again. A total
   that comes back null means no rows matched: say so. Check that joins
   didn't multiply rows when a total looks too large, that the result's
   `company` is the one you meant, and treat the current month as partial.
7. **Answer.** Lead with the answer and the company it is about, then a
   short table (at most ~15 rows) with R$ values, kWh and MMM/YYYY periods,
   then the assumptions and filters you used. Always say how current the
   data is, from `dataAsOf`: "Dados atualizados até 05/01/2026 14:05
   (horário de Brasília)". If `freshness` is `behind`, add that the most
   recent changes may be missing. Show the SQL only when the user asks.

## When a call fails

The error text starts with its type; the message after it is in pt-BR and
can be shown to the user as it is.

- `ValidationError`: the query was rejected or failed. Read the message,
  hint and position, fix the SQL against what `describe_analytics_schema`
  returned, and retry, up to 3 times; then tell the user what failed.
- `ForbiddenError`: only administrators can run analyses. Tell the user and
  stop. For one specific client or UC, the `look-up-client` workflow still
  works.
- `NotFoundError` about the company: the company wasn't found or isn't
  available to this user, because only super administrators can analyze
  other companies. Say so; don't retry with another uuid.
- `ServerError`: analytics is unavailable or not enabled, its data is too
  far behind, or too many queries were made. Tell the user to try again in a
  few minutes, or after the wait the message gives.

## Example

"Me dê o total pago entre 1 e 31 de dezembro de 2025 para a Energia
Fictícia Ltda":

1. `get_current_user` → the user's company is another one.
2. `search_companies` with name "Energia Fictícia Ltda" → one match,
   registered as "ENERGIA FICTÍCIA GERAÇÃO SOLAR LTDA", with
   `isCurrentUserCompany` false; keep its uuid. Had it found none: retry
   with "Fictícia", then check `search_clients`, then ask for the CNPJ and
   say that only super administrators can analyze other companies.
3. `describe_analytics_schema`, then with the boleto view it describes →
   the columns for the amount paid and the payment date, and the verified
   example closest to "total pago" in a date range.
4. `run_analytics_query` with that `companyUuid`: one row with the sum of
   the amounts paid with a payment date from 2025-12-01 to 2025-12-31, and
   the number of those boletos.
5. Answer: "A Energia Fictícia Geração Solar Ltda recebeu R$ … em …
   boletos pagos entre 1 e 31 de dezembro de 2025. Dados atualizados até …
   (horário de Brasília)."
