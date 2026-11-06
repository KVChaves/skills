---
name: KGames
description: Cria jogos multiplayer locais (navegador + servidor Python) com a mesma base do CTI Bombers - menu inicial (Singleplayer / Multiplayer / Personagens / Ranking), salas com lobby e "Pronto!", chat, skins enviadas por jogadores, ranking, theme.json para trocar assets e deploy opcional. Use quando o usuário quiser começar um jogo novo (qualquer gênero: luta, corrida, arena, plataforma, puzzle...) ou adaptar essa base.
---

# KGames — base para jogos multiplayer locais em Python + navegador

Base extraída do **CTI Bombers** (repositório de referência: `/home/kayran/Projetos/bomberman-prealpha`; leia lá quando precisar do código real, ele é a fonte da verdade). A ideia: tudo que **não depende do gênero** (conexão, salas, lobby, menu, skins, ranking, assets, deploy) é copiado/mantido; só a **simulação** (`game.py`), o **desenho** e os **controles** são reescritos para o jogo novo.

## 0. Como usar esta skill

1. **Pergunte só o que bloqueia**: nome do jogo, gênero/regras centrais, nº máximo de jogadores, se terá bots no singleplayer. O resto use o padrão abaixo e avise.
2. Crie o projeto novo **copiando a infraestrutura genérica** do repositório de referência (seção 3), não reescrevendo do zero. Renomeie strings ("CTI Bombers", `ctib-name`, `cti-*` classes podem ficar).
3. Reescreva apenas os pontos marcados **[GÊNERO]** (seção 4).
4. Teste com `python3 server/server.py` e duas abas do navegador (precisa de 2 jogadores para iniciar; ou use singleplayer com bot).

Regras de ouro herdadas do projeto:
- **Servidor autoritativo**: o cliente só envia *intenção* (teclas), o servidor simula e manda o estado. O cliente nunca decide dano, morte ou pontos.
- **Só biblioteca padrão do Python 3.8+** (asyncio, sem pip). O anfitrião não instala nada; os outros jogam pelo navegador. Não introduza dependências sem o usuário pedir.
- **Cliente sem build**: HTML + JS + CSS puros, sem npm/bundler.
- **Português (pt-BR)** em UI, comentários e mensagens de erro, como no original.
- **Assets trocáveis sem tocar no código**: tudo que for imagem vem de `theme.json`; sem imagem = desenho padrão (fallback), então o jogo roda mesmo sem arte.

## 1. Estrutura de pastas

```
projeto/
├── server/
│   ├── server.py      # GENÉRICO: HTTP estático + WebSocket + Hub (salas) + History + Skins + Store + chat + laço do jogo
│   ├── game.py        # [GÊNERO] Room, Player, regras, snapshot()
│   ├── maps.py        # [GÊNERO] "mapas"/fases/stages (dicionário MAPS) — o servidor só precisa de id, name, w, h
│   └── bots.py        # [GÊNERO] IA dos bots (opcional)
├── public/
│   ├── index.html     # GENÉRICO: telas title / skin / single / multi / game(+lobby) / ranking
│   ├── game.js        # genérico (rede, menus, lobby, chat, skins) + [GÊNERO] (desenho, entrada)
│   ├── style.css
│   ├── config.js      # window.CTI_CONFIG = { server: "" }  → endereço do servidor (vazio = mesmo host)
│   └── assets/
│       ├── theme.json # aponta para as imagens; tudo opcional
│       ├── LEIAME.md  # documenta cada campo do theme.json
│       ├── backgrounds/ skins/ fonts/ effects/ ...   (uma pasta por tipo de imagem)
│       └── cti-ui/    # kit de UI em pixel art (classes cti-*): reaproveitar inteiro
├── data/              # gerado em runtime (history.json, skins/) — no .gitignore
├── docs/DEPLOY.md
├── Dockerfile  render.yaml  .gitignore  .dockerignore  vercel.json  .vercelignore
└── README.md
```

