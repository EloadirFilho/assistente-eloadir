# Plano de construção — Assistente pessoal

**Data:** 27/10/2026
**Origem:** pedido do dono: um assistente pessoal com acesso ao sistema da
empresa e aos dados pessoais dele, ligado ao Obsidian, que responde por um web
app e pelo WhatsApp (texto e áudio), usado de três aparelhos — PC de casa, PC
da CS e celular.
**Estado:** **rascunho para discussão.** Nada foi decidido nem construído. As
recomendações abaixo são minhas, com o porquê; as perguntas da seção 10 são
do dono responder, de preferência numa sessão de `/grill-with-docs` antes de
qualquer código.
**Onde mora:** repositório próprio e privado,
`github.com/EloadirFilho/assistente-eloadir`, separado do CRM de propósito
(seção 2). Nasceu em 27/10/2026 no repositório do CRM, como
`PLANO_ASSISTENTE_PESSOAL.md`, e veio para cá no mesmo dia.

---

## 0. O problema

Hoje a informação que o dono usa no dia está espalhada: o que a empresa está
fazendo vive no ERP e no CRM, a vida pessoal vive no Google (agenda, e-mail,
Drive) e no financeiro do casal, e o que ele pensa e anota não tem um lugar
fixo. Para saber "o que eu tenho hoje" ou "o que ficou combinado com o
arquiteto X", ele precisa abrir quatro telas.

O pedido é **um único ponto de entrada**: perguntar ou mandar fazer, por
escrito ou por voz, de onde estiver, e receber a resposta no mesmo canal.

**Critério de sucesso, proposto (o dono corrige):** *"mando um áudio no
WhatsApp perguntando o que tenho para hoje, e recebo em menos de um minuto a
agenda, os clientes que estão esperando retorno e o que anotei ontem."*

---

## 1. O desenho, em uma figura

```
  PC casa ─┐                                  ┌─► Obsidian (cofre de notas)
  PC CS ───┼─ web app (só pela rede privada) ─┤
  Celular ─┤                                  ├─► ERP e CRM (leitura)
           │                                  │
           └─ WhatsApp ──► Meta ──► webhook ──┤   ┌─► Google: agenda, e-mail, Drive
                                              │   │
                         ┌────────────────────┴───┴──┐
                         │  ASSISTENTE (servidor)     │
                         │  cérebro: Claude, via API  │
                         │  ouvido: transcrição       │
                         │  voz: síntese de fala      │
                         │  memória: o próprio cofre  │
                         └────────────────────────────┘
```

Um serviço só no servidor (Node/TypeScript, o mesmo jeito do backend do CRM,
para o dono não ter duas tecnologias para manter). Ele recebe a mensagem — do
web app ou do WhatsApp —, pergunta ao Claude, o Claude decide quais
"ferramentas" usar (ler nota, procurar no ERP, ver a agenda...), e a resposta
volta pelo mesmo canal por onde a pergunta chegou.

---

## 2. Onde hospedar — RECOMENDAÇÃO: servidor novo, pequeno e separado

O `srv1866680` é o óbvio, porque já existe. **Recomendo não usá-lo**, por três
motivos, em ordem de peso:

1. **Raio de estrago.** O assistente vai ler e-mail e mensagem de cliente —
   texto escrito por terceiros — e vai ter ferramentas que agem. Um e-mail bem
   construído para enganar o modelo (o nome técnico é *prompt injection*) é um
   risco real e conhecido. Se o assistente mora no mesmo servidor do ERP de
   produção, com Docker e banco ao lado, o pior caso é o ERP inteiro. Num
   servidor separado, o pior caso é o próprio assistente.
2. **Mistura do pessoal com a empresa.** E-mail pessoal, financeiro do casal e
   notas íntimas não deveriam estar na mesma máquina que outros agentes (o do
   ERP, o do CRM) acessam como root.
3. **Memória.** O `SERVIDOR.md` mostra 5 GiB livres depois da limpeza de
   11/09, e o CRM ainda está crescendo. O assistente quer ~1 GiB próprio
   (mais, se um dia a transcrição rodar local).

**O que proponho:** uma VPS pequena no mesmo provedor do servidor atual
(2 vCPU, 4 a 8 GiB). O que pesa é chamada de API, não processamento local.

**A alternativa, se o dono preferir começar sem custo novo:** rodar no
`srv1866680` com teto de memória (`mem_limit`, como o `edge-functions`),
porta só em `127.0.0.1`, **sem** montar o socket do Docker e com um usuário de
banco só de leitura — e mudar de máquina antes da fase 6 (ações). É aceitável
para as fases 1 a 3, que não tocam nada da empresa.

