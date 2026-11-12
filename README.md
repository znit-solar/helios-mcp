# Helios MCP

Conecte o seu assistente de IA (Claude, ChatGPT ou Codex) ao **Helios**, o
sistema de gestão de energia solar. Pelo assistente você consulta e cadastra
clientes, instalações CEMIG, unidades consumidoras (UCs) e contatos, e
consulta cobranças e boletos, contas de energia, relatórios GD e unidades
geradoras (somente leitura), conversando em português, sem abrir o sistema.

A conexão usa o protocolo [MCP](https://modelcontextprotocol.io) e o seu
login do Helios: o assistente só vê e altera os dados da empresa da sua
conta, com as mesmas permissões que você tem no sistema.

**Endereço do servidor MCP:**

```text
https://helios.datastrata.app/api/mcp/mcp
```

## Antes de começar

- Você precisa de uma conta no Helios (e-mail e senha). As contas são
  criadas pelo administrador do Helios: se você ainda não tem uma, peça
  pelo contato em [Ajuda](#ajuda).
- Na primeira conexão o assistente abre a tela de login do Helios no
  navegador; entre com a sua conta e autorize o acesso. Nenhuma senha ou
  chave fica salva neste repositório ou na configuração do assistente.
- Escolha **um** jeito de conectar cada assistente. Instalar o plugin e
  também configurar o servidor à mão duplica as ferramentas.

## Claude Code

O plugin instala o servidor do Helios e as skills (roteiros de trabalho,
veja [abaixo](#skills)).

1. Adicione este repositório como marketplace e instale o plugin:

   ```text
   /plugin marketplace add znit-solar/helios-mcp
   /plugin install helios@helios-mcp
   ```

   Pelo terminal, o equivalente é
   `claude plugin marketplace add znit-solar/helios-mcp` e
   `claude plugin install helios@helios-mcp`.
2. Se o Claude Code pedir, rode `/reload-plugins`.
3. Digite `/mcp`, escolha **helios** e selecione **Authenticate**. Faça
   login no Helios na janela do navegador que abrir.

Para atualizar, use `/plugin marketplace update helios-mcp`.

Se preferir só o servidor, sem as skills:

```bash
claude mcp add --transport http helios https://helios.datastrata.app/api/mcp/mcp
```

Depois faça o login pelo `/mcp`, como no passo 3.

## Claude Desktop e claude.ai

O Claude Desktop e o claude.ai aceitam plugins. O plugin instala o
servidor do Helios e as skills (roteiros de trabalho, veja
[abaixo](#skills)). Ele fica salvo na sua conta Claude e passa a valer no
app para desktop, no site, no celular, no Cowork e no Claude Code.

### Com o plugin (recomendado)

1. Abra **Customize** (Personalizar) e depois **Plugins**.
2. Clique em **Add** (Adicionar) e em **Add marketplace**.
3. Digite `znit-solar/helios-mcp` (ou
   `https://github.com/znit-solar/helios-mcp`) e confirme.
4. Em **Discover**, escolha o plugin **Helios** e clique em **Add**
   (ou **Install**).
5. Abra o plugin, vá à aba **Connectors** e clique em **Connect** no
   Helios; faça login no Helios na janela do navegador. Instalar o plugin
   não conecta o servidor sozinho.
6. Numa nova conversa, digite `/` para ver as skills do plugin.

Para atualizar, abra o marketplace e clique em **Check for updates** (ou
ative **Sync automatically**).

Nos planos Team e Enterprise, um Owner da organização pode restringir os
marketplaces e os conectores permitidos (em **Organization settings**);
se não conseguir adicionar o marketplace ou o conector, peça a ele.
Depois disso, cada pessoa clica em **Connect** e entra com a própria
conta do Helios.

### Só o servidor, sem as skills

O servidor também entra como um conector personalizado:

1. Abra **Customize > Connectors** (Personalizar > Conectores).
2. Clique em **Add** e depois em **Add custom connector**.
3. Em nome, digite `Helios`; em URL, cole
   `https://helios.datastrata.app/api/mcp/mcp`. Clique em **Continue**.
4. Mantenha as opções de autenticação marcadas como **Detected**. Se o
   Claude perguntar pelo cliente OAuth, escolha **Register automatically**.
5. Clique em **Add** e depois em **Connect**, e faça login no Helios.

O plano gratuito do Claude permite um conector personalizado, que pode ser
o Helios. Nos planos Team e Enterprise, só um Owner da organização pode
adicionar o conector (em **Organization settings > Connectors**).

## Codex (CLI, app e extensão de IDE)

### Com o plugin (CLI e app)

O plugin instala o servidor e as skills.

1. Adicione o marketplace:

   ```bash
   codex plugin marketplace add znit-solar/helios-mcp
   ```

2. No Codex CLI, digite `/plugins`, instale **Helios** e faça login no
   Helios quando pedir. No app, instale pela aba **Plugins**.
3. Comece uma nova sessão para carregar as ferramentas e as skills.

### Só o servidor (CLI, app e extensão de IDE)

A extensão de IDE não instala plugins, mas usa a mesma configuração do
CLI. Adicione ao arquivo `~/.codex/config.toml`:

```toml
[mcp_servers.helios]
url = "https://helios.datastrata.app/api/mcp/mcp"
# Pede confirmação antes de qualquer ferramenta que altera dados.
default_tools_approval_mode = "writes"
```

Ou, pelo terminal:
`codex mcp add helios --url https://helios.datastrata.app/api/mcp/mcp`.

Depois faça o login:

```bash
codex mcp login helios
```

Em um terminal sem navegador, use `codex mcp login helios --no-browser`,
abra o link exibido, faça login e cole de volta no terminal o endereço
completo para onde o navegador foi redirecionado. No Codex, `/mcp` mostra
os servidores conectados. Se o login expirar, rode `codex mcp logout helios`
e depois `codex mcp login helios`.

## ChatGPT

O servidor entra como um plugin personalizado, criado e usado no ChatGPT
na web (chatgpt.com), não no app de celular. O que funciona depende do
seu plano, segundo a documentação da OpenAI em novembro de 2026:

| Plano | Helios no ChatGPT |
|---|---|
| Free, Go | Não permite servidores MCP personalizados. Use o Claude, que permite um conector personalizado no plano gratuito |
| Plus | Não documentado pela OpenAI. Se a opção **Add custom MCP server** aparecer, as consultas devem funcionar; teste antes de contar com cadastros |
| Pro | Consultas (somente leitura). Cadastros e alterações podem ser recusados pelo ChatGPT |
| Business, Enterprise, Edu | Completo, com cadastros e alterações. Só administradores e proprietários do workspace (e, no Enterprise/Edu, membros autorizados) criam o plugin; depois cada pessoa faz login com a própria conta do Helios |

1. Abra [chatgpt.com/plugins](https://chatgpt.com/plugins).
2. Clique no **+** e em **Add custom MCP server**.
3. Em nome, digite `Helios`. Em **Connection**, informe a **Server URL**
   `https://helios.datastrata.app/api/mcp/mcp`.
4. Em autenticação, escolha **OAuth**. Não informe Client ID nem segredo: o
   Helios aceita o registro automático do cliente (DCR).
5. Leia o aviso de risco, selecione **I understand and want to continue** e
   depois **Create as a plugin**.
6. Instale o plugin criado (em **Personal**, ou no workspace onde você o
   criou) e faça login no Helios.
7. Numa conversa, digite `@` e escolha **Helios**.

O ChatGPT pede confirmação antes de cada ação que altera dados. Quando o
Helios ganhar ferramentas novas, atualize o plugin na página de detalhes
dele, nas configurações.

## Primeiro teste

Com o assistente conectado, pergunte:

> Quem sou eu no Helios?

O assistente chama a ferramenta `get_current_user` e responde com o seu
nome, e-mail e a empresa em que está trabalhando. Se aparecer um erro de
autenticação, refaça o login (passo de login do seu assistente, acima).

Depois experimente, por exemplo:

- "Busque o cliente com CPF 123.456.789-09."
- "De quem é a instalação 3000123456?"
- "O boleto de FEV/2026 desse cliente já foi pago?"
- "Cadastre um cliente novo a partir desta conta de energia."

## O que o Helios MCP faz

A lista completa de ferramentas, com o que cada uma faz e os seus
parâmetros, está em **[TOOLS.md](TOOLS.md)**. O assistente lê essas
mesmas descrições direto do servidor, então você não precisa copiar nada
para ele.

Ferramentas de consulta são marcadas como somente leitura e, em geral,
rodam sem perguntar. Ferramentas que cadastram, alteram ou excluem dados
fazem os assistentes pedirem a sua confirmação antes. Leia o que vai ser
gravado antes de confirmar.

### Skills

No Claude Code, no Claude Desktop e no Codex, o plugin traz roteiros para
tarefas comuns. O assistente os usa sozinho quando o pedido combina; no
Claude Code e no Claude Desktop você também pode chamá-los pelo nome
(digite `/`):

| Skill | Para quê |
|---|---|
| `onboard-client` | Cadastrar um cliente completo: cliente, instalação, UC e contatos, sem duplicar o que já existe |
| `look-up-client` | Encontrar um cliente por CPF/CNPJ, número da instalação ou da UC, telefone ou e-mail, e consultar as suas cobranças, boletos, contas de energia e relatórios GD |
| `manage-contacts` | Incluir, alterar, vincular, desvincular ou remover contatos das UCs |

## Desconectar

- Claude Code: `/mcp`, escolha **helios** e selecione **Clear
  authentication**, ou desinstale com `/plugin uninstall helios@helios-mcp`.
- Claude Desktop e claude.ai: **Customize > Connectors**, abra o Helios e
  desconecte ou remova. Se instalou o plugin, remova-o também em
  **Customize > Plugins** (menu do plugin > **Remove**).
- Codex: `codex mcp logout helios`.
- ChatGPT: remova o plugin em [chatgpt.com/plugins](https://chatgpt.com/plugins).

## Ajuda

Para criar uma conta no Helios ou resolver problemas de login e de acesso,
fale com **Marcos Medeiros** pelo WhatsApp:
[+55 (38) 99150-7455](https://wa.me/5538991507455).