## 2. O que é genérico (manter igual)

### 2.1 Servidor (`server/server.py`)
- `asyncio.start_server(handle, host, port)`; porta via `--port`/`PORT`, host via `--host`/`HOST` (padrão `0.0.0.0`).
- `handle()` roteia: `/ws` (upgrade WebSocket, com checagem de `ALLOWED_ORIGINS`), `/health` (hospedagens), `/skins/<arquivo>` (imagens enviadas pelos jogadores), qualquer outro caminho = arquivo estático de `public/` (com proteção contra path traversal via `realpath`).
- **WebSocket escrito à mão** (`ws_frame`, `read_frame`, handshake com GUID) — sem biblioteca. Limite de quadro 1 MB (cabe skin de 512 KB em base64). Responde ping (opcode 9 → pong 10).
- `Client.send_raw`: **descarta quadros** de cliente lento (`write_buffer > 256 KB`). Não bloqueie o laço do jogo por causa de um cliente.
- `Hub`: `rooms{}`, `clients`, `history`, `skins`. Métodos: `on_message` (despacha), `enter` (create/join), `leave`, `kick`, `chat`, `custom_skin`, `tick`, `menu_broadcast`.
- `game_loop()`: tick de **1/30 s** com correção de atraso (dorme quase tudo e cede o resto; se atrasar >250 ms, recomeça em vez de acelerar o jogo). A cada 30 ticks (≈1 s) faz `menu_broadcast` (lista de salas + ranking só se mudou). No Windows chama `timeBeginPeriod(1)`.
- Sala sem humanos é apagada no tick. IP real vem de `X-Forwarded-For` (atrás de proxy).
- `Store` (opcional): chave-valor no Supabase via REST (`SUPABASE_URL`/`SUPABASE_KEY`), gravação assíncrona em thread com retry. Sem as variáveis, `History` e `Skins` usam arquivos em `data/` (`HISTORY_FILE`, `SKINS_DIR` sobrescrevem). Escrita de arquivo sempre `.tmp` + `os.replace`.
- Ao iniciar imprime os endereços da LAN (`lan_ips()`), é assim que o anfitrião diz aos amigos onde entrar.

### 2.2 Protocolo (JSON por WebSocket; campo `t` = tipo)

| Cliente → servidor | Quando | Efeito |
|---|---|---|
| `rooms` | menu | pede lista de salas + ranking |
| `create` `{room?, single?, bots?, name, skin, nskins, map, rounds, time, ...opções}` | menu | cria sala (single = privada, com bots, já "pronta") e entra |
| `join` `{room, name, skin, nskins}` | menu | entra numa sala (erro se cheia / expulso / inexistente) |
| `leave` | sala | sai |
| `settings {...}` | lobby, só gerente | muda mapa/rodadas/tempo/opções |
| `ready {v}` | lobby ou final | alterna "Pronto!" |
| `skin {skin}` | lobby | troca personagem (rejeita se outro já usa) |
| `addbot` / `rmbot` | lobby, só gerente, só multiplayer | bots na sala |
| `kick {id}` | sala, só gerente | expulsa humano (nome vai pra `banned` da sala) |
| `chat {text}` | sala | 200 chars, 0,7 s entre mensagens, histórico de 40 por sala |
| `skin_add {owner,name,data,animated}` / `skin_del {id}` | qualquer lugar | skin enviada pelo jogador (≤512 KB, PNG/JPG/GIF/WebP validados por *magic bytes*; ≤5 por dono, ≤200 total) |
| `in {dx,dy}` e ações `bomb`, `det`... | partida | **[GÊNERO]** entrada |

