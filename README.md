# Tibia Sentinel

Monitor visual e central de alertas para Tibia, com interface inspirada nos bots clássicos (Tibia Bot NG), mas **sem automação**: o programa só lê os pixels que já estão na tela e avisa o jogador.

- **Dashboard** com HP, mana, confiança das leituras, pré-visualização de cada região e alertas recentes
- **Captura visual** apenas das regiões selecionadas (barra de HP, barra de mana, battle list, área do jogo, chat, condições)
- **Healing Monitor**: alertas de HP/mana e **cura automática** (hotkey do jogo apertada no cliente do seu OTServer quando a barra fica abaixo do limite)
- **Alertas** configuráveis com som (volume por regra), notificação desktop, cooldown e histórico em SQLite
- **Auto Target**: lê a battle list da tela, clica no primeiro monstro (ou aperta a hotkey de atacar próximo) e usa magias e runas por hotkey, com cooldown e mana mínima; o cavebot para de andar durante o combate
- **Target Monitor**: reconhecimento de criaturas por template matching (OpenCV), com prioridade e confiança
- **Modo Demo**: cliente sintético gerado em tempo real para testar tudo sem o jogo aberto
- **Replay e gravação**: grave sessões da tela e reproduza depois (pasta de PNGs ou vídeo) para ajustar regras
- **Cavebot Navigator** (simulador): editor de waypoints, A* com múltiplos andares, escadas e portais, desvio de obstáculos, detecção de travamento com recuperação automática, rotas circulares, pausa/retomada e parada de emergência
- **Cliente OT**: o mesmo motor andando com o personagem no cliente do **seu OTServer**: teclas enviadas para a janela do cliente, posição confirmada pelo minimapa, obstáculos aprendidos ao esbarrar, loop em segundo plano e rota de teste para depot
- **Hotkeys** globais do próprio app, **Logs** ao vivo e **Settings** completas

> **Segurança:** processo independente sem elevação; não injeta DLLs, não lê nem altera memória, não intercepta rede, não modifica arquivos do jogo, só envia teclas (andar e hotkeys de cura) para a janela de cliente de OTServer conectada por você na página **Cliente OT** (o cliente oficial do Tibia com BattlEye é recusado), não envia cliques, não pede credenciais e não envia dados para fora da máquina. O projeto não promete indetectabilidade. Ferramentas de terceiros podem violar as regras do jogo: use por sua conta e risco.

## Requisitos

