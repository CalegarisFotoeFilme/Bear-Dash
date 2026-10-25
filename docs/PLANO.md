# Bear-Dash: TripperDash++ com iPhone bloqueado, conta Apple gratuita e menos delay

## Contexto

O app oficial da RE renderiza o Google Maps no iPhone e só transmite para o Tripper Dash com a
tela acesa, o que acaba com a bateria. Além disso, o dash atualiza com atraso. O usuário já
carrega o celular na moto. Para o Waze, a decisão foi um display CarPlay separado de ~R$ 300,
ligado no pós-chave, então **o Waze fica fora deste projeto**.

A base é o [kolaCZek/TripperDashPlusPlus](https://github.com/kolaCZek/TripperDashPlusPlus)
(PolyForm Noncommercial 1.0.0, uso pessoal liberado; a Bear 650 foi confirmada por beta testers):
- **funciona com a tela bloqueada**: CoreLocation em background mantém o app vivo, o frame
  526×300 é composto via CGContext na CPU e codificado em H.264 pelo VideoToolbox
- envia RTP para `192.168.1.1:5000` a **4 fps, 512 kbps e keyframe a cada 1 s**
- mapa OSM, rotas e busca pelo MapKit, voz no modo `audio`

Dois obstáculos:
1. **Conta gratuita:** o README exige o Apple Developer pago só por causa do entitlement
   *Hotspot Configuration* (o auto-join do Wi-Fi da moto). O fork "ManualWiFi" (solai-4) não
   o remove.
2. **Delay:** 4 fps significa até 250 ms só de espera de frame. Somam-se o buffer do
   encoder/GOP de 1 s e o GPS a 1 Hz, sem interpolação, então a seta "pula" uma vez por
   segundo. O app oficial também roda a ~4 fps, então **sim, o fps baixo é parte da causa
   do delay que você percebe**. O better-dash relata que **8–12 fps funcionam** no dash.

## Respostas às suas dúvidas

**"Dá para compilar 1 vez e ficar permanente?"** Com Apple ID gratuito, não. O iOS invalida o
perfil de assinatura depois de **7 dias** e o app para de abrir até ser reassinado. (A conta
paga é **anual**, US$ 99 por ano, não mensal, e estende a validade para 1 ano.) O TrollStore,
que permitia instalação permanente, não funciona no iOS 18.6+, que é o mínimo do app. Os
caminhos de custo zero são:
1. ~~App Store/TestFlight oficial do autor~~: **não está disponível na App Store do Brasil** (descartado).
2. **SideStore (grátis):** compilamos 1 vez e o próprio iPhone reassina a cada 7 dias, sozinho
   e sem Mac. Só precisa abrir o SideStore de vez em quando.
3. Xcode no Mac uma vez por semana (1 min), como reserva.

**"O mapa é igual ao do celular, com radares e alertas?"** Não é o Google Maps. O mapa é
desenhado pelo próprio app (OpenStreetMap, claro ou escuro), mas tem **mais** informação que o
app da RE: **radares e trechos de velocidade média**, **placa de limite de velocidade da via**
(dados do OSM), alertas de clima, cards de chamada, setas de manobra e previsão de rota. A
cobertura de radares do OSM no Brasil é boa nas capitais e nas rodovias principais, mas não é
tão completa quanto a de um app pago.

**"Consigo usar o Radarbot?"** No dash, não. O iOS não deixa um app capturar a tela de outro,
ainda mais com o aparelho bloqueado. Mas **o Radarbot roda ao mesmo tempo, em segundo plano, e
dá os alertas por áudio** (no intercomunicador/fone), enquanto o TripperDash++ mostra o mapa no
dash. Os dois convivem. Junto com o radar do OSM que aparece no próprio dash, isso deve bastar.

## Decisões tomadas
- O TripperDash++ **não está na App Store do Brasil**, então seguimos com o fork próprio.
- A instalação será pelo **SideStore** (grátis), com custo total de R$ 0.
- **Ao aprovar este plano:** só faço o commit do plano em `docs/PLANO.md` no branch e o
  push. **A programação não começa agora.** Você vai pedir o início pelo Mac.

## Abordagem (execução futura, quando você pedir pelo Mac)

O repositório `Bear-Dash` vira um fork pessoal do TripperDash++, com os avisos de licença e
autoria preservados no `LICENSE`/`NOTICE`, mais 3 patches:

### Patch 1: compila com Apple ID gratuito (Personal Team)
- Remover `com.apple.developer.networking.HotspotConfiguration` do `.entitlements` e o
  código de `NEHotspotConfigurationManager` (envolto em `#if HOTSPOT_AUTOJOIN`, desligado por
  padrão).
- O app já tem um fallback de conexão manual (PR #147 upstream), que vamos aproveitar.
- Substituir por um fluxo "Wi-Fi manual": a tela de pareamento instrui a entrar uma vez em
  Ajustes → Wi-Fi → `RE_XXXX_XXXXXX`. O iOS lembra a rede e passa a entrar sozinho depois.
  O app detecta a conexão tentando o handshake em `192.168.1.1`.
- Auditar todos os outros entitlements/capabilities (NetworkExtension, App Groups, ActivityKit,
  WidgetKit, Push) e remover ou deixar opcional o que o Personal Team não assina. A Live
  Activity e o widget podem ficar se assinarem; senão, ficam atrás de uma flag de build.
- Manter os Background Modes `location` e `audio`, que funcionam na conta grátis.

### Patch 2: menos delay, mais fps
- Expor em Ajustes avançados: **fps (4/6/8/10/12)**, bitrate (512 k–1.5 M) e intervalo de
  keyframe. Começar com 8 fps / 1 Mbps / keyframe a cada 2 s.
- VideoToolbox em modo baixa latência: `kVTCompressionPropertyKey_RealTime = true`,
  `AllowFrameReordering = false` e `MaxFrameDelayCount = 0`.
- **Interpolação de posição** (dead-reckoning com velocidade e rumo do `CLLocation`) entre os
  fixes de GPS, para a seta e o mapa andarem suavemente a cada frame.
- `desiredAccuracy = kCLLocationAccuracyBestForNavigation` e
  `activityType = .automotiveNavigation`.
- Pré-busca dos tiles OSM ao longo da rota, para não ficar esperando a rede durante o trajeto.

### Patch 3: Bear 650 + bateria
- Preset "Bear 650" com o SSID salvo e o modo escuro automático.
- Pausar o encoder quando a moto estiver parada há mais de 60 s (reenviando só o keyframe a
  1 fps) para economizar CPU.

### Reassinatura a cada 7 dias (limite da conta grátis)
- Principal: **SideStore** (reassina sozinho no iPhone, sem Mac).
- Reserva: Xcode no Mac. Os dois ficam documentados em `docs/instalacao.md`.

**Custo total:** R$ 0. O Radarbot é opcional, e você usa a versão grátis ou paga que já tem.

## Arquivos principais
- `TripperDashPP/TripperDashPP.entitlements`: remover o Hotspot
- o serviço de conexão Wi-Fi/Hotspot (achar com `grep NEHotspotConfiguration`): fluxo manual
- o encoder de vídeo VideoToolbox e o scheduler de frames (achar com
  `grep VTCompressionSession`): fps e baixa latência
- o gerenciador de localização: precisão e interpolação
- a tela de Ajustes: novos controles
- `README.md`, `NOTICE`, `docs/instalacao.md`, `docs/latencia.md`

## Verificação
1. **No Mac:** o build e a assinatura com o Personal Team passam sem erro de entitlement.
2. **Em bancada:** rodar o `fake_dash` (Python, já existe no repositório) no Mac, conectar o
   iPhone e conferir com `ffplay` que o stream chega a 8 fps.
3. **Na moto parada:** entrar no Wi-Fi uma vez pelos Ajustes, o handshake conecta, bloquear o
   iPhone e o mapa continua no dash por 30 min.
4. **Rodando:** comparar 4 vs 8 vs 12 fps no mesmo trajeto, olhando a fluidez e o delay nas
   curvas, e medir a bateria (meta: < 10 %/h com a tela bloqueada).

## Limitação
Este ambiente é Linux: eu escrevo e faço push dos patches, mas **o build e o teste no iPhone
precisam ser feitos por você, no Mac**. Vou deixar o passo a passo pronto em
`docs/instalacao.md`.

## Git
Branch `claude/tripper-dash-battery-drain-bawnx8`, com um commit por patch e push nesse branch.