| Servidor → cliente | Conteúdo |
|---|---|
| `hello` | `{max, maps:[{id,name,w,h}], skins}` — uma vez ao conectar |
| `rooms` | lista `Room.info()` (só salas multiplayer com gente conectada) |
| `history` | linhas do ranking (só quando mudou) |
| `you` | `{id}` do jogador na sala (limpa o chat local) |
| `state` | `Room.snapshot()` a cada tick (30/s) para todos da sala |
| `chat`, `skins`, `skin_added`, `left`, `error {msg}` | autoexplicativos |

Entrada: o cliente envia **só quando a direção muda** (`sendDir`, compara com `lastDir`); ao perder foco (`blur`) ou focar o chat, solta todas as teclas. Ações pontuais usam `!e.repeat`.

### 2.3 Sala e fases (`server/game.py` → `Room`)
Máquina de estados, **mantenha**:

```
lobby ──(≥2 conectados e todos "Pronto")──▶ countdown ──▶ playing ──(1 vivo / tempo)──▶ over
  ▲                                              ▲                                       │
  │                                              └──────── próxima rodada ◀──────────────┤
  └──(timer FINAL_TIME ou nobody ready)── final ◀──(acabaram as rodadas, sem empate)─────┘
                                            └─(todos "Pronto" de novo)─▶ new_match()
```

- `COUNTDOWN + INTRO_TIME` (título da rodada, depois 3-2-1), `ROUND_OVER_TIME`, `FINAL_TIME`.
- **Gerente (host)** = primeiro humano; se sair, herda o próximo humano. Só ele muda `settings`, adiciona/remove bots e expulsa.
- Quem entra com a partida em andamento: `alive=False, waiting=True`, joga na próxima rodada.
- Quem sai no meio: `connected=False` (fica na tabela até o lobby); no lobby é removido.
- Empate no fim das rodadas → rodadas de desempate até haver um único líder em pontos.
- Nomes duplicados viram "Nome 2"; skin duplicada na sala é trocada pela primeira livre; `MAX_PLAYERS` (6 no original) → erro "Sala cheia".
- `Room.step()` chamado 1× por tick pelo Hub; `Room.snapshot()` devolve o dict enviado a todos; `Room.info()` o resumo da lista de salas.
- **Singleplayer = uma `Room(single=True)`** privada: jogador já pronto + N bots, não aparece na lista, não pontua no ranking.

### 2.4 Pontuação e ranking
- Pontos por rodada: vencedor `WIN_POINTS`; empate: `DRAW_POINTS` aos últimos a morrer. Vencer a partida dá bônus no ranking.
- **Ranking só conta partidas multiplayer com ≥ `MIN_RANKED_HUMANS` (3) humanos** (anti-farm); `rank_points`/`rank_wins` separados dos pontos da partida.
- `History.record()` soma **por IP** (quem usa vários nomes é uma pessoa); nome exibido = o último usado. `data/history.json`.

### 2.5 Cliente (`public/game.js` + `index.html`)
Telas (`<section id="screen-X" class="screen">`, troca por `show("X")`): `title`, `skin` ("Personagens", também usada de dentro da sala), `single`, `multi`, `game`.

**Menu inicial (manter as 4 opções)**:
- **Singleplayer** → formulário (mapa, adversários, rodadas, tempo, opções) → `create{single:true}`.
- **Multiplayer** → tabela de salas abertas (inclui as já em jogo: "Entrar (próx. rodada)") + "Criar sala" (formulário inline) → `create`/`join`.
- **Personagens** → campo de nome (obrigatório; `playerInfo()` bloqueia e redireciona se vazio), grade de skins, upload de skin própria (estática ou spritesheet animado).
- **Ranking** → painel sobreposto com a tabela.

**Lobby = overlay dentro de `screen-game`** (`#lobbyBox`): título da sala, `roomSettings` (só o gerente edita; os outros veem), lista de jogadores com avatar/badges (gerente, pronto), botões `+Bot/−Bot`, **Pronto!**, Trocar skin, Sair; chat ao lado. Depois do jogo: `#finalTable` + "Pronto para nova partida".