---

## 3. Os três aparelhos — RECOMENDAÇÃO: rede privada (Tailscale)

O web app vai mostrar e-mail, finanças e notas. **Ele não deveria estar aberto
na internet**, nem atrás de senha.

**Tailscale** cria uma rede privada entre os aparelhos do dono: instala nos
três (PC casa, PC CS, celular) e no servidor, e o web app só abre para quem
está nessa rede. Para quem está fora, o endereço simplesmente não existe. O
plano gratuito pessoal cobre com folga quatro aparelhos.

Isso resolve de uma vez três coisas que, de outro jeito, seriam três projetos:

| Necessidade | Com Tailscale |
|---|---|
| Login seguro no web app | Estar na rede **é** o login. Sem senha para vazar. |
| Sincronizar o Obsidian com o servidor | O Syncthing (seção 4) conversa pela rede privada |
| Assistente ler o ERP sem abrir porta do banco | O servidor novo enxerga o antigo pela rede privada, e a porta 5432 continua fechada para a internet (`SERVIDOR.md`, "Segurança") |

**A única coisa que fica pública** é o endereço do webhook do WhatsApp, porque
a Meta precisa alcançá-lo — e ele só aceita mensagem assinada pela Meta, como
o do CRM já faz.

**O custo:** no celular, o Tailscale fica ligado como VPN (consome pouco). Se
um dia estiver desligado, o WhatsApp continua funcionando — ele é o canal de
"qualquer lugar"; o web app é o de "mesa".

**O web app é um PWA**: abre no navegador, e no celular dá para "Adicionar à
tela inicial" e ele vira um ícone como de aplicativo. Sem loja, sem instalar
nada além disso.

---

## 4. Obsidian — RECOMENDAÇÃO: Syncthing, com o servidor como quarto aparelho

O cofre do Obsidian é uma pasta de arquivos `.md`. Para o assistente ler e
escrever nele, **o servidor precisa ter uma cópia viva dessa pasta**.

| Opção | Funciona nos 3 aparelhos | O servidor participa | Observação |
|---|---|---|---|
| **Syncthing** | Sim (Windows; Android com Syncthing-Fork; iPhone com Möbius Sync, pago) | **Sim** — o servidor é só mais um aparelho, ligado 24h | **Recomendado.** Gratuito, sem nuvem de terceiro, pela rede privada |
| Obsidian Sync (pago, oficial) | Sim, é o melhor nos aparelhos | Mal — não foi feito para servidor | Bom se o dono já paga; aí o servidor ficaria de fora |
| Google Drive | Nos PCs sim; no celular o Obsidian não enxerga pasta do Drive | Possível, mas desajeitado | Não recomendo |
| Git | Nos PCs sim; no celular é sofrido | Sim | Não recomendo para quem não programa |

**Regras de escrita do assistente no cofre** (para nunca estragar uma nota do
dono):

- Ele **cria** notas livremente numa pasta própria, `Assistente/`.
- Nas notas do dono, ele **só acrescenta** no fim (ex.: a nota do dia), nunca
  apaga nem reescreve, a menos que o dono peça com todas as letras.
- Toda nota que ele cria leva no topo de onde veio (`origem: whatsapp`,
  data e hora), para o dono saber o que foi ele.
- Se o Syncthing marcar conflito (dois aparelhos mexeram ao mesmo tempo), ele
  avisa e não tenta resolver sozinho.

**O cofre é também a memória do assistente.** Em vez de ele "lembrar" de coisas
num banco que o dono não enxerga, ele lembra escrevendo no cofre
(`Assistente/Memória.md`, `Assistente/Pessoas/`...). O dono pode abrir, ler,
corrigir ou apagar. Memória que o dono não consegue ler é memória em que ele
não consegue confiar.

---

## 5. WhatsApp — RECOMENDAÇÃO: número próprio do assistente, pela API oficial

Para conversar com o assistente pelo WhatsApp, **o assistente precisa de um
número dele**: o dono manda mensagem para esse número, como manda para
qualquer contato.

**Opção recomendada: número novo na Cloud API (a mesma API oficial do CRM).**

- Um chip novo (ou eSIM) só para o assistente. Não pode ser o número da
  empresa (+55 35 3551-7562, que é do CRM) nem o particular do dono (é ele
  quem vai mandar mensagem).