- Windows 10 (20H1+) ou Windows 11, x64
- Para compilar: [.NET 8 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- Para executar o `.exe` publicado (self-contained): nada além do Windows

## Compilar e executar

```powershell
cd bot-teste
dotnet restore
dotnet build -c Release
dotnet run --project src\TibiaSentinel.App -c Release
```

### Testes

```powershell
dotnet test
```

A suíte (82 testes, xUnit) cobre:

- leitura de barras em imagens sintéticas, com texto sobreposto e ruído;
- template matching, cache e múltiplas ocorrências;
- motor de alertas: limites, cooldown, supressão de regras redundantes, desconexão e criaturas;
- canais de entrega: mute, falha isolada por canal;
- configurações em JSON (ida e volta, recuperação de arquivo corrompido) e histórico SQLite;
- **ponta a ponta com sessões gravadas**: sequência de PNGs e vídeo MJPG passando pelo pipeline completo (captura → visão → regras → alertas);
- loop de monitoramento real (início, ticks e parada limpa);
- captura GDI real da tela e fábrica de fontes;
- navegação: A* (quinas, andares, escadas, portais), rota circular multiandar, desvio de obstáculos e criaturas, travamento e recuperação, pausa/emergência, rótulos de próximo waypoint, importação/exportação.
- painel do Cavebot: Start pela interface com o timer real, cinco waypoints com obstáculos e volta ao início; exceção no ciclo vira parada de emergência;
- Cliente OT: leitura do deslocamento do minimapa (com a cruz central fixa), passos confirmados e recusados, calibração, parada imediata, perda de rastreio, recusa do cliente oficial, entrega real de teclas a uma janela Win32 e **ponta a ponta com janela real** (captura GDI + `PostMessage`) percorrendo uma rota de depot e voltando ao início.

### Gerar o executável

```powershell
.\build\publish.ps1                      # testes + TibiaSentinel.exe self-contained em artifacts\publish
.\build\publish.ps1 -FrameworkDependent  # pacote menor; exige .NET 8 Desktop Runtime instalado
```

## Primeiros passos

1. Abra o app e clique em **Experimentar modo Demo** para ver tudo funcionando com o cliente sintético. Em cerca de 30 s o HP cai e os alertas de cura e de HP crítico disparam.
2. Para usar com o jogo: **Settings → Fonte de captura: Tela**, depois **Regiões monitoradas → Selecionar** para cada região. Arraste um retângulo sobre a área; nas barras, selecione só o interior.
3. Ajuste os limites em **Healing Monitor** e os demais alertas em **Alerts** (som, volume, cooldown, notificação, histórico). Use **▶ Testar** para ouvir o alerta.
4. Clique em **Iniciar monitoramento** (ou use o atalho `Ctrl+Shift+F9`).

### Target Monitor

Adicione uma criatura e defina o template: **Capturar da tela** (arraste sobre o ícone da criatura na battle list) ou um arquivo PNG. Em **Alerts**, ative a regra "Criatura monitorada" (filtrando por nome ou não). **Criaturas do modo Demo** gera templates prontos para testar com o Demo.

### Cavebot Navigator

Roda **somente no simulador de mapa em grade** (3 andares, escadas, portais e criaturas em movimento). O motor fica em `TibiaSentinel.Navigation`, desacoplado por `INavigationEnvironment`. Além do simulador `GridWorld`, há o ambiente do cliente de OTServer (página **Cliente OT**, abaixo).

- **Editar a rota:** clique no mapa para adicionar waypoints. Use o botão direito para colocar ou tirar obstáculos e as setas/WASD para mover o personagem com a navegação parada (**● Gravar** adiciona a posição atual).
- **Waypoints:** cada um tem nome, X/Y/Z, ação (andar, escada, portal), transição (ao chegar, perto, esperar N ciclos) e próximo waypoint (rótulo opcional). Rotas podem ser circulares e são importadas/exportadas em JSON (`%APPDATA%\TibiaSentinel\routes`).
- **Controles:** Start, Pausar/Retomar, Stop e **Parada de emergência** (exige liberação antes de reiniciar).
- **Testes ao vivo:** **Bloquear** põe um obstáculo no caminho (o motor recalcula), **Prender** cerca o personagem (o motor detecta o travamento e aplica a recuperação: recalcular → passo lateral → pular waypoint → erro).

### Cliente OT (seu OTServer)

Usa o motor do Cavebot no cliente do seu próprio OTServer (OTClient, OTCv8, clientes 8.x). O projeto `TibiaSentinel.OtClient` implementa `INavigationEnvironment` assim:

- **Movimento:** teclas de andar enviadas para a janela do cliente (`PostMessage`, sem precisar de foco; ou `SendInput`, que traz a janela para frente). Padrão do OTClient: setas e teclado numérico (7, 9, 1, 3) nas diagonais.
- **Confirmação do passo:** o minimapa é capturado da tela e comparado com o quadro anterior (correlação normalizada, ignorando o marcador central). O passo só conta quando o mapa se desloca casas inteiras; a tecla é reenviada enquanto o cliente está no meio de outro passo e, se nada mudar dentro do tempo limite, o passo conta como recusado.
- **Mapa:** o mapa do servidor não é conhecido. A área começa toda livre e o motor marca as casas recusadas (paredes, lockers, outros jogadores), recalculando o caminho. As posições são relativas ao ponto onde você conectou.
- **Loop em segundo plano** (`NavigationRunner`): cada passo real leva centenas de milissegundos e não trava a interface. Stop e parada de emergência interrompem na hora (a parada também envia Esc).

**Teste no depot:**

1. Abra o cliente do seu OT, entre com o personagem no depot e deixe a janela visível, com o minimapa aberto.
2. Em **Cliente OT**, escolha a janela do cliente e clique em **Selecionar** para arrastar sobre a área do minimapa (sem bordas nem botões).
3. Clique em **Conectar** e depois em **Calibrar**: o personagem dá um passo e volta, e o app mede os pixels por casa do minimapa.
4. Monte a rota. O jeito mais simples é **● Gravar rota andando**: clique, ande com o personagem pelo depot no próprio cliente e clique em **■ Parar gravação** no fim. A rota ganha um ponto onde a gravação começou, outro a cada N passos (campo **Ponto a cada**, padrão 6) e o último onde parou, e volta ao ponto inicial. Também dá para usar **Criar rota de teste** (5 pontos em anel em volta do personagem, terminando no início) ou clicar no mapa e usar **● Gravar posição**.
5. Clique em **▶ Start**. O personagem percorre os pontos e volta ao início; o mapa mostra o caminho e as casas recusadas. **⏸ Parar de andar** (ou Esc com o cliente em foco) só interrompe o andar; o **▶ Start** continua do mesmo ponto.

As diagonais são feitas com dois passos de seta: as teclas do teclado numérico viram números no chat quando ele está ativo.

### Cura automática

Em **Healing Monitor → Cura automática**, cada regra tem barra (HP ou mana), limite em %, hotkey do jogo (F1–F12, 0–9) e cooldown. A cada leitura, por barra, o app aperta a hotkey da regra mais urgente (menor %) que já saiu do cooldown. Requisitos: cliente conectado na página **Cliente OT** (as teclas vão para essa janela), barras de HP e mana selecionadas em **Settings** e monitoramento ligado.

Limitações atuais: troca de andar não é suportada (ao perder o rastreio do minimapa a navegação para com parada de emergência); o minimapa precisa ficar no mesmo zoom da calibração e sem marcações que se movem sozinhas. Use somente no seu servidor ou em servidores que permitem bots.

### Auto Target

Em **Auto Target**, clique em **⟳ Sincronizar battle list** (acha o título "Battle List" na janela do cliente) e ligue a chave. A battle list é lida pelo padrão do OTClient: barrinha de vida com contorno preto sob o nome de cada criatura e contorno vermelho no ícone do alvo. Sem alvo marcado, o app clica no primeiro da lista (ou aperta a hotkey de atacar próximo); com alvo, usa o primeiro ataque da lista que já saiu do cooldown e tem a mana mínima. Com **Parar de andar no cavebot** marcado, a rota da página **Cliente OT** pausa enquanto houver monstro e continua quando a battle list esvazia.

### Leitura das barras por número

Nos clientes em que a barra não muda de cor (ex.: mana azul mesmo vazia), use **⟳ Sincronizar** sob HP ou Mana no **Healing Monitor**: o app acha o texto "atual / máximo" da barra com o OCR do Windows e passa a ler pelos números.

### Gravar e reproduzir sessões

Em **Settings → Gravar sessão para testes**, inicie a gravação (PNGs da tela inteira + `regions.json`). Depois clique em **Usar como replay**: a fonte vira Replay e as regiões são carregadas automaticamente. Também é possível apontar para um vídeo (`.mp4`, `.avi`).

## Dados locais

Tudo fica em `%APPDATA%\TibiaSentinel`:

| Arquivo/pasta | Conteúdo |
|---|---|
| `settings.json` | Configurações (escrita atômica; arquivo corrompido é preservado e substituído pelos padrões) |
| `history.db` | Histórico de alertas (SQLite) |
| `logs\` | Logs diários (Serilog, retenção de 14 dias) |
| `sounds\` | Sons padrão gerados na primeira execução (`info`, `warning`, `critical`) |
| `templates\` | Templates de criaturas |
| `recordings\` | Sessões gravadas |

**Modo portátil:** crie um arquivo vazio `portable.flag` ao lado do `.exe` para guardar os dados em `.\data`.

## Estrutura

```
bot-teste/
├── TibiaSentinel.sln
├── Directory.Build.props
├── build/publish.ps1                 # testes + publicação do .exe
├── docs/ARCHITECTURE.md              # arquitetura, fluxo, threads, decisões
├── src/
│   ├── TibiaSentinel.Core/           # domínio, contratos, EventBus, AlertEngine, MonitoringService
│   ├── TibiaSentinel.Vision/         # OpenCV: barras, mudança, templates, replay, cliente sintético
│   ├── TibiaSentinel.Capture/        # captura GDI de regiões + fábrica de fontes
│   ├── TibiaSentinel.Infrastructure/ # JSON, SQLite, NAudio, Serilog
│   ├── TibiaSentinel.Navigation/     # Cavebot Navigator: rotas, A*, estados, recuperação, simulador
│   └── TibiaSentinel.App/            # WPF/MVVM, tema escuro, telas e serviços de UI
└── tests/TibiaSentinel.Tests/        # xUnit (unidade, visão e ponta a ponta)
```

Detalhes em [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

## Roadmap

| Marco | Conteúdo |
|---|---|
| v0.1 | Arquitetura, Dashboard, captura de regiões, HP/mana, alertas sonoros e desktop, histórico, Target Monitor básico, modo Demo, replay/gravação, hotkeys, logs |
| **v0.2 (atual)** | Cavebot Navigator no simulador: waypoints, A* multiandar, recuperação de travamento, rotas circulares, painel WPF |
| v0.3 | Detecção específica de players na battle list e de remetente de PM (hoje são heurísticas de mudança visual); calibração assistida de cores das barras |
| v0.4 | Fonte Windows Graphics Capture; overlay opcional com status; exportação do histórico (CSV) |
"# bot-sentinel" 