Helpers reutilizáveis: `el(tag, props, ...kids)`, `field(label, ctrl)`, `select(opts, val, onchange)`, `toggle(val, onchange)`, `toast(msg)`, `send(obj)`, `show(screen)`, `serverUrl()`, `connect()` (reconexão com espera crescente até 5 s, banner "Sem conexão…"), `renderOnce`, `avatar(skinKey)`.

**Formulários de configuração são duplicados em 3 lugares** (single, create, lobby `rf`) e precisam ficar sincronizados: se adicionar uma opção, mexa em `buildForms`, `buildRoomForm`, no `send(...)` dos botões, em `apply_settings` (servidor, **validando e limitando**) e em `snapshot()["settings"]`.

Persistência local: `localStorage` (`ctib-name`, `ctib-skin`) sempre dentro de try/catch.

Layout: `main` = chat | coluna do canvas (+HUD de poderes, overlay do lobby) | HUD lateral (cronômetro, placar, controles, legenda). Controles de toque (analógico + botões) ativam sozinhos em `pointer: coarse`.

### 2.6 Assets e tema (`public/assets/theme.json`)
Regra: **uma pasta por tipo de imagem**; no JSON o caminho é relativo a `assets/` (`"skins/skin1.webp"`). F5 no navegador basta, sem reiniciar servidor. Campo vazio/`null` = desenho padrão.

- `pixelArt` (desliga suavização), `background` (fundo do menu), `tiles`/`maps.<id>.tiles`/`tilesets` (cenário), `items.<tipo>` (ícones), `bombs`, `flame`, `fx` (efeitos em spritesheet de uma linha), `skins`.
- **Skin** = nome de arquivo OU objeto `{name, image, frameWidth, frameHeight | grid:[cols,rows], frames, fps, rows:{d,l,r,u}, scale, offsetY, color, shape, pixelArt}`. Linhas do spritesheet: baixo, esquerda, direita, cima. Coluna 0 = parado.
- Skins numéricas (do tema) vs. string (enviadas, ids `c…`): `isCustom(key)`; completadas até `MIN_SKINS` com cores padrão se faltarem arquivos. Uploads vão para `data/skins/` (ou Supabase) e são servidos em `/skins/`. Spritesheet animado enviado: PNG/WebP 6×4.
- `loadImg(src)` falha em silêncio (console) e devolve `null` → todo `draw*` tem fallback colorido. Mantenha esse contrato para qualquer asset novo.
- Documente cada campo novo em `assets/LEIAME.md`. Ao criar um tipo novo de asset: (1) pasta, (2) campo no `theme.json`, (3) carregamento em `loadTheme()`, (4) fallback no desenho, (5) linha no LEIAME.
- UI kit `cti-ui/cti-ui.css`: variáveis no topo (`--px` espessura do pixel, paletas `--cti-*`, fontes). Para outra identidade visual, troque paleta/fontes/PNGs aqui; as classes (`cti-btn`, `cti-panel`, `cti-table`, `cti-input`, `cti-toggle`, `cti-badge`, `cti-plaque`, `cti-toast`, `cti-key`) continuam.

### 2.7 Rede e deploy
- **LAN (padrão)**: `python3 server/server.py`; o servidor entrega o site e o WebSocket na mesma porta. Firewall: liberar TCP 8000.
- **Hospedado**: site estático (Vercel, pasta `public/`) + servidor Python em Docker (Render/Fly/Railway; `Dockerfile` copia `server/` e `public/`, `CMD python server/server.py`). `public/config.js` → `server: "meu-app.onrender.com"` (ou `?server=` na URL para testar). `ALLOWED_ORIGINS` restringe quem abre WebSocket. **Uma única instância** (salas ficam na memória). Disco de hospedagem grátis é efêmero → `HISTORY_FILE`/`SKINS_DIR` em volume ou Supabase (`scripts/migrar_supabase.py`, `supabase/`).
- `.gitignore`: `data/`, `_local/`, `__pycache__/`, `.env*`.

