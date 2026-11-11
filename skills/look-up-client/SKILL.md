---
name: look-up-client
description: Find a Helios client and show its data, consumer units (UCs), CEMIG installations and contacts, plus their invoices and boletos, CEMIG energy bills, GD reports and generation units. Use when the user asks about a client, a CPF/CNPJ, an installation or UC number, who a phone or email belongs to, or a client's billing and generation ("buscar cliente", "dados do cliente", "de quem é essa UC", "cobranças do cliente", "boleto está pago?", "conta de energia de FEV/2026", "saldo de créditos").
---

# Look up a client

Read-only workflow: it never writes, so it needs no confirmation. If the
user then asks to change something, confirm the change with them before
calling any create, update or delete tool.

The Helios tools document their own parameters and formats. Read their
descriptions; this skill only gives the order of the work.

## Ground rules

- Answer the user in Brazilian Portuguese.
- Results only include the company of the logged-in user. If you have not
  called `get_current_user` in this conversation and the user seems unsure
  which company they are in, call it.
- Report what the tools return, and say so when they don't cover the
  question instead of guessing. Invoices (cobranças) and their boletos,
  CEMIG energy bills, GD reports and generation units are read-only here:
  no tool issues, cancels or changes them, so point the user to the Helios
  web app for that.
- Quote amounts, kWh and dates exactly as the tools return them. Never
  recompute a total or a balance.

## Steps

1. **Start from the most precise identifier the user gave:**
   - CPF or CNPJ: `search_clients` by document (strip dots, dashes and
     slashes);
   - installation or UC number: `search_consumer_units` by that number,
     which also returns the client and installation;
   - phone or email: `search_contacts`;
   - name only: `search_clients` by name. A name can match several clients:
     list them (name, formatted CPF/CNPJ, city) and ask which one.
2. **Fill in the rest** as the question needs:
   - the client's UCs: `search_consumer_units` by the client's document
     (each UC comes with its client, installation and contact links);
   - a UC's contact details: `search_contacts` by the UC's uuid;
   - installation details: `search_installations`;
   - invoices and boleto status: `search_invoices` by the client's or the
     UC's uuid (optionally a period or status), then `get_invoice` for line
     items, payment and the boleto;
   - CEMIG energy bills: `search_energy_bills` by the UC's uuid or the
     installation number;
   - generation, compensation and credit balance: `search_gd_reports` by
     the UC's uuid or the installation number, and a period when the user
     names one;
   - generation units (UGs): `search_generation_units` by name or
     installation number, then `get_generation_unit`.
3. **Answer.** Summarize in pt-BR: format CPF (`000.000.000-00`), CNPJ
   (`00.000.000/0000-00`) and CEP (`00000-000`), and show uuids only if the
   user asks for them. Say clearly when nothing was found, and suggest
   another identifier to try.
