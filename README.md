# Assistente pessoal — Eloadir

Assistente pessoal do dono da CS Fachadas: responde por um web app (só na
rede privada) e pelo WhatsApp, em texto e em áudio, ligado ao cofre do
Obsidian, com leitura do sistema da empresa (ERP e CRM) e dos dados pessoais
(Google). Usado de três aparelhos: PC de casa, PC da CS e celular.

## Estado em 27/10/2026

**Só o plano existe.** Nada decidido, nada construído, nenhum servidor
contratado. O plano está em [`PLANO.md`](PLANO.md), com as recomendações e o
porquê de cada uma.

**Retomar aqui.** A primeira sessão (27/10/2026) foi aberta no Claude Code
**na nuvem**, a partir do repositório do CRM: escreveu o plano, criou este
repositório e trouxe o plano para cá. A pasta já foi baixada na Área de
Trabalho de um dos PCs (a do outro PC ainda não foi conferida). **A próxima
sessão abre no aplicativo do Claude, como projeto local nesta pasta**, e não
mais na nuvem.

## Onde fica a pasta

Nos dois PCs, na **Área de Trabalho** (`Desktop/assistente-eloadir`), fora do
Google Drive de propósito: o Drive sincronizando a pasta `.git` no meio de uma
alteração foi um dos problemas registrados no README do CRM. O que mantém os
computadores iguais é o GitHub (`git pull` / `git push`), não o Drive.

Se no PC a Área de Trabalho ficar dentro do OneDrive, o caminho é
`~/OneDrive/Desktop/assistente-eloadir` em vez de `~/Desktop/assistente-eloadir`.

### Nuvem ou local: o que cada um alcança

| Sessão | Alcança | Não alcança |
|---|---|---|
| **Local** (aplicativo, projeto nesta pasta) | A pasta do PC, o Git, a chave SSH do servidor (`~/.ssh/id_ed25519_csfachadas`) | O outro PC |
| **Nuvem** (claude.ai/code ou celular) | O GitHub: ler, escrever e enviar commit | O PC do dono, a chave SSH, o servidor |

**Trabalhou na nuvem?** Antes de mexer na pasta do PC, rode `git pull` nela.
**A chave SSH nunca vai para a nuvem.** Tudo o que toca o servidor é feito em
sessão local.

## Próximo passo

0. Abrir a próxima sessão no aplicativo do Claude, como projeto local na pasta
   `Desktop/assistente-eloadir`, e no outro PC baixar a pasta também.

1. Responder as 10 perguntas da seção 10 do `PLANO.md` (servidor, celular,
   Obsidian, número do WhatsApp, quais dados pessoais entram...). As quatro
   primeiras travam a Fase 0; o resto pode esperar a fase que depende delas.

## Projetos vizinhos

- **CRM WhatsApp** — `github.com/EloadirFilho/crm-whatsapp-cs-fachadas`.
  O assistente vai **ler** o CRM (Fase 4) e, com confirmação, criar
  Compromisso e Aviso (Fase 6). Não é parte dele.
- **ERP CS Fachadas** — leitura combinada com o agente do ERP antes (seção 7
  do plano). Nunca alterar o schema `erp` por conta própria.

**Nenhum segredo neste repositório.** Chave de API, token e senha ficam no
`.env` do servidor e nunca são colados na conversa.