## 3. Receita para começar um jogo novo

1. Copie do repositório de referência (sem `.git`, sem `data/`): `server/server.py`, `public/index.html`, `public/style.css`, `public/config.js`, `public/assets/cti-ui/`, `public/assets/fonts/`, `Dockerfile`, `render.yaml`, `.gitignore`, `.dockerignore`, `docs/DEPLOY.md`, `vercel.json`, `.vercelignore`. Crie `public/assets/theme.json` mínimo e `LEIAME.md`.
2. Crie `server/game.py` com o **esqueleto da seção 4.1** e `server/maps.py` com 1–2 mapas.
3. Em `server/server.py` troque só: nome do jogo nas mensagens, `from game import MAX_PLAYERS, TICK, Room`, `from maps import MAPS`. O resto não deve precisar mudar.
4. Em `public/game.js` mantenha rede/menus/lobby/skins/chat; apague o desenho e a entrada do Bomberman (`drawGrid`, `drawBombs`, `drawFlames`, teclas `Z/C/X`, tabelas de powerups etc.) e escreva os novos.
5. Em `index.html` ajuste: título, textos de controles, HUD (`#myPowers`, `#legend`) e botões de toque.
6. Rode, abra 2 abas, entre numa sala, dê "Pronto!" nas duas e confirme: contagem → jogo → fim de rodada → tela final → nova partida / lobby.
7. Atualize README (como jogar, modos, pontuação, controles) e `assets/LEIAME.md`.

## 4. [GÊNERO] — o que escrever de novo

### 4.1 Esqueleto mínimo de `Room`/`Player`

```python
TICK = 1 / 30          # ver seção 5
MAX_PLAYERS = 4
class Player:  # campos obrigatórios (o server.py e o snapshot usam): id, name, skin, client, bot, ready,
               # connected, waiting, in_round, played, wins, points, rank_points, rank_wins, kills, ip,
               # match_win, alive, x, y, spawn, died_tick;  reset() recomeça tudo da rodada
class Room:
    # obrigatórios: id, name, single, players{}, host, banned, chat[], phase, map, rounds, round_time,
    # add_player, remove_player, add_bot, apply_settings, on_message(p,m)->str|None, step(),
    # snapshot()->dict("t":"state"...), info(), humans(), connected(), preview(), to_lobby()
```
Copie `Room` do original e **substitua apenas** `update_world`, `update_player`, `snapshot` (campos do jogo), `start_round` (posicionar jogadores), `end_round` (condição de vitória) e o tratamento de ações em `on_message`. Fases, lobby, host, ranking, bots-slot ficam.

### 4.2 Checklist do que depende do gênero
- **Condição de fim de rodada** (último vivo / mais pontos no tempo / primeiro a cruzar a linha).
- **Entrada**: quais mensagens `t` existem e como o servidor guarda (`p.dx`, `p.want_*`). Ações pontuais viram flags consumidas no tick; entrada contínua vira estado.
- **Snapshot**: mande o mínimo; números arredondados (`round(v, 2/3)`); estruturas como listas curtas `[x, y, ...]`, não dicts.
- **Mapa**: contrato com o resto = `{id: {"name","w","h", ...}}`; o servidor só envia `id/name/w/h` no `hello`.
- **Bots**: função `think(room, p)` que escreve a mesma entrada que um humano escreveria (nunca altera estado diretamente).
- **Desenho**: `frame(now)` por `requestAnimationFrame`, suavizando posições (`vis{}`) entre snapshots; `drawCharacter(g, sk, face, cx, cy, size, moving, now)` já resolve spritesheet por direção.

## 5. Decisões de design que dependem do gênero (decidir com o usuário, não assumir)

A base não impõe gênero. Antes de escrever `game.py`, resolva estes pontos perguntando só o que for ambíguo:

