# whatsapp-mcp

MCP do WhatsApp pessoal para o Claude (e qualquer cliente MCP): **buscar contatos, ler conversas,
enviar mensagens, arquivos e áudios** pela sua própria conta, conectada como um aparelho vinculado
(igual ao WhatsApp Web).

O histórico fica num SQLite local na sua máquina e só chega ao modelo quando uma tool é chamada.

Baseado no [LukasHaas/whatsapp-mcp](https://github.com/LukasHaas/whatsapp-mcp), que por sua vez é
um fork do [lharries/whatsapp-mcp](https://github.com/lharries/whatsapp-mcp). Usa a biblioteca
[whatsmeow](https://github.com/tulir/whatsmeow).

## O que esta versão muda

Correções desta versão:

- **Mensagens enviadas aparecem no histórico.** O WhatsApp não devolve ao aparelho que enviou a
  própria mensagem, então tudo que saía pelo MCP ficava invisível para as tools de leitura. Agora o
  bridge grava o envio na hora. Com isso, perguntas como "quem está esperando resposta minha?"
  passam a funcionar.
- **Nome de quem fala em grupo.** Participantes de grupo que não estão na sua agenda apareciam como
  número cru para sempre. Agora o bridge guarda o nome de exibição (ou o nome verificado, em contas
  business) que vem em cada mensagem recebida, e as tools usam esse nome.
- **Documentos com nome.** Arquivos enviados como documento chegavam como "Unknown". Agora vão com o
  nome do arquivo.

Herdado do LukasHaas: busca na agenda completa (não só nos chats recentes), resolução de LID
(contatos que aparecem como ID numérico), busca sem diferenciar acentos e maiúsculas, e sync de
histórico sob demanda.

## Requisitos

- [Go](https://go.dev/dl/) (para compilar o bridge)
- [`uv`](https://docs.astral.sh/uv/) (`brew install uv` ou `curl -LsSf https://astral.sh/uv/install.sh | sh`)
- FFmpeg, opcional: só para converter áudio em mensagem de voz (`brew install ffmpeg`)
- WhatsApp no celular, para vincular o aparelho

## Instalação

### 1. Clonar e compilar o bridge

```bash
git clone https://github.com/saviobatista/whatsapp-mcp.git ~/whatsapp-mcp
cd ~/whatsapp-mcp/whatsapp-bridge
go build -o whatsapp-bridge .
```

### 2. Vincular sua conta (uma vez)

```bash
./whatsapp-bridge
```

Aparece um QR code no terminal. No celular: **WhatsApp > Configurações > Aparelhos conectados >
Conectar um aparelho**, e escaneie. Espere aparecer `✓ Connected to WhatsApp!` e deixe alguns
minutos para o histórico sincronizar. Depois disso, `Ctrl+C`.

A sessão fica salva em `whatsapp-bridge/store/`. Das próximas vezes o bridge reconecta sem QR.

### 3. Deixar o bridge sempre rodando (macOS)

O bridge precisa estar no ar para as mensagens novas chegarem e para enviar. No macOS, o launchd
sobe o bridge no login e o reinicia se ele cair:

```bash
cd ~/whatsapp-mcp
mkdir -p ~/Library/Logs/whatsapp-mcp
sed -e "s|__BRIDGE_DIR__|$PWD/whatsapp-bridge|g" -e "s|__LOG_DIR__|$HOME/Library/Logs/whatsapp-mcp|g" \
  launchd/local.whatsapp-mcp.bridge.plist > ~/Library/LaunchAgents/local.whatsapp-mcp.bridge.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.whatsapp-mcp.bridge.plist
```

Logs em `~/Library/Logs/whatsapp-mcp/`. Em Linux, ou sem launchd, rode o bridge dentro de um
`tmux`/`screen`, ou crie um serviço do systemd.

### 4. Registrar o MCP

Claude Code:

```bash
claude mcp add whatsapp -s user -- uv --directory ~/whatsapp-mcp/whatsapp-mcp-server run main.py
```

Claude Desktop (`~/Library/Application Support/Claude/claude_desktop_config.json`) ou Cursor
(`~/.cursor/mcp.json`):

```json
{
  "mcpServers": {
    "whatsapp": {
      "command": "/opt/homebrew/bin/uv",
      "args": ["--directory", "/Users/SEU_USUARIO/whatsapp-mcp/whatsapp-mcp-server", "run", "main.py"]
    }
  }
}
```

Use o caminho completo do `uv` (`which uv`): apps de interface gráfica não herdam o `PATH` do
terminal. Reinicie o cliente depois de registrar.

## Como usar

Peça em linguagem natural. Exemplos:

- "quais conversas do WhatsApp estão esperando resposta minha?"
- "resume o grupo da família de hoje"
- "o que o Pedro me mandou essa semana?"
- "manda pra Ana: chego em 10 minutos"
- "manda esse PDF pro João: ~/Downloads/proposta.pdf"
- "baixa a foto que a Maria mandou ontem"

### Tools

| Tool | O que faz |
|------|-----------|
| `search_contacts(query)` | Busca contatos por nome ou telefone na agenda completa |
| `list_chats(query, limit, page, include_last_message, sort_by)` | Lista conversas, por atividade recente ou nome |
| `get_chat(chat_jid \| phone_number \| contact_jid)` | Dados de uma conversa; com `contact_jid`, todas as conversas que envolvem a pessoa |
| `list_messages(after, before, phone_number, sender_phone_number, chat_jid, contact_jid, query, ...)` | Mensagens com filtros por período, conversa, pessoa e texto, com contexto em volta |
| `get_message_context(message_id, before, after)` | Mensagens antes e depois de uma mensagem específica |
| `send_message(recipient, message, media_path, as_audio)` | Envia texto, arquivo ou mensagem de voz para pessoa (telefone ou JID) ou grupo (JID) |
| `download_media(message_id, chat_jid)` | Baixa a mídia de uma mensagem e devolve o caminho local |

**Destinatário:** telefone com código do país e sem `+` (ex. `5511999998888`), ou JID
(`5511999998888@s.whatsapp.net` para pessoa, `...@g.us` para grupo). O Claude acha o JID com
`search_contacts` ou `list_chats`.

**Mídia:** `send_message` com `media_path` envia imagem, vídeo ou documento. Com `as_audio=true`,
envia como mensagem de voz: arquivos `.ogg` Opus vão direto, e outros formatos são convertidos se o
FFmpeg estiver instalado. Mídia recebida guarda só os metadados até alguém chamar `download_media`.

## Como funciona

1. **Bridge em Go** (`whatsapp-bridge/`): conecta como aparelho vinculado, grava chats e
   mensagens em `store/messages.db` e expõe uma API local em `http://localhost:8080/api` para
   enviar, baixar mídia e pedir sync.
2. **Servidor MCP em Python** (`whatsapp-mcp-server/`): lê o SQLite para buscas e leitura, e chama
   a API do bridge para enviar e baixar.

## Atualizar

```bash
cd ~/whatsapp-mcp && git pull
cd whatsapp-bridge && go build -o whatsapp-bridge .
launchctl kickstart -k gui/$(id -u)/local.whatsapp-mcp.bridge
```

Sem o `kickstart`, o launchd continua rodando o binário antigo.

## Problemas comuns

| Sintoma | Solução |
|---------|---------|
| Envio falha com erro de conexão | O bridge não está rodando. Veja `launchctl list \| grep whatsapp-mcp` e os logs |
| Nenhuma mensagem aparece | Depois de vincular, o histórico leva alguns minutos para sincronizar |
| Pede QR de novo / sessão removida | Apague `whatsapp-bridge/store/whatsapp.db`, rode `./whatsapp-bridge` no terminal e escaneie de novo |
| Limite de aparelhos | Remova um aparelho antigo em Aparelhos conectados, no celular |
| Histórico dessincronizado | Apague `store/messages.db` e `store/whatsapp.db` e vincule de novo |
| `websocket: close 1006` nos logs | Queda de rede passageira; o bridge reconecta sozinho. Se repetir sem nunca conectar, verifique firewall, VPN ou proxy |

## Avisos

- `whatsapp-bridge/store/` guarda **as chaves da sua sessão e todas as suas mensagens**. Nunca
  compartilhe nem faça commit dessa pasta (ela já está no `.gitignore`).
- Este é um cliente não oficial do WhatsApp. Use com volume de pessoa, não de robô: automação em
  massa pode levar a bloqueio da conta.
- Um agente que lê mensagens de terceiros e também pode enviar está sujeito a prompt injection
  ([lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/)): uma mensagem
  maliciosa pode tentar induzir um envio ou vazar dados. `send_message` não tem confirmação
  própria, então mantenha a aprovação de tools do seu cliente MCP ligada para ela.

## Licença

MIT, ver [LICENSE](LICENSE). Copyright original de Luke Harries.