- **Um App da Meta separado e, de preferência, uma WABA separada**, no mesmo
  Gerenciador de Negócios. Motivo técnico: o endereço do webhook é por App.
  No mesmo App do CRM, as mensagens do assistente chegariam no backend do CRM
  — misturando o pessoal do dono com as conversas da empresa.
- **Ele só responde ao número do dono.** Qualquer outro remetente é ignorado
  em silêncio e anotado no registro. Sem isso, quem descobrir o número tem um
  assistente com acesso ao ERP.
- Tudo que o CRM já aprendeu vale aqui e se reaproveita: assinatura
  `X-Hub-Signature-256`, download de mídia, token permanente de Usuário do
  Sistema, o `subscribed_apps` que a WABA exige (README do backend).

**A regra das 24 horas.** A Meta só deixa mandar mensagem livre até 24h depois
da última mensagem do dono. Para o assistente **puxar** conversa (o resumo da
manhã, um lembrete às 15h), fora dessa janela, precisa de **modelo de
mensagem aprovado** — a mesma coisa da Fase 10 do CRM. Custa por mensagem
(categoria utilidade, centavos), e o resumo diário custa um por dia.

**A opção descartada: ponte não oficial (whatsmeow, Baileys, Evolution API).**
Funciona sem aprovação da Meta e sem custo por mensagem, mas o número pode ser
banido a qualquer momento, e o `SERVIDOR.md` conta a história do
`whatsmeow-bridge` que morreu e ficou 13 dias reconectando contra nada sem
ninguém perceber. Para algo que guarda a chave do ERP, não.

---

## 6. Áudio — ouvir e falar

**Ouvir (áudio que o dono manda → texto).** O Claude não ouve áudio
diretamente; a transcrição é um serviço à parte, chamado antes do Claude.

| Opção | Observação |
|---|---|
| **Gemini Flash** | **Recomendado para começar.** O projeto já tem a chave (`GEMINI_API_KEY`, usada no Panorama e no Robô), aceita áudio do WhatsApp direto e entende bem português falado com sotaque e ruído de fábrica |
| Whisper / transcrição da OpenAI | Excelente em português; seria mais uma conta |
| Whisper rodando no próprio servidor | Sem custo por uso e nada sai da máquina, mas lento em 2 vCPU para áudio longo. Fica como evolução |

**Falar (resposta em áudio).** Síntese de fala, depois convertida para o
formato de nota de voz do WhatsApp (`ogg/opus`, com `ffmpeg`), para aparecer
como áudio gravado e não como arquivo anexo.

| Opção | Observação |
|---|---|
| ElevenLabs | A voz mais natural em português hoje; é a que eu testaria primeiro |
| Google (Gemini TTS / Cloud TTS) | Boa, mesma conta Google que já existe |
| OpenAI TTS | Boa e barata; mais uma conta |

**Regra proposta de quando responder em áudio:** responde em áudio quando o
dono mandou áudio, ou quando ele pedir. **Lista, número, endereço, valor e
nome próprio vão também por escrito** logo abaixo — ninguém anota telefone
ouvindo áudio. Resposta longa (mais de ~1 minuto falado) vai só em texto, com
oferta de resumo em áudio.

No web app: botão de microfone (segurar para falar) e botão de "ouvir
resposta" em cada mensagem.

---

## 7. O que o assistente alcança — e com qual permissão

**Princípio: ler é livre, agir pede confirmação, e algumas coisas ele nunca
faz.**

| Fonte | Como ele chega lá | Ler | Agir |
|---|---|---|---|
| **Obsidian** | Arquivos do cofre sincronizado | Tudo | Criar em `Assistente/`, acrescentar nas notas do dono (seção 4) |
| **CRM** | API do próprio backend do CRM, com um usuário só dele | Conversas, cards, compromissos | Fase 6: criar Compromisso/Aviso **com confirmação** |
| **ERP** | Usuário de banco **só de leitura**, em views escolhidas, pela rede privada | Clientes, projetos, financeiro que o dono escolher | Nenhuma no começo |
| **Google Agenda** | API do Google, conta do dono | Eventos | Criar/mover evento com confirmação |
| **Gmail** | API do Google | Buscar e ler | Rascunho, sim. **Enviar, nunca sem confirmação** |
| **Google Drive** | API do Google | Buscar e ler | Nenhuma no começo |
| **Financeiro do casal** | A definir (seção 10) | O que o dono liberar | Nenhuma |