- **Taxa de simulação e de envio**: `TICK` 1/30 serve para grade/turnos/ação lenta; ação rápida pode pedir 60 de simulação e snapshot a 20–30 Hz com interpolação no cliente. Mantenha o snapshot pequeno.
- **Modelo de entrada**: direção única (`dx,dy`), conjunto de teclas seguradas (bitmask), cliques/mira, ou comandos por turno. Contínuo = envie só quando mudar; pontual = evento com flag consumida no tick.
- **Espaço**: grade (tiles) ou mundo contínuo; colisão; câmera fixa ou seguindo jogadores.
- **Condição de fim de rodada e de partida**: último vivo, pontos, tempo, objetivo. Reaproveite `rounds`, `time`, desempate e `final`; renomeie na UI conforme o jogo (rodada, set, volta...).
- **Vida/recurso**: vidas, vida em barra, porcentagem, munição... campos do `Player` e do `snapshot`.
- **Equipes**: FFA ou times (afeta placar, pontuação e spawn).
- **Personagens vs. skins**: no original a skin é só visual e única por sala. Se o jogo tiver personagens com atributos/habilidades, separe `character` (gameplay, pode repetir ou não) de `skin` (visual) e ajuste `_free_skin`/validação em `Room.on_message`; skins enviadas por jogadores passam a ser variações de um personagem. Spritesheets com outras animações (não só 4 direções) exigem estender o objeto de skin do `theme.json`, mantendo `rows` como fallback.
- **Mapas/fases/pistas**: o contrato com o resto é `{id: {"name","w","h",...}}`; o conteúdo é livre.
- **Opções de sala**: toda opção nova vale o aviso da seção 2.5 (formulários em 3 lugares + `apply_settings` validando).
- **Bots**: opcionais; escrevem a mesma entrada que um humano. Sem bots, esconda o `+Bot/−Bot` e deixe o singleplayer como treino/solo.
- **HUD**: troque `#myPowers`/`#legend`/controles pelo que o jogo precisa.
- **Ranking**: manter a regra de ≥3 humanos ou ajustar; defina quais estatísticas contam (vitórias, pontos, abates...).

## 6. Armadilhas já resolvidas no original (não reintroduzir)

- Laço do jogo sem correção de relógio oscilava no Windows → `timeBeginPeriod(1)` + espera híbrida (sleep + `sleep(0)`).
- Cliente lento travando o servidor → descartar quadros acima de 256 KB no buffer.
- Teclas “presas” ao alt-tab ou ao digitar no chat → limpar `held` em `blur` e `chatInput.onfocus`; `keydown` ignora `INPUT/SELECT/TEXTAREA`; Espaço faz `blur()` de botão focado.
- Site em `https` só abre `wss://` (exceto localhost) → `serverUrl()` já converte.
- Hospedagem grátis dorme → cliente reconecta sozinho e mostra aviso; não trate como erro fatal.
- Sala fantasma: apague no `tick` quando não houver humanos; `leave()` do último humano remove a sala.
- Uploads: valide *magic bytes* (não a extensão), tamanho, dono, quota; nome de arquivo com `re.fullmatch(r"[\w-]+\.\w+")`; servir só dentro de `SKINS_DIR`.
- Strings do cliente entram no DOM via `textContent`/`el()`, **nunca `innerHTML`** (nomes de jogador e chat vêm de outros usuários).
- Toda mensagem do cliente é não confiável: converta com `_int()`, limite com `max/min`, ignore o que não for dict/tipo esperado.
- Rodar mais de uma instância quebra as salas (estão em memória).

## 7. Ao entregar uma etapa

Diga ao usuário: o que veio da base sem mudança, o que foi reescrito, como rodar (`python3 server/server.py`, endereço da LAN), e o que ficou para depois (assets reais, bots, deploy). Não crie documentação extra além de README/LEIAME/DEPLOY já previstos.
