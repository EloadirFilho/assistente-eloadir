# Assistente pessoal — Eloadir

Assistente pessoal do dono da CS Fachadas: responde por um web app (só na
rede privada) e pelo WhatsApp, em texto e em áudio, ligado ao cofre do
Obsidian, com leitura do sistema da empresa (ERP e CRM) e dos dados pessoais
(Google). Usado de três aparelhos: PC de casa, PC da CS e celular.

## Estado em 27/10/2026

**Só o plano existe.** Nada decidido, nada construído, nenhum servidor
contratado. O plano está em [`PLANO.md`](PLANO.md), com as recomendações e o
porquê de cada uma.

## Próximo passo

Responder as 10 perguntas da seção 10 do `PLANO.md` (servidor, celular,
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