**Sobre o ERP, uma trava que não é minha decidir:** criar um usuário de leitura
e dar a ele acesso às views do schema `erp` **é mexer no ERP**, e a regra
registrada no README ("Conexões") é: *nunca alterar nada do schema `erp`;
mudança que cruze a fronteira é decisão de arquitetura entre os dois
projetos*. Isso tem que ser combinado com o agente do ERP, que é quem decide
quais views expor e com que filtro. Como o CRM já mora dentro do ERP
(entregas 1.34.0 em diante), o caminho mais limpo talvez seja o ERP expor uma
função própria para o assistente, como já faz com `erp.crm_meu_acesso()`.

**"Agir com confirmação" funciona assim:** o assistente descreve o que vai
fazer ("Vou criar na sua agenda: Visita Arq. Fulano, quinta 14h, Rua X.
Confirma?") e espera. No WhatsApp, com dois botões (Confirmar / Cancelar); no
web app, idem. Sem resposta em 10 minutos, desiste. **Nenhuma ação que sai da
casa — mandar e-mail, mandar WhatsApp para cliente ou para a equipe — acontece
sem esse passo, nunca**, mesmo que um e-mail ou uma mensagem lida peça.

---

## 8. O cérebro

- **Modelo:** Claude, pela API da Anthropic, com uso de ferramentas.
  `claude-sonnet-5-5` como padrão (bom equilíbrio entre qualidade e custo para
  conversa do dia a dia); `claude-haiku-5-5` para tarefas pequenas e
  repetitivas (classificar a mensagem, resumir um e-mail); `claude-opus-5-5`
  sob pedido ("pensa com calma nisso"). O backend do CRM já fala com o Claude
  (`backend/src/robo/provedores.ts`) e já conta custo por chamada
  (`backend/src/ia/precos.ts`) — os dois se reaproveitam.
- **Uma conversa contínua**, a mesma nos dois canais: o que foi dito no
  WhatsApp aparece no web app e vice-versa. Histórico num banco pequeno do
  próprio assistente (SQLite ou Postgres local), não no banco `crm`.
- **Busca no cofre:** começa com busca de texto simples nos arquivos, que para
  um cofre pessoal (centenas a poucos milhares de notas) basta. Busca por
  significado (embeddings) só se a simples deixar de achar o que devia.
- **Registro de tudo:** cada ferramenta chamada, com o que foi pedido e o que
  voltou, fica num log que o web app mostra. É como o dono descobre por que o
  assistente respondeu o que respondeu — e como se percebe um comportamento
  estranho.

---

## 9. As fases

Cada fase termina com algo que o dono usa de verdade. O critério de
"funcionou" é o teste que ele mesmo faz.

### Fase 0 — Fundação (sem código do assistente)

- Decidir as perguntas da seção 10.
- Servidor (novo ou o atual com travas), Tailscale nos 4 aparelhos, Syncthing
  nos 4 aparelhos com o cofre.
- Repositório próprio, privado — **feito em 27/10/2026** (`assistente-eloadir`).
- **Funcionou quando:** uma nota criada no celular aparece no PC de casa, no
  PC da CS e no servidor em menos de um minuto, e o endereço do servidor abre
  dos três aparelhos com o Tailscale ligado e **não abre** com ele desligado.

### Fase 1 — Web app conversando com o cofre

- Chat em texto no web app (PWA), com o Claude e as ferramentas do Obsidian
  (buscar, ler, criar, acrescentar).
- **Funcionou quando:** do celular, o dono pergunta "o que anotei sobre o
  arquiteto X?" e recebe a resposta certa citando a nota; pede "anota que
  liguei para ele hoje" e a anotação aparece no Obsidian do PC de casa.

### Fase 2 — WhatsApp em texto

- Número próprio, App e WABA próprios, webhook, só o número do dono é
  atendido, conversa compartilhada com o web app.
- **Funcionou quando:** o mesmo teste da Fase 1, pelo WhatsApp; e uma
  mensagem de outro número não recebe resposta nenhuma e aparece no log.

### Fase 3 — Áudio

- Transcrição do áudio recebido; resposta em nota de voz quando a regra da
  seção 6 manda; microfone e "ouvir" no web app.
- **Funcionou quando:** o dono grava um áudio de 30 segundos andando na
  fábrica e recebe a resposta em áudio, com os nomes e números também por
  escrito.

### Fase 4 — Leitura da empresa (ERP e CRM)

- Combinado com o agente do ERP (seção 7). Usuário só de leitura, pela rede
  privada.
- **Funcionou quando:** "quais clientes estão esperando retorno há mais de 2
  dias?" e "qual a situação do projeto da loja Y?" voltam com o mesmo que as
  telas do CRM e do ERP mostram.

### Fase 5 — Dados pessoais (Google e o que mais for decidido)

- Agenda, Gmail (leitura e rascunho), Drive (leitura).
- **Funcionou quando:** "o que tenho amanhã?" junta a agenda pessoal e os
  compromissos da empresa numa resposta só.

### Fase 6 — Agir e puxar conversa

- Ações com confirmação (agenda, compromisso/aviso no CRM, rascunho de
  e-mail). Modelo de mensagem aprovado na Meta para o resumo da manhã e os
  lembretes.
- **Antes desta fase, o assistente sai do `srv1866680`**, se tiver começado
  lá.
- **Funcionou quando:** às 7h o dono recebe o resumo do dia sem ter pedido; e
  "me lembra de ligar para o Fulano às 15h" toca às 15h.

---

## 10. Perguntas para o dono

A ordem é a do que trava primeiro.

1. **Servidor:** VPS nova e separada (recomendado) ou começar no `srv1866680`
   com as travas da seção 2?
2. **Celular:** Android ou iPhone? Muda o aplicativo do Syncthing (e se ele é
   pago) e alguns detalhes do PWA.
3. **Obsidian:** já usa? Se sim, onde está o cofre hoje e como sincroniza? Se
   já paga o Obsidian Sync, a seção 4 muda.
4. **Número do WhatsApp:** dá para comprar um chip/eSIM só para o assistente?
   E o App/WABA ficam no Gerenciador de Negócios da CS ou num seu, pessoal?
5. **Dados pessoais:** o que exatamente entra? Gmail, Google Agenda, Drive, o
   financeiro do casal, banco? E a(o) sócia(o) do financeiro do casal
   concorda que esses dados passem pelo assistente?
6. **ERP:** quais informações da empresa você quer perguntar a ele? (Isso
   decide quais views o agente do ERP expõe.)
7. **Quem usa:** só você, para sempre? Se um dia outra pessoa usar, muda a
   segurança desde o começo — melhor saber agora.
8. **Voz:** prefere voz masculina ou feminina, mais formal ou mais solta? E
   quer que ele responda em áudio sempre que você mandar áudio, ou só quando
   pedir?
9. **Teto de gasto por mês** com API (Claude, transcrição, voz, mensagens da
   Meta). Com o teto, o assistente avisa quando passar de 80%, como o
   Panorama já faz com tentativas por dia.
10. **Domínio:** um subdomínio da CS (`assistente.csfachadas.com.br`) ou um
    domínio pessoal? Como o conteúdo é pessoal, eu iria de pessoal.

---

## 11. Riscos

| Risco | O que acontece | O que o plano faz |
|---|---|---|
| E-mail ou mensagem de cliente com instrução escondida para o modelo | O assistente "obedece" e faz algo que o dono não pediu | Ler não age sozinho; toda ação passa por confirmação; ação para fora nunca é automática; servidor separado |
| Alguém descobre o número do assistente | Conversa com algo que conhece o ERP | Só o número do dono é atendido |
| Celular perdido | Quem pegar tem WhatsApp e Tailscale abertos | Tirar o aparelho do Tailscale pelo painel (um clique) e, no WhatsApp, a confirmação por PIN para perguntas financeiras (a decidir) |
| Dado de cliente e dado pessoal indo para a API de IA | Questão de LGPD e de confiança | A API da Anthropic não usa dados enviados por API para treinar modelo por padrão; mesmo assim, só vai para o modelo o trecho necessário para a pergunta, não o banco inteiro |
| Conflito de sincronização no cofre | Nota duplicada ou versão perdida | Assistente só acrescenta, nunca reescreve; conflito vira aviso |
| Serviço caiu e ninguém percebeu (a lição do `whatsmeow`) | O dono manda mensagem e nada volta | `/saude` vigiado; se o assistente ficar fora mais de 10 minutos, um aviso chega por e-mail |
| Custo sobe sem ninguém ver | Conta da API surpresa | Teto mensal (pergunta 9), custo por conversa visível no web app |
| Segredo vazado na conversa (já aconteceu três vezes no CRM) | Chave exposta | Mesma regra do README: chave só no `.env` do servidor, nunca colada na conversa nem no repositório |
