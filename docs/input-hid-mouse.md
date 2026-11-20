# Teclado e mouse USB no Switch: HID, cursor e sensibilidade

**Status:** pesquisa + desenho. Nada aqui foi medido no hardware deste port ainda; o que está
marcado como *confirmado* vem do código do libnx, de projetos que testaram em Switch, ou de
documentação oficial. O resto está marcado como *provável*, *desconhecido* ou *especulação*.

**Resumo.** O patch atual já lê teclado e mouse USB pelo `hid` do libnx e produz eventos winit
decentes. O jogo **já tem** um cursor próprio (sprite 48×72 desenhado a 24×36 px físicos, com
anti-aliasing e fade) e uma velocidade de cursor (`cursorSpeed`, 50–200%, em
`horizon-graphics.txt`), mas o backend ainda faz tudo uma vez por frame, também expõe a posição
absoluta de um ponteiro que o Switch expõe num espaço de 1280×720, não mede a taxa efetiva e o
cursor não tem escala nem tempo de fade configuráveis. A proposta é: um **poller dedicado a 500 Hz** (2 ms)
que drena os 17 relatórios do LIFO para um ring buffer com timestamps; o event loop do winit
consome esse ring e entrega **delta bruto** (a câmera e o `move_menu_cursor` já separam os usos),
com `CursorMoved` absoluto só para semear a posição e para touch/pad. O cursor existente ganha
`cursorScale` e `cursorIdle`; as preferências continuam em `horizon-graphics.txt` e o
`options.txt` continua vanilla. O alvo de 500 Hz é do **loop de leitura**, não uma promessa de
500 relatórios/s do hardware: a taxa efetiva depende do mouse e do stack USB do Switch e será
medida por `sampling_number`.

---

## 1. O que o patch atual faz, linha por linha

Arquivo principal: `switch/patches/winit-0.30.13/src/platform_impl/nx/native_inputs.rs`.
O patch enviado em `DuduSync-horizon-ports/patches/input-hid.patch` é uma versão anterior desse
mesmo código; a diferença relevante está anotada no fim desta seção.

### 1.1 Inicialização e chamada

Em `event_loop.rs` (`EventLoop::new`):

```rust
let rc = unsafe { ffi::hidInitialize() };          // serviço hid
unsafe { ffi::hidInitializeTouchScreen(); }
unsafe { ffi::hidInitializeKeyboard(); }           // aborta o processo se falhar
unsafe { ffi::hidInitializeMouse(); }              // idem
```

`hidInitializeKeyboard/Mouse` chamam `_hidActivateKeyboard/Mouse` e, em caso de erro, chamam
`diagAbortWithResult` — não há `Result` para tratar (confirmado no
[hid.c](https://github.com/switchbrew/libnx/blob/master/nx/source/services/hid.c)).

Depois, em `EventLoop::run`, uma vez por iteração do loop:

```rust
if focused {
    native.poll(keyboard_id, mouse_id, grabbed, &mut queued);
} else {
    native.reset_focus();
}
poll_touch(...);
while let Some(pending) = queued.pop_front() { /* entrega ao handler */ }
```

Ou seja: **uma leitura de teclado e mouse por frame**, e só com a janela focada. `poll_touch`
continua rodando mesmo sem foco, mas o applet não recebe toque nesse estado.

### 1.2 O que `poll` faz

```rust
let mut keys = [HidKeyboardState::default(); 17];
let mut mice = [HidMouseState::default(); 17];
let key_count   = hidGetKeyboardStates(keys.as_mut_ptr(), 17).min(17);
let mouse_count = hidGetMouseStates(mice.as_mut_ptr(), 17).min(17);
```

- `REPORTS = 17`, o tamanho do LIFO do libnx.
- Em `resync` (primeiro poll, perda de foco, mouse recém-conectado), corta para **1** relatório:
  pega só o mais novo e ignora o backlog, para não replayar teclas/botões que já terminaram.
- Itera `keys[..n].iter().rev()` e `mice[..n].iter().rev()`: o libnx devolve **o mais novo em
  `[0]`**, então o `rev()` processa do mais antigo para o mais novo.
- Só processa se `report.sampling_number > self.keyboard.sampling_number` (ou `self.mouse...`).
  É isso que evita contar o mesmo relatório várias vezes, já que o LIFO continua devolvendo os
  últimos 17 mesmo sem novidade.

### 1.3 Teclado

`keyboard_report` faz diff dos quatro `u64` de `keys` (256 usages HID), emite
`WindowEvent::ModifiersChanged` quando os bits de modificador mudam e `KeyboardInput` para cada
tecla que entrou/saiu. O auto-repeat é do próprio patch: 400 ms de atraso, 33 ms de intervalo,
exceto Caps Lock/Scroll Lock/Num Lock. O caractere é US HID/ASCII; `physical_key` é layout-independente.

### 1.4 Mouse

`mouse_report` emite, nesta ordem:

1. `DeviceEvent::MouseMotion { delta: (dx, dy) }` com os deltas crus do relatório (a câmera do
   jogo aplica a sensibilidade vanilla).
2. `WindowEvent::CursorMoved` com `cursor(&report)`, mas **só quando `!grabbed`** (nos menus).
3. `WindowEvent::MouseWheel` com `LineDelta(wheel_delta_x, wheel_delta_y)`.
4. `WindowEvent::MouseInput` para os bits 0..4 (Left, Right, Middle, Forward, Back), por diff
   contra `self.buttons`; ao desconectar, zera os botões e emite os releases.

**O backend não desenha cursor, mas o jogo já desenha um.** O `native_inputs.rs` só emite
eventos. No jogo, `draw_mouse_cursor` (`game.rs`) desenha o sprite `horizon/cursor` a 24×36 px
físicos, com fade depois de 2 s, e `move_menu_cursor` move `self.cursor` a partir dos deltas
crus do `DeviceEvent::MouseMotion`, usando `cursorSpeed` (50–200%) de `horizon-graphics.txt`.
O `CursorMoved` absoluto só **semeia** a posição no primeiro relatório (`mouse_have_pos`): depois
disso o jogo ignora o ponteiro absoluto para a aceleração do sistema não brigar com o
`cursorSpeed`. Ou seja, a arquitetura “delta para o cursor, absoluto só para semear” já existe;
o que falta é escala/fade configuráveis, expor a velocidade na página Options → Mouse e medir a
taxa. O patch do backend sozinho nunca desenhou cursor; o cursor é do jogo.

`cursor()`:

```rust
let screen = super::current_size();               // 1280x720 ou 1920x1080
let x = report.x.clamp(0, 1279) as f64 * screen.width  as f64 / 1280.0;
let y = report.y.clamp(0, 719)  as f64 * screen.height as f64 / 720.0;
```

O código local já escala para 1080p. **O patch de `DuduSync-horizon-ports` ainda usa
`WIDTH`/`HEIGHT` sem escala**; se aquele patch for aplicado em outro build, o cursor fica preso
no box de 1280×720 em docked — exatamente o bug relatado no
[moonlight-nx #62](https://github.com/rock88/moonlight-nx/issues/62). O build deste repo usa o
fonte local (o `Cargo.toml` aponta para `switch/patches/winit-0.30.13`), não o patch do
DuduSync; a comparação serve para quem for portar o patch para outro lugar.

### 1.5 Limites do patch atual

| Limite | Consequência |
|---|---|
| Uma leitura por frame | A 60 fps, 16,7 ms entre leituras; a 1000 Hz isso são ~17 relatórios, o tamanho exato do LIFO. Qualquer jitter perde o mais antigo. |
| Sem thread própria | Um hitch de mundo/geração maior que 17 ms (a 1000 Hz) ou 136 ms (a 125 Hz) perde movimento. |
| Cursor existe, mas é fixo | `draw_mouse_cursor` desenha 24×36 px físicos, sem escala e com fade fixo (2 s + 0,5 s); a velocidade só está no launcher/`horizon-graphics.txt`, não em Options → Mouse. |
| Posição absoluta `x/y` | É o ponteiro do sistema no espaço 1280×720; em docked precisa de escala, e o ponteiro pode encostar na borda. |
| Taxa sem gaps/connected | O tree atual já loga `Horizon HID: ... mouse reports/s` (1 s com `HORIZON_HID_LOG=1`, 5 s sem), mas não conta relatórios perdidos nem mostra `connected`. |
| Sem ring buffer | O que o frame não drenar, o LIFO sobrescreve. |
| Um mouse, um teclado | O shared memory tem um único LIFO de mouse e um de teclado; não há ID de dispositivo. |
| Teclado sem `attributes` | Não há como saber se o teclado foi desconectado; tecla presa só sai no próximo relatório/foco. |
| `set_cursor_position`/`set_cursor_visible` | São no-op no backend nx; não existe ponteiro do sistema para esconder. |
| Wheel em `wheel_delta_x` | Alguns mouses reportam o scroll vertical em X; o patch só olha Y. |
| Layout US | Teclado ABNT2 produz `;` em vez de `ç`, etc. O libnx não expõe o layout do sistema. |

---

## 2. API do libnx: o que é confirmado

Fonte primária: [`hid.h`](https://github.com/switchbrew/libnx/blob/master/nx/include/switch/services/hid.h)
e [`hid.c`](https://github.com/switchbrew/libnx/blob/master/nx/source/services/hid.c). A wiki
[Switchbrew HID services](https://switchbrew.org/wiki/HID_services) confirma o layout do shared memory.

### 2.1 Estruturas e assinaturas

```c
typedef struct HidMouseState {
    u64 sampling_number;
    s32 x, y;                  // ponteiro no espaço 1280x720
    s32 delta_x, delta_y;      // movimento do relatório
    s32 wheel_delta_x, wheel_delta_y;
    u32 buttons;               // HidMouseButton_Left=1<<0 ... Back=1<<4
    u32 attributes;            // bit0 Transferable, bit1 IsConnected
} HidMouseState;               // 40 bytes

typedef struct HidKeyboardState {
    u64 sampling_number;
    u64 modifiers;             // HidKeyboardModifier (Control=1<<0, Shift=1<<1, ...)
    u64 keys[4];               // 256 usages HID
} HidKeyboardState;            // 48 bytes

size_t hidGetMouseStates(HidMouseState *states, size_t count);
size_t hidGetKeyboardStates(HidKeyboardState *states, size_t count);
void   hidInitializeMouse(void);
void   hidInitializeKeyboard(void);
```

O patch local declara essas mesmas structs em `mod.rs` e tem as duas asserções de tamanho:

```rust
const _: () = assert!(size_of::<HidKeyboardState>() == 48);
const _: () = assert!(size_of::<HidMouseState>() == 40);
```

Está correto.

### 2.2 LIFO: 17 relatórios, mais novo primeiro

O shared memory do `hid` tem, para cada tipo, um `RingLifo` com `storage[17]`
([`HidMouseLifo`](https://github.com/switchbrew/libnx/blob/master/nx/include/switch/services/hid.h)).
`hidGetMouseStates` chama `_hidGetStates(..., max_states = 17, ..., count)`; a função:

- lê `header->count` (quantos relatórios válidos há, 0..17) e corta em `count`;
- lê `header->tail` e calcula a posição do mais antigo;
- escreve o **mais novo em `states[0]`** e o mais antigo no fim;
- confere `sampling_number` antes/depois da cópia e a continuidade entre relatórios; se um
  escritor passou por cima durante a leitura, recarrega `count`/`tail` e recomeça.

Consequências práticas:

- **Não existe “leitura destrutiva”**: o LIFO continua devolvendo os últimos 17; o filtro por
  `sampling_number` é obrigatório. O patch atual faz isso.
- **Profundidade = 17 relatórios**, não 17 ms. A cobertura em tempo depende da taxa:

| Taxa efetiva | Período | 17 relatórios cobrem |
|---|---:|---:|
| 125 Hz | 8 ms | 136 ms |
| 250 Hz | 4 ms | 68 ms |
| 500 Hz | 2 ms | 34 ms |
| 1000 Hz | 1 ms | 17 ms |

- A 1000 Hz, um frame de 60 fps (~16,7 ms) já chega perto dos 17 ms de histórico. Um poller a
  2 ms mantém o histórico sempre jovem e evita depender do frame.

### 2.3 Conexão, desconexão e múltiplos dispositivos

- `HidMouseAttribute_IsConnected` é o **bit 1** de `attributes` (o bit 0 é `Transferable`).
  O patch local usa `1 << 1`; correto.
- **Não há evento de conexão/desconexão** para mouse/teclado no libnx. A única fonte é o
  `attributes` de cada relatório. Se o sistema não publicar um último relatório com o bit
  limpo, o app não fica sabendo.
- **Teclado não tem `attributes`**: `HidKeyboardState` só tem `sampling_number`, `modifiers` e
  `keys`. Não dá para detectar desconexão de teclado pelo estado.
- O `HidSharedMemory` tem **uma** seção `mouse` e **uma** `keyboard` (confirmado no header:
  `HidMouseSharedMemoryFormat mouse; HidKeyboardSharedMemoryFormat keyboard;`). Não há array
  por dispositivo nem ID. Vários mouses/teclados são fundidos pelo sistema em um único fluxo;
  a política exata de fusão não é documentada (provável: OR dos estados / soma dos deltas).

### 2.4 Inicialização

`hidInitializeMouse()` e `hidInitializeKeyboard()` **abortam o processo** em caso de falha
(`diagAbortWithResult`), então o backend não tem como retornar erro. Isso já está comentado no
`event_loop.rs`. `hidInitialize()` abre o serviço e mapeia o shared memory; sem ele,
`hidGetMouseStates` aborta com `LibnxError_NotInitialized`.

### 2.5 Espaço de coordenadas: 1280×720

Isto é confirmado por vários ports independentes e por um issue de outro port:

- O patch local clampa `x` em `0..1279` e `y` em `0..719` e escala para o tamanho da janela.
- O port de Project Zomboid escala `dx * W / 1280.0` e `dy * H / 720.0`
  ([PR #12](https://github.com/leo-snczki/zomboid-switch/pull/12)).
- O port de Drastic DS faz `mouse.x * panel_width / 1280.0` e `mouse.y * panel_height / 720.0`
  ([input_sampler.c](https://github.com/NaGaa95/DrasticDS_nx/blob/master/source/input_sampler.c)).
- O issue [moonlight-nx #62](https://github.com/rock88/moonlight-nx/issues/62) relata
  exatamente o sintoma de usar `x/y` sem escala em 1080p: o cursor não passa do box 720p.

Ou seja: `x`, `y`, `delta_x` e `delta_y` estão no **espaço virtual de 1280×720**, independente
de handheld/docked. Para o cursor em 1080p, multiplique por `1920/1280 = 1,5` (ou, melhor, por
`screen.height / 720`). Para a câmera, o delta já é independente de resolução; não escale de novo.

### 2.6 O que o libnx **não** expõe

- DPI/CPI do mouse: não há campo. O sistema converte contagens em unidades do ponteiro com um
  fator fixo; um mouse de 1600 DPI anda mais que um de 400 DPI para o mesmo movimento físico.
  O ajuste de velocidade do cursor é o compensador.
- Bateria do mouse: não há campo; as APIs de bateria do libnx são para Npad.
- Layout de teclado: não há `hidGetKeyboardLayout`. O sistema tem configuração de layout USB
  desde o 6.0.0, mas o libnx entrega usages HID crus; o app decide o mapa (o patch usa US).
- Raw input “de verdade”: não há um canal documentado separado do ponteiro do sistema.
  `delta_x/y` é o mais próximo de raw que o applet recebe.

---

## 3. Taxa de relatórios: 125, 250, 500 ou 1000 Hz?

### 3.1 O que está confirmado

- **Controles oficiais e o adaptador GameCube**: 125 Hz (8 ms). O Titan Two documenta “USB Default
  125 Hz, USB Maximum 125 Hz” para o Switch
  ([wiki](https://www.consoletuner.com/wiki/doku.php?id=t2%3Ausage_guides%3Asystems%3Aswitch));
  o reverse engineering do Pro Controller com fio mede 125 Hz
  ([GBAtemp](https://gbatemp.net/threads/reverse-engineering-the-switch-pro-controller-wired-mode.475226/));
  o adaptador GameCube oficial (WUP-028) tem endpoint de 8 ms
  ([gcadapter-oc-nx](https://github.com/KakarottoCake/gcadapter-oc-nx)).
- **O intervalo USB é programado pelo módulo `usb` do Horizon**, a partir do descritor do
  endpoint. O projeto
  `gcadapter-oc-nx` só consegue mudar 8 ms → 500/1000 Hz patchando esse módulo em memória
  ([README](https://github.com/KakarottoCake/gcadapter-oc-nx)). Isso sugere que não há um clamp
  global de 125 Hz no stack; um dispositivo com endpoint de 1 ms *pode* ser pollado a 1 ms, mas
  essa parte é inferência, não medição.

### 3.2 O que não está confirmado

- A taxa efetiva de um **mouse HID genérico** (não um controle) no Switch 1. Não achei uma
  medição pública de “mouse de 1000 Hz entrega 1000 Hz no `hidGetMouseStates`”. O caminho é:
  descritor do mouse → xHCI (`usb` module) → `hid` module → shared memory. Cada etapa pode
  limitar.
- Se o `hid` module **repassa cada relatório** ou faz uma amostragem própria (ex.: 60/125 Hz)
  antes de publicar. O LIFO de 17 e os relatos de múltiplos samples por frame sugerem que
  repassa, mas não é documentado.
- Se o mouse de 1000 Hz é realmente pollado a 1 ms no Switch. O endpoint pode declarar 1 ms, mas
  não há uma medição pública do `hid` com um mouse gamer.
- Se o `hid` publica acima de 60 Hz de fato. Vários ports leem mais de um relatório por frame e
  comentam “HID samples faster than we render”
  ([nx_pointer.c](https://github.com/ChanseyIsTheBest/wmw_nx/blob/main/source/nx_pointer.c)),
  mas isso é indício, não uma especificação.
- Se há aceleração de ponteiro do sistema aplicada a `delta_x/y`, ou só a `x/y`.
- Se `sampling_number` reinicia em reconexão/sleep (o filtro `>` atual ignoraria tudo se
  reiniciasse para 0).

### 3.3 Por que 500 Hz é a meta realista

**500 Hz é a taxa do loop de leitura, não do hardware.** Uma thread pode chamar
`hidGetMouseStates` a 500 Hz — a leitura é do shared memory e não depende do frame do applet.
O loop acorda a cada 2 ms, drena os 17 relatórios e enfileira só os novos. Se o mouse entrega 1000 Hz, cada leitura vê ~2
relatórios e drena os dois; **não há perda**, só uma latência de amostragem de até 2 ms. Se o
mouse entrega 125 Hz, o loop vê o mesmo relatório ~4 vezes e o filtro por `sampling_number`
descarta as duplicatas; a taxa efetiva logada será 125 Hz.

Vantagens de 500 Hz sobre 125 Hz:

- 2 ms de latência de amostragem em vez de 8 ms.
- O LIFO de 17 cobre 34 ms (a 500 Hz) ou 17 ms (a 1000 Hz) **no momento da leitura**; o ring
  buffer do poller segura o que o frame não consumiu.
- Um hitch de 100 ms não perde movimento: o poller continua drenando para o ring.

Por que não 1000 Hz:

- Não é o CPU que pesa (ler 17×40 bytes 1000×/s é ~0,7 MB/s); é acordar a thread 1000×/s e
  competir com o renderizador num Tegra de 4 núcleos.
- A 60 fps, o ganho de 1 ms vs. 2 ms de amostragem é imperceptível; o frame já impõe 16,7 ms.
- A 1000 Hz, o LIFO cobre só 17 ms; qualquer jitter de scheduler se aproxima do limite.

**Conclusão honesta:** 500 Hz é o ponto doce do loop. A taxa efetiva de relatórios pode ser
125 Hz (mouse de escritório/dongle) ou 1000 Hz (mouse gamer), e só o log de `sampling_number`
responde. Não prometa 500 Hz ao usuário; mostre a taxa medida.

### 3.4 Como medir

No poller, a cada leitura:

```text
new = sampling_number > last_sampling
gaps += sampling_number - last_sampling - 1   // relatórios que o LIFO já sobrescreveu
last_sampling = sampling_number
```

A cada 5 s:

```text
hid: mouse 124.8 Hz (624 relatórios, 0 gaps, 0 dropped, 2 no ring, connected=true)
hid: mouse 997.3 Hz (4987 relatórios, 3 gaps, 0 dropped, 1 no ring, connected=true)
```

`sampling_number` é a única fonte de verdade; `hidGetMouseStates` devolver 17 sempre não
significa 17 relatórios novos. Também logue os primeiros 20 relatórios com
`sampling`, `dx`, `dy`, `x`, `y`, `buttons`, `attributes` para conferir o espaço de coordenadas
e o wheel.

---

## 4. Projetos existentes: o que aproveitar

| Projeto | URL | Licença | O que faz | O que aproveitar |
|---|---|---|---|---|
| **libnx** | https://github.com/switchbrew/libnx | ISC | API HID, LIFO de 17, `hidGetMouseStates` | A API e a semântica de drenagem; é a base de tudo |
| **zomboid-switch PR #12** | https://github.com/leo-snczki/zomboid-switch/pull/12 | sem licença declarada | Shim GLFW com teclado/mouse USB, 17 samples, cursor acumulado, release ao desconectar, handover para o pad | **Ideias only** (sem licença): drenar 17, `ms_seen`, `mouse_travel > 24`, escala `W/1280`, release no unplug |
| **key2con** | https://github.com/EnzoBernardinoMartins/key2con | MIT | Sysmodule que transforma teclado/mouse em controle virtual; loop de 1 ms, 7 samples, acumula deltas | Código compatível: loop de 1 ms, filtro por `sampling_number`, acumulação de deltas, sensibilidade |
| **wmw_nx / papersplease_nx** | https://github.com/ChanseyIsTheBest/wmw_nx | MIT | Módulo `nx_pointer.c`: touch + mouse USB + giroscópio + cursor PNG + sensibilidade em `pointer.cfg` | Código compatível: cursor PNG com hotspot, sensibilidade persistida, escala docked, `wheel_y ?: wheel_x`, presença por atividade |
| **Moonlight-Switch** | https://github.com/XITRIX/Moonlight-Switch | GPL-3.0 | Cliente de streaming; mouse relativo, velocidade e scroll configuráveis | **Ideias only** (GPL): separar velocidade do cursor da sensibilidade da câmera, quantização do scroll |
| **moonlight-nx** | https://github.com/rock88/moonlight-nx | GPL-3.0 | Port original; `SwitchMouseFrontend` lê 1 sample | **Ideias only**: o issue #62 documenta o box 720p; o código lê só 1 sample (não copiar) |
| **RetroArch** | https://github.com/libretro/RetroArch/blob/master/input/drivers/switch_input.c | GPL-3.0 | Driver de input do Switch; mouse físico + touch | **Ideias only**: diff de botões, wheel; é um exemplo do que **não** fazer (1 sample/frame) |
| **Sphaira** | https://github.com/NaGaa95/sphaira | GPL-3.0 | Menu homebrew; Joy-Con 2 como mouse e cursor próprio | **Ideias only**: cursor que some após 2 s, seta com contorno |
| **sys-hidplus** | https://github.com/luvvyamy/sys-hidplus | GPL-3.0 | Sysmodule de controle via rede | Não se aplica a mouse HID; só mostra injeção de pad virtual |
| **gcadapter-oc-nx** | https://github.com/KakarottoCake/gcadapter-oc-nx | GPL-2.0 | Patcheia o intervalo USB de 8 ms para 500/1000 Hz | **Evidência**, não código: mostra que o `usb` module programa o bInterval e que mexer em sysmodule é arriscado |
| **Joy-Con 2 Mouse** | https://github.com/ABpa-lite/joycon2mouse | sem licença | Joy-Con 2 como mouse no Switch 2 | Não se aplica ao Switch 1; referência para o mouse do Switch 2 (`padPointer`/`Nmouse`) |

### 4.1 O que cada um ensina de concreto

- **zomboid-switch**: o loop certo é `hidGetMouseStates(ms, 17)` + `for (i = n-1; i >= 0; i--)`
  + `if (ms[i].sampling_number <= ms_seen) continue;`. O release no unplug e o handover por
  “movimento > 24 px” são boas ideias. O código não tem licença; reimplemente, não copie.
- **key2con**: prova que um loop de 1 ms com `svcSleepThread(1_000_000)` é viável num
  sysmodule e que acumular deltas entre leituras funciona. É MIT; pode ser adaptado com
  atribuição.
- **wmw_nx**: o `do_mouse()` é o melhor exemplo público de drenagem de 16 samples com
  `sampling_number`, fallback de wheel e cursor PNG. É MIT; pode ser adaptado com atribuição.
- **RetroArch**: lê `hidGetMouseStates(&mouse_state, 1)` e usa `delta_x` direto; a 1000 Hz isso
  perde a maioria dos relatórios. Serve como alerta, não como modelo.
- **Moonlight-Switch**: o `InputManager` separa “mouse speed multiplier” de “scroll multiplier”
  e usa movimento relativo; a arquitetura de settings é boa referência, mas o código é GPL.

### 4.2 Licenças

- **ISC (libnx)**, **MIT (key2con, wmw_nx, papersplease_nx, DrasticDS_nx)**: podem ser
  adaptados com aviso de copyright/licença.
- **GPL-2.0/3.0 (Moonlight, RetroArch, Sphaira, sys-hidplus, gcadapter-oc)**: só ideias; copiar
  contaminaria o port (que é MIT).
- **Sem licença (zomboid-switch, Joy-Con 2 Mouse)**: não copiar; reimplementar a ideia.

---

## 5. USB, Bluetooth e o que o Switch expõe

### 5.1 USB

- Funciona **docked** e **handheld via adaptador USB-C**, segundo o PR do zomboid (testado em
  dock) e o README do wmw_nx (“A USB mouse works in both handheld and docked”).
- O Switch vê o mouse como HID; o `hid` module publica um único fluxo. Um dongle 2.4 GHz
  (Logitech Unifying, receptor de teclado/mouse) aparece como USB HID e funciona igual.
- O mouse **não** informa DPI/CPI, bateria ou ID; o sistema também não expõe isso ao applet.

### 5.2 Bluetooth

**O Switch 1 não pareia teclado/mouse Bluetooth genéricos.** A Nintendo responde oficialmente
que headphones e teclado por Bluetooth “não podem ser usados”
([Q&A coreano](https://www.nintendo.co.kr/support/switch/qna/116)); a página de Bluetooth Audio
só lista A2DP/SBC, ou seja, áudio
([Nintendo](https://www.nintendo.com/my/support/switch/accessories/bluetooth_audio.html)). O
Bluetooth do console é para controles (Joy-Con/Pro) e áudio, não para HID genérico.

Consequência para o requisito “USB/Bluetooth keyboard and mouse must work”: **um mouse/teclado
Bluetooth só funciona através do receptor USB/dongle** (ou de um adaptador BT→USB, ou do cabo
USB se o teclado tiver). Aí ele é USB HID e o port funciona. Pareamento direto no System
Settings não existe no Switch 1; no modo applet o port não consegue criar esse suporte — um
sysmodule de BT HID seria outro projeto, com risco de firmware e de conflito com o `btdrv`.

### 5.3 Vários dispositivos, bateria e wake

- **Vários mouses**: o shared memory tem um LIFO só; o sistema funde. Não há como separar,
  escolher o “mouse 1” ou mostrar bateria por device. Dois mouses podem “brigar” pelo ponteiro.
- **Vários teclados**: um único snapshot; a fusão é provável (OR), mas não documentada.
- **Bateria**: não há campo no `HidMouseState`; as APIs de bateria do libnx são para Npad.
- **Wake/sleep**: o applet é suspenso; o USB pode reenumerar. Não há API para acordar o console
  pelo mouse. O app deve tratar `IsConnected` e re-sincronizar ao voltar o foco.
- **Switch 2**: o firmware 20+ tem `Nmouse` (`InitializeNmouse`, `SetNmousePointerPosition`,
  `EnableShowingCursor`, etc.) e o Joy-Con 2 como mouse
  ([HID services](https://switchbrew.org/wiki/HID_services)). Não é o alvo deste port (Switch 1,
  Erista, firmware 22.5).

---

## 6. Arquitetura proposta

### 6.1 Visão geral

```text
[hid system module]
   │  shared memory: mouse LIFO[17], keyboard LIFO[17]
   ▼
[HidPoller thread]  a cada 2 ms (500 Hz)
   │  drena 17, filtra sampling_number, timestamp, estatísticas
   ▼
[sync_channel ring, 512 amostras]  (mouse)  +  [ring 128] (keyboard)
   ▼
[winit EventLoop::run → NativeInputs::poll]  uma vez por frame
   │
   ├─ delta: soma(dx,dy) → DeviceEvent::MouseMotion
   │    ├─ câmera: mouse.on_move (vanilla)
   │    └─ menu: move_menu_cursor (dpi * cursorSpeed)
   ├─ CursorMoved absoluto só no resync/semente + touch/pad
   ├─ botões: diff → MouseInput
   └─ wheel: normaliza → MouseWheel
   ▼
[Game]  câmera vanilla + draw_mouse_cursor (sprite 24x36)
```

### 6.2 O poller dedicado

Novo arquivo: `switch/patches/winit-0.30.13/src/platform_impl/nx/hid_poller.rs`.

```rust
pub(crate) struct MouseSample {
    pub sampling: u64,
    pub x: i32, pub y: i32,
    pub dx: i32, pub dy: i32,
    pub wheel_x: i32, pub wheel_y: i32,
    pub buttons: u32,
    pub attributes: u32,
    pub at: Instant,          // quando o poller observou o relatório
}

pub(crate) struct KeyboardSample {
    pub sampling: u64,
    pub modifiers: u64,
    pub keys: [u64; 4],
    pub at: Instant,
}

pub(crate) struct HidPoller {
    mouse_rx: Receiver<MouseSample>,
    keyboard_rx: Receiver<KeyboardSample>,
    stats: Arc<Stats>,
    stop: Arc<AtomicBool>,
    thread: Option<JoinHandle<()>>,
}
```

O loop:

```rust
fn run(stop: Arc<AtomicBool>, tx_mouse: SyncSender<MouseSample>,
       tx_keyboard: SyncSender<KeyboardSample>, stats: Arc<Stats>) {
    // Sincroniza com o relatório mais novo antes de começar; 0 replayaria o LIFO.
    let mut last_mouse = newest_mouse_sampling();
    let mut last_keyboard = newest_keyboard_sampling();
    while !stop.load(Ordering::Relaxed) {
        drain_mouse(&mut last_mouse, &tx_mouse, &stats);
        drain_keyboard(&mut last_keyboard, &tx_keyboard, &stats);
        stats.log_if_due();
        // 2 ms. Se o sleep do newlib arredondar para 4 ms, troque por
        // svcSleepThread(2_000_000) (declare em ffi).
        thread::sleep(Duration::from_millis(2));
    }
}
```

`drain_mouse`:

```rust
let mut states = [HidMouseState::default(); 17];
let n = unsafe { hidGetMouseStates(states.as_mut_ptr(), 17) }.min(17);
for s in states[..n].iter().rev() {          // mais antigo -> mais novo
    if s.sampling_number <= last_mouse { continue; }
    if s.sampling_number > last_mouse + 1 {
        stats.gaps.fetch_add(s.sampling_number - last_mouse - 1, Relaxed);
    }
    last_mouse = s.sampling_number;
    stats.reports.fetch_add(1, Relaxed);
    stats.connected.store(s.attributes & HID_MOUSE_ATTRIBUTE_CONNECTED != 0, Relaxed);
    let sample = MouseSample { /* ... */ at: Instant::now() };
    if tx_mouse.try_send(sample).is_err() {
        stats.dropped.fetch_add(1, Relaxed);
    }
}
```

Detalhes:

- **Capacidade**: 512 amostras de mouse (1 s a 500 Hz, 0,5 s a 1000 Hz) e 128 de teclado.
  `try_send`; se cheio, descarta a **nova** e incrementa `dropped`; no próximo frame, se
  `dropped > 0`, o consumidor faz resync (descarta o backlog e pula para o mais novo) para não
  despejar movimento velho de uma vez.
- **Primeiro drain**: no start, `last_mouse`/`last_keyboard` recebem o `sampling_number` do
  relatório mais novo (uma leitura antes do loop). Sem isso, o primeiro drain replaya até 17
  relatórios antigos.
- **Timestamps**: `Instant::now()` por amostra observada; não é o timestamp do hardware (o
  `HidMouseState` não tem um). A ordem é garantida por `sampling_number`; o timestamp serve
  para log, idle fade e dt.
- **Custo**: cada `hidGet...` é uma cópia do shared memory, sem IPC com o serviço `hid`; 500
  leituras/s com ~1,5 KB cada é barato. A prioridade normal basta; prioridade alta pode roubar
  CPU do renderizador. Se o log mostrar gaps por starvation, aí sim suba um pouco.
- **Falha de spawn**: `HidPoller::start` devolve `None`; `NativeInputs` cai no modo atual
  (poll por frame) e loga um aviso.
- **Shutdown**: `Drop` seta `stop` e dá `join` (o sleep de 2 ms garante saída rápida). O
  processo não chama `hidExit`; não há risco de usar o shared memory depois do fim.
- **Sem foco**: o poller pode dormir 16 ms e drenar/descartar; ao voltar o foco, o `resync` pega
  o relatório mais novo. Economiza wakeups quando o jogo está em segundo plano.

### 6.3 O consumidor no event loop

`NativeInputs` deixa de ser o leitor e passa a ser o **decodificador**:

```rust
pub(crate) fn poll(&mut self, keyboard_id, mouse_id, grabbed, out: &mut PendingEvents) {
    let mut motion = (0i64, 0i64);
    for sample in self.poller.drain_mouse() {
        if sample.sampling <= self.last_mouse { continue; }   // dupla proteção
        self.last_mouse = sample.sampling;
        if sample.attributes & CONNECTED == 0 {
            self.release_buttons(out);
            self.connected = false;
            continue;
        }
        if !self.connected { self.resync = true; self.connected = true; }
        let sync = self.resync;
        if sync {
            // Semeia a posição absoluta uma vez; o jogo ignora as próximas.
            if !grabbed {
                self.seed_cursor(mouse_id, sample.x, sample.y, screen_w, screen_h, out);
            }
            self.resync = false;
        } else {
            motion.0 += sample.dx as i64;
            motion.1 += sample.dy as i64;
        }
        // Só “corta” o movimento acumulado quando um evento precisa da ordem.
        if sample.buttons != self.last_buttons {
            self.flush_motion(mouse_id, &mut motion, out);
            self.emit_buttons(sample.buttons, out);
            self.last_buttons = sample.buttons;
        }
        if sample.wheel_x != 0 || sample.wheel_y != 0 {
            self.flush_motion(mouse_id, &mut motion, out);
            self.emit_wheel(sample.wheel_x, sample.wheel_y, out);
        }
    }
    self.flush_motion(mouse_id, &mut motion, out);
    // teclado: drena KeyboardSample e chama keyboard_report em ordem
}
```

`flush_motion` emite **um** `DeviceEvent::MouseMotion` com a soma (e nada se a soma for zero);
o jogo já decide o que fazer:

- `self.captured && self.screen == Playing` → `mouse.on_move(...)`, a câmera vanilla.
- `cfg!(target_os = "horizon") && !self.captured && (title ou screen != Playing)` →
  `move_menu_cursor(dx, dy)`, que aplica `dpi = size[0]/1280`, `cursorSpeed` e
  `step = dpi * speed / batch.scale`.

O backend **não** acumula posição de cursor. `CursorMoved` só é emitido no resync/semente
(posição absoluta) — e pelo touch/pad, que têm o seu próprio caminho. O jogo já tem
`mouse_have_pos`: no Switch, depois do primeiro relatório absoluto, ele ignora `CursorMoved` do
mouse e usa só os deltas. Isso é o que impede a aceleração do ponteiro do sistema de brigar com
o `cursorSpeed`.

**Ordem no resync:** emita um `DeviceEvent::MouseMotion` (mesmo com delta 0) **antes** do
`CursorMoved` de semente. O jogo só marca `mouse_device` no `device_event`; se o `CursorMoved`
chegar primeiro, ele não é reconhecido como mouse e o `mouse_have_pos` não é setado. A
alternativa é fazer o próprio `WindowEvent::CursorMoved` setar `mouse_device` quando o
`device_id` é o do mouse.

Regras de ouro:

- **Câmera e cursor usam o mesmo delta, mas o backend não escala nenhum dos dois.** A câmera
  recebe a soma crua; o `move_menu_cursor` do jogo aplica `dpi * cursorSpeed / batch.scale`.
  Nunca derive a câmera da posição do cursor (o clamp da borda mataria o giro) nem o cursor da
  posição absoluta a cada frame (o ponteiro do sistema é 720p e pode ter aceleração própria).
- **`flush_motion` antes de botão/wheel**: mantém a ordem “mexeu → clicou”. A soma de vários
  relatórios vira **um** `DeviceEvent::MouseMotion` por trecho entre botões/wheel (e um no fim
  do frame); isso reduz a fila sem perder delta.
- **`sampling_number` duas vezes**: o poller filtra o que já enfileirou; o consumidor filtra o
  que já emitiu. Nem duplicata nem buraco.
- **Resync** em: primeiro poll, perda/ganho de foco, `IsConnected` de falso→verdadeiro,
  `dropped > 0`, `sampling_number` andou para trás. Resync = descartar o backlog, semear a
  posição absoluta do relatório mais novo e não emitir delta.
- **Grab**: o delta continua indo para a câmera; o jogo não chama `move_menu_cursor` quando
  `captured`. No `capture(false)`, o jogo centraliza `self.cursor`; o próximo `CursorMoved` de
  semente/pad/touch mantém a posição coerente.
- **Fallback**: se `poller` for `None`, faz o `hidGetMouseStates(17)` direto como hoje. O
  caminho de fallback tem o mesmo filtro; não misture os dois no mesmo frame.

### 6.4 Por que não usar só o LIFO do libnx

O LIFO já é um ring de 17. A thread acrescenta três coisas que ele não dá:

1. **Histórico maior que 17 relatórios** para o frame que trava (ring de 512).
2. **Leitura a 2 ms** em vez de 16,7 ms, o que mantém o LIFO jovem e reduz a latência de
   amostragem.
3. **Medição e timestamp** independentes do frame.

Se o frame roda a 60 fps e nunca trava, o patch atual já drena tudo a 125 Hz e quase tudo a
1000 Hz. A thread é para o caso real (geração de mundo, GC, hitch) e para medir.

### 6.5 Alternativas descartadas

- **Ler USB direto com `usbhs`/libusbhsfs**: pegaria o mouse do sistema, exigiria um driver HID
  próprio e quebraria o ponteiro dos menus; o applet pode não ter permissão. Não vale.
- **Patchear o `usb` module para 1000 Hz**: é o que o `gcadapter-oc-nx` faz para o adaptador
  GameCube, com riscos de reenumeração e de o controle sumir. Fora do escopo de um applet.
- **Loop de 1000 Hz**: ganho de 1 ms, custo de 1000 wakeups/s e LIFO de 17 ms. Não compensa.
- **Cursor do sistema**: nos ports consultados não há um cursor do sistema desenhado no applet;
  `set_cursor_visible` é no-op. O cursor tem de ser desenhado pelo jogo.

---

## 7. Cursor: o que já existe e o que falta

### 7.1 O que já existe (não é para reescrever)

O cursor do mouse **já está implementado** no jogo, e não no backend. Os pontos:

- `crates/horizon-client/src/hud.rs::cursor_image()` rasteriza um sprite 48×72 de uma seta
  branca com contorno escuro, com **4×4 supersampling** (anti-aliasing). O sprite entra no atlas
  como `horizon/cursor`; `CURSOR_SIZE = (24, 36)` é o tamanho físico de desenho. O hotspot é o
  canto superior esquerdo.
- `crates/horizon-client/src/game.rs::draw_mouse_cursor` desenha o cursor em todos os menus:
  só no Switch, só com `!self.captured`, só se `self.mouse_seen`, e só se o pad não estiver
  mostrando o ponteiro dele (`self.pad.menu.pointer_shown()`). Em gameplay puro
  (`title.is_none() && screen == Screen::Playing`) ele não aparece.
- `paint_mouse_cursor` desenha em **tamanho físico fixo**: `CURSOR_SIZE.0 / batch.scale` por
  `CURSOR_SIZE.1 / batch.scale`, com `rect_uv` e alpha; o clamp mantém o cursor dentro da tela.
- `move_menu_cursor` move `self.cursor` a partir do delta cru do `DeviceEvent::MouseMotion`:
  `dpi = size[0] / 1280`, `speed = horizon.cursor_speed / 100`,
  `step = dpi * speed / batch.scale`, depois `clamp(0, batch.width/height)`. É a separação
  “delta para o cursor, absoluto só para semear” já funcionando.
- `WindowEvent::CursorMoved` de um mouse real só **semeia** `self.cursor` no primeiro relatório
  (`mouse_have_pos`); depois disso o jogo ignora a posição absoluta do mouse e usa os deltas.
  Touch e pad continuam entrando por `CursorMoved` absoluto.
- `cursorSpeed` já existe em `horizon-graphics.txt` (50–200%, default 100), com controle no
  launcher/`launcher_v2.rs` e uso no `startup_menu.rs`.
- O fade também já existe: 2 s visível, 0,5 s de fade, calculado em `draw_mouse_cursor` e
  repetido no caminho do título/loading (`game.rs` ~13700).

**Nos ports consultados, o applet não recebe um ponteiro do sistema**: o “ponteiro gigante” que
se vê é o cursor do pad (`draw_pad_cursor`, 10×10 GUI px) ou o da TV/OS fora do applet. O cursor
do mouse é o sprite 24×36 acima; se ele parecer grande ou pequeno, o problema é de **escala**,
não de ausência.

### 7.2 O que falta (o escopo real do P2)

| Falta | Onde | Correção |
|---|---|---|
| `cursorScale` | `paint_mouse_cursor` | `w = CURSOR_SIZE.0 * scale / batch.scale`; faixa 0.5–2.0; default 1.0 |
| `cursorIdle` | `draw_mouse_cursor` e o caminho do título | trocar o `2.0`/`0.5` fixos por `settings.horizon.cursor_idle` (0–10 s; 0 = sempre) |
| `cursorSpeed` na página Options → Mouse | `options_ui.rs` | novo `Opt::CursorSpeed` lendo/gravando `settings.horizon.cursor_speed` (50–200%) |
| Checar foco | `draw_mouse_cursor` | adicionar `!self.focused` (hoje o cursor pode ficar 2 s após perder o foco) |
| Código de fade duplicado | `game.rs` ~13700 (título) | extrair `mouse_cursor_opacity(&self) -> f32` e usar nos dois lugares |
| Escala do sprite em telas grandes | `cursor_image`/`CURSOR_SIZE` | o sprite 48×72 é 2× do tamanho 24×36; em `cursorScale = 2.0` ele é desenhado 1:1, ainda nítido; acima disso ficaria borrado, por isso a faixa para em 2.0 |
| Cursor em gameplay não-grab | `draw_mouse_cursor` | hoje ele some se `screen == Playing`; se o jogador abrir o chat/inventário, `screen != Playing` e ele volta; comportamento correto |

O que **não** fazer: criar um `MouseCursor` novo, um `game/cursor.rs` ou um `horizon-mouse.txt`
só para o cursor. O código e o arquivo já existem; duplicar criaria dois ponteiros e duas
velocidades.

### 7.3 Desenho e escala

`paint_mouse_cursor` já é textured e anti-aliased; não troque por uma seta de `fill` pixel-art
sem necessidade. A escala:

```rust
let scale = settings.horizon.cursor_scale;       // 0.5 ..= 2.0
let w = CURSOR_SIZE.0 * scale / batch.scale;
let h = CURSOR_SIZE.1 * scale / batch.scale;
let x = cursor.0.clamp(0.0, (batch.width - w).max(0.0));
let y = cursor.1.clamp(0.0, (batch.height - h).max(0.0));
batch.rect_uv(x, y, w, h, rect, rgba(255, 255, 255, alpha));
```

- Em `scale = 1.0`, o sprite 48×72 é reduzido para 24×36 (2× downscale, ótimo).
- Em `scale = 2.0`, é desenhado 48×72 (1:1, ainda nítido).
- Em `scale = 0.5`, é 12×18 (4× downscale; o supersampling segura).
- Não faça `floor()` no `x/y` do sprite: como ele é textured e reduzido, o snap duro pode
  tremer; o `rect_uv` já é preciso o suficiente. Se quiser pixel-perfect, arredonde a posição
  final em pixels físicos, não em GUI.

### 7.4 Visibilidade

- **Grab**: `self.captured` esconde; o backend já não emite `CursorMoved` nesse estado.
- **Idle fade**: qualquer `MouseMotion`, `CursorMoved`, `MouseInput` ou `MouseWheel` atualiza
  `mouse_last`; o alpha é 1 até `cursorIdle` e cai em 0,5 s. `cursorIdle = 0` = sempre visível.
- **Handover com o pad**: `pad.menu.pointer_shown()` esconde o cursor do mouse; quando o mouse
  se mexe, `pad_menu_frame` vê a diferença em `self.cursor`, limpa `pad.menu.pos`/`focus`, e o
  cursor do mouse volta.
- **Touch**: `poll_touch` emite `CursorMoved` com `DeviceId::TOUCH`; o cursor segue qualquer
  `CursorMoved`, não só o do device `MOUSE`. Em handheld, dedo e mouse compartilham o cursor.
- **Foco perdido**: adicione `!self.focused` ao gate; ao voltar, o primeiro `CursorMoved` do
  resync mostra de novo.
- **Título/loading**: o caminho ~13700 desenha o mesmo sprite; use o mesmo `opacity`/escala para
  não ter dois comportamentos.

### 7.5 Onde desenhar

Já está em `game.rs::hud()`, depois de `draw_pad_cursor`:

```rust
spans::time("hud_draw_mouse_cursor", || self.draw_mouse_cursor(&mut batch));
```

Não mova para o backend: o cursor é UI do jogo, com o atlas e o GUI scale do jogo.

---

## 8. Configurações e sensibilidade

### 8.1 Onde guardar (reaproveitando o que existe)

- **`options.txt` continua vanilla**: `sensitivity`, `mouseWheelSensitivity`, `invertMouseY`,
  `discreteMouseScroll` já existem e são lidos por `Settings`. Não invente chaves Horizon ali.
- **`cursorSpeed` já está em `horizon-graphics.txt`** (50–200%, default 100), com
  `Graphics::cursor_speed` em `crates/horizon-client/src/graphics.rs`. O caminho certo é
  **acrescentar** `cursorScale` e `cursorIdle` ao mesmo `Graphics`/arquivo, não criar um
  `horizon-mouse.txt` paralelo. O `horizon-graphics.txt` já é o arquivo Horizon de ajustes de
  vídeo/UI e o launcher já o edita.
- **Launcher**: o launcher de desktop tem `game_home()` e já edita `horizon-graphics.txt` para
  o `graphicsApi`; pode fazer o mesmo com `cursorSpeed`/`cursorScale`/`cursorIdle` e a linha
  `sensitivity` do `options.txt`. **No Switch não há launcher separado**: o controle tem de
  estar em Options → Mouse (e o launcher in-game já tem o ciclo de `cursorSpeed`).

### 8.2 Chaves, defaults e faixas

| Chave | Padrão | Faixa | O que faz |
|---|---:|---:|---|
| `cursorSpeed` | 100 | 50–200% | Já existe: multiplicador do cursor em `move_menu_cursor`; 100% = 1280 unidades virtuais cruzam a tela uma vez |
| `cursorScale` | 1.0 | 0.5–2.0 | Novo: tamanho do sprite (24×36 × escala, em px físicos) |
| `cursorIdle` | 2.0 | 0–10 s | Novo: tempo até o fade; 0 = sempre visível |
| `cursorAcceleration` | 0.0 | 0–1 | Opcional/P3: curva de aceleração **do cursor**; 0 = linear (não existe ainda) |

A sensibilidade da câmera continua sendo o `sensitivity` vanilla (0–1, default 0.5), com a
curva cúbica de `mouse.rs::sensitivity_scales`: `ss = 0.6*s + 0.2`, `mod = ss^3`, `sens = mod*8`.
Não reimplemente essa curva no backend; o `MouseMotion` é a entrada dela. Também não duplique
`invertMouseY`: o vanilla já cobre a inversão de Y.

**Por que não uma faixa maior?** A faixa atual de `cursorSpeed` (50–200%) é 4× e cobre
720p/1080p. Se
quisermos mais, mude o clamp em `Graphics::parse` (hoje `clamp(50, 200)`) e os passos do
launcher; não crie uma segunda chave. Para o cursor, `cursorScale` já cobre o tamanho.

### 8.3 Curva de aceleração (opcional, P3)

Default **off**. Se um dia entrar, que seja só no cursor (`move_menu_cursor`):

```text
speed = |d| / dt                      // unidades/s no frame
factor = 1 + accel * clamp(speed / 4000, 0, 1) * 0.75
out = d * factor
```

- Só no cursor; a câmera fica linear (paridade vanilla).
- `accel = 0` é linear; `accel = 1` dá no máximo 1,75× em movimentos rápidos.
- Aceleração prejudica memória muscular; deixe claro no tooltip. Se o sistema já acelerar
  `delta`, ligar isso piora; teste antes.
- **Não implemente isso antes de medir** se o `delta` do Switch já vem acelerado; se vier, a
  curva só piora.

### 8.4 UI

**Options → Mouse** (`options_ui.rs`, `Page::Mouse`) já existe; adicione:

- `Opt::CursorSpeed` — lê/grava `s.horizon.cursor_speed` (50–200); `Kind::Slider((speed - 50) /
  150)`, label `horizon.options.cursor_speed`, valor `100%`.
- `Opt::CursorScale` — lê/grava `s.horizon.cursor_scale` (0.5–2.0); `Kind::Slider((scale - 0.5)
  / 1.5)`, label `horizon.options.cursor_scale`.
- `Opt::CursorIdle` — lê/grava `s.horizon.cursor_idle` (0–10 s); `Kind::Slider(idle / 10.0)`,
  label `horizon.options.cursor_idle`; `0` mostra “Off”.
- A sensibilidade vanilla (`Opt::Sensitivity`) já está na mesma página.

Adicione as chaves em `crates/horizon-client/assets/horizon/lang/en_us.json` e `ru_ru.json`
(e, se houver, nos demais idiomas). O `Opt::kind`/`Opt::set`/`Opt::label` ficam em
`options_ui.rs`. O `Settings` **não** ganha um `MouseSettings` novo: use `settings.horizon`
(Graphics), que já é salvo em `horizon-graphics.txt`.

**Launcher in-game**: `launcher_v2.rs::cycle_cursor_speed` já existe; acrescente `cursorScale` e
`cursorIdle` ao mesmo grupo de opções (ou à página Graphics), com os mesmos limites.

**Launcher de desktop**: uma seção “Mouse” em `settings_page` com um stepper (o launcher não tem
widget de slider) para `cursorSpeed` e `sensitivity`; grava
`game_home()/horizon-graphics.txt` e a linha `sensitivity` do `options.txt`, preservando o resto.
No Switch, a página in-game é a única; o launcher não é usado no console.

---

## 9. Plano por fases

### P0 — o que já está no tree e o que falta

**Já existe no fonte local** (verificado no tree de trabalho, não reimplemente):

- Log `Horizon HID: <n> mouse reports/s` (1 s com `HORIZON_HID_LOG=1`, 5 s sem), em
  `native_inputs.rs::poll`.
- Reset de `self.mouse`/`self.buttons` quando `sampling_number` do mouse anda para trás
  (reconexão).
- Síntese de desconexão quando `mouse_count == 0` e `self.connected` (libera botões presos).
- `wheel_delta_y` com fallback para `wheel_delta_x`.
- `CursorMoved` antes de `MouseInput` quando a posição mudou.
- `cursor()` já escala de 1280×720 para 1920×1080 em docked.

**Falta (P0 pequeno):**

- `native_inputs.rs::poll`: no log de taxa, incluir `gaps` (soma de
  `sampling_number - last - 1`) e `connected`; hoje só mostra relatórios/s.
- `native_inputs.rs::poll`: aplicar o mesmo reset de `sampling_number` ao **teclado** (hoje só
  o mouse tem o guard).
- `native_inputs.rs::mouse_report`: logar o wheel cru uma vez; se vier em múltiplos de 120 por
  notch, normalizar (`/120` ou `signum`) em vez de mandar 120 linhas.
- `native_inputs.rs::cursor`: logar o maior `x`/`y` visto (uma vez por modo) para confirmar
  1280×720 em docked; se aparecer 1920, ajustar o clamp/semente.
- `draw_mouse_cursor`: adicionar `!self.focused` (1 linha; pode ir no P2).

Aceite:

- Log mostra `Horizon HID: ... reports/s, gaps=N, connected=true` a 125/500/1000.
- Desconectar o mouse com botão apertado emite release; reconectar não deixa botão preso.
- Wheel que só reporta `wheel_delta_x` funciona e não rola 120 linhas.
- Em docked, o cursor alcança o canto inferior direito.
- Nenhuma regressão no desktop (módulo só compila em `target_os = "horizon"`).

### P1 — poller dedicado + ring (1–2 PRs)

Objetivo: 500 Hz de leitura, sem perder delta em hitch, com taxa medida.

Arquivos/funções:

- **Novo** `switch/patches/winit-0.30.13/src/platform_impl/nx/hid_poller.rs`:
  - `pub(crate) struct HidPoller`, `MouseSample`, `KeyboardSample`, `Stats`.
  - `HidPoller::start() -> Option<Self>`; `fn run(...)`; `fn drain_mouse(...)`;
    `fn drain_keyboard(...)`; `fn log_if_due(...)`; `impl Drop`.
  - `fn decode_mouse(states: &[HidMouseState], last: &mut u64) -> Vec<MouseSample>` pura, para
    testar no host sem libnx.
- `native_inputs.rs`:
  - `NativeInputs::new(poller: Option<HidPoller>)`.
  - `poll()`: drena os rings; `flush_motion`; `buttons`; `wheel`; `keyboard_report` em ordem;
    fallback direto se `poller == None`.
  - `resync` em foco/connect/drop/gap; `last_mouse`/`last_keyboard`.
- `event_loop.rs`:
  - `EventLoop` ganha `hid: Option<HidPoller>`; `new` inicia depois de `hidInitializeMouse`;
    `run` desestrutura `self` e passa o poller para `NativeInputs`; `Drop` para a thread.
  - Opcional `Shared.input_dirty` para acordar `wait()` na hora; sem isso, o `wait` já acorda a
    cada 4 ms.
- `mod.rs`: `mod hid_poller;`.

Aceite:

- 10 s de movimento contínuo: `gaps = 0`, `dropped = 0`, taxa estável.
- Hitch artificial de 200 ms: a soma dos deltas antes/depois confere; nenhum movimento perdido.
- Perda/ganho de foco: sem movimento fantasma, sem botão preso.
- Unplug/replug: sem botão preso; cursor re-sincroniza no primeiro relatório.
- CPU < 1% de um núcleo a 500 Hz (medir com o sys-clk/overlay).

### P2 — cursor: escala, fade e exposição (1 PR)

Objetivo: o cursor já existe; falta deixá-lo ajustável e acessível. **Não** criar um
`MouseCursor`/`game/cursor.rs`/`horizon-mouse.txt` novo.

Arquivos/funções:

- `crates/horizon-client/src/graphics.rs` (o `Graphics` de `horizon-graphics.txt`):
  - `pub cursor_scale: f32` (default 1.0; parse `clamp(0.5, 2.0)`), `pub cursor_idle: f32`
    (default 2.0; `clamp(0.0, 10.0)`), ao lado do `cursor_speed` que já existe.
  - Incluir no `Default`, no `parse`, no `serialize` e nos testes.
- `crates/horizon-client/src/game.rs::paint_mouse_cursor`:
  - ganha um parâmetro `scale: f32` (quem chama passa `settings.horizon.cursor_scale`);
  - `w = CURSOR_SIZE.0 * scale / batch.scale`, `h = CURSOR_SIZE.1 * scale / batch.scale`.
- `crates/horizon-client/src/game.rs::draw_mouse_cursor`:
  - trocar o `2.0`/`0.5` fixos por `settings.horizon.cursor_idle` e um fade de 0,5 s;
  - adicionar `!self.focused` ao gate;
  - extrair `fn mouse_cursor_opacity(&self) -> f32` e usar também no caminho do título
    (~13700), que hoje repete a conta.
- `crates/horizon-client/src/options_ui.rs`:
  - `Opt::CursorSpeed` (lê/grava `s.horizon.cursor_speed`), `Opt::CursorScale` (0.5–2.0),
    `Opt::CursorIdle` (0–10 s); `kind` (~303), `label` (~205), `set` (~638);
    `Page::Mouse` (~1523).
- `crates/horizon-client/src/launcher_v2.rs` / `startup_menu.rs`:
  - `cycle_cursor_speed` já existe; acrescentar `cursorScale`/`cursorIdle` ao mesmo grupo, se
    quiser espelhar no launcher in-game.
- `crates/horizon-client/assets/horizon/lang/en_us.json` e `ru_ru.json`: chaves
  `horizon.options.cursor_speed`, `horizon.options.cursor_scale`, `horizon.options.cursor_idle`.

Aceite:

- Sprite nítido em 720p e 1080p; `cursorScale` 0.5–2.0 muda o tamanho físico.
- Fade após `cursorIdle` (default 2 s); `0` = sempre visível; reaparece no movimento/clique.
- Some no grab e no gameplay; aparece nos menus/título; handover com pad/touch sem dois cursores.
- Settings persistem em `horizon-graphics.txt`; `cursorSpeed` 50–200%; câmera 0–1 (vanilla).
- No desktop, o cursor do sistema continua normal; o `draw_mouse_cursor` já é
  `cfg!(target_os = "horizon")`.

### P3 — opcional

- Launcher de desktop: seção “Mouse” + stepper; testes de arquivo como `switch_to_dx12`.
- Overlay F3 com `mouse Hz`, `gaps`, `dropped` (útil para testers).
- `horizon-ctl input-status` devolvendo JSON com a taxa.
- Experimento de 1000 Hz (medir CPU/latência; provavelmente não vale).
- Layout ABNT2: tabela de usages → caracteres; ler a configuração do sistema se o libnx expuser
  no futuro.
- Switch 2: `Nmouse`/Joy-Con 2; fora do escopo.

---

## 10. Riscos e o que é especulação

### 10.1 Confirmado vs. provável vs. desconhecido

| Nível | Afirmação |
|---|---|
| **Confirmado** | `HidMouseState`/`HidKeyboardState` têm os campos acima; 17 entradas; mais novo em `[0]`; `sampling_number`; `IsConnected` no bit 1; teclado sem `attributes`; uma seção de mouse e uma de teclado; `hidInitialize*` aborta em erro; `x/y/delta` em 1280×720; nos ports consultados o applet não recebe ponteiro do sistema; o Switch 1 não pareia BT HID genérico; controles oficiais/GCN a 125 Hz; o `usb` module programa o bInterval. |
| **Provável** | Mouses genéricos de 125 Hz (bInterval 8 ms) entregam 125 Hz; mouses de 1000 Hz podem entregar 1000 Hz se o `usb` honrar 1 ms; vários mouses são fundidos; o sistema pode aplicar aceleração em `x/y` e não em `delta`. |
| **Desconhecido** | Taxa efetiva de um mouse qualquer no `hidGetMouseStates`; se o `hid` repassa ou amostra; política de fusão; comportamento em sleep/wake; reset de `sampling_number`; se `x/y` chegam a 1920×1080 em algum firmware; dead zone na borda. |
| **Especulação** | 500 Hz é suficiente para 60 fps; 1000 Hz é overkill; o cursor pixel-art vai agradar mais que um AA; a thread reduz perda em hitch. |

### 10.2 Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Taxa efetiva < 500 Hz | alta | sem 500 Hz real | Logar; aceitar 125 Hz; não prometer |
| `x/y` 720p em docked | confirmado em outros ports | cursor preso no box | Acumular delta; escalar por `screen_h/720` |
| Aceleração do sistema em `delta` | desconhecida | câmera não linear | Testar; se não linear, curva opcional ou aceitar |
| BT HID nativo | confirmado que não | requisito não atendido | Documentar; usar dongle USB |
| Vários mouses fundidos | provável | sem seleção/bateria | Documentar; um mouse por vez |
| `sampling_number` reinicia | desconhecida | filtro `>` para de emitir | Guardar salto para trás e resync |
| Thread starvation | baixa | gaps | Ring maior; prioridade; logar |
| Ring cheio | baixa | perda de movimento | `dropped` + resync; capacidade 512 |
| Licença GPL | média | contaminação | Só MIT/ISC; GPL como ideia |
| Docked/undocked | média | cursor pula | Reescalar no `Resized`; testar |
| CPU/heat | baixa | frame drop | 500 Hz; medir; não 1000 Hz |

### 10.3 O que ainda precisa de hardware

1. Rodar o P0 e ver a taxa real de um mouse 125 Hz, um 500 Hz e um 1000 Hz.
2. Medir `x`/`y` máximos em docked; confirmar 1280×720.
3. Mover o mouse até a borda e reverter: há dead zone?
4. Fazer movimentos lentos e rápidos e comparar `sum(dx)` com a distância física: há aceleração?
5. Desconectar/reconectar mouse e teclado com botões/teclas presos.
6. Dormir/acordar (HOME/sleep) e ver `sampling_number`/`IsConnected`.
7. Dois mouses ao mesmo tempo; dois teclados.
8. Dongle 2.4 GHz e adaptador BT→USB.
9. Hitch de 200 ms (gerar mundo) e conferir o ring.
10. CPU/overlay a 500 Hz.

---

## 11. Checklist de teste (hardware)

- [ ] Build do NSP com o patch local; `sdmc:/switch/horizon/logs/latest.log` aberto.
- [ ] Mouse USB no dock; cursor aparece, anda, clica, scroll.
- [ ] Mouse USB via adaptador USB-C em handheld; idem.
- [ ] Teclado USB no dock; WASD, Shift, Ctrl, Alt, F3, Esc, Enter.
- [ ] Teclado ABNT2: documentar o que sai errado (layout US).
- [ ] Desconectar mouse com botão apertado → release.
- [ ] Desconectar teclado com tecla apertada → próxima tecla/sair da tela limpa.
- [ ] Dock/undock com o cursor no canto → posição relativa correta.
- [ ] Grab (mundo) → cursor some, câmera responde; menu → cursor volta.
- [ ] Pad no modo pointer → cursor do pad; mover o mouse → handover.
- [ ] Touch em handheld → cursor segue o dedo.
- [ ] `cursorSpeed` 50 / 100 / 200%; `cursorScale` 0.5 / 1.0 / 2.0.
- [ ] Fade em `cursorIdle` (2 s default); `cursorIdle = 0` sempre visível.
- [ ] Reiniciar o jogo; settings persistem.
- [ ] Log: taxa Hz, gaps, dropped; anotar no doc.

---

## 12. Referências

- libnx `hid.h` — https://github.com/switchbrew/libnx/blob/master/nx/include/switch/services/hid.h
- libnx `hid.c` — https://github.com/switchbrew/libnx/blob/master/nx/source/services/hid.c
- libnx Changelog — https://github.com/switchbrew/libnx/blob/master/Changelog.md
- Switchbrew HID services — https://switchbrew.org/wiki/HID_services
- Switchbrew 6.0.0 (layout de teclado USB) — https://switchbrew.org/wiki/6.0.0
- libnx `HidMouseState` docs — https://switchbrew.github.io/libnx/structHidMouseState.html
- Moonlight-NX #62 (box 720p) — https://github.com/rock88/moonlight-nx/issues/62
- Moonlight-NX `SwitchMouseFrontend.cpp` — https://github.com/rock88/moonlight-nx/blob/master/src/controls/switch/SwitchMouseFrontend.cpp
- zomboid-switch PR #12 — https://github.com/leo-snczki/zomboid-switch/pull/12
- key2con — https://github.com/EnzoBernardinoMartins/key2con
- wmw_nx `nx_pointer.c` — https://github.com/ChanseyIsTheBest/wmw_nx/blob/main/source/nx_pointer.c
- papersplease_nx — https://github.com/ChanseyIsTheBest/papersplease_nx
- Moonlight-Switch — https://github.com/XITRIX/Moonlight-Switch
- RetroArch `switch_input.c` — https://github.com/libretro/RetroArch/blob/master/input/drivers/switch_input.c
- Sphaira — https://github.com/NaGaa95/sphaira
- sys-hidplus — https://github.com/luvvyamy/sys-hidplus
- gcadapter-oc-nx — https://github.com/KakarottoCake/gcadapter-oc-nx
- Titan Two, Switch polling — https://www.consoletuner.com/wiki/doku.php?id=t2%3Ausage_guides%3Asystems%3Aswitch
- Toad King, Pro Controller wired 125 Hz — https://gbatemp.net/threads/reverse-engineering-the-switch-pro-controller-wired-mode.475226/
- HandHeldLegend GC adapter (8 ms) — https://github.com/HandHeldLegend/GC-Adapter-ESP32-S3
- Nintendo Korea, BT keyboard não suportado — https://www.nintendo.co.kr/support/switch/qna/116
- Nintendo, Bluetooth Audio (A2DP/SBC) — https://www.nintendo.com/my/support/switch/accessories/bluetooth_audio.html
- DrasticDS_nx `input_sampler.c` — https://github.com/NaGaa95/DrasticDS_nx/blob/master/source/input_sampler.c

*Consultado em 2026-11-20. O texto do libnx é a conversão oficial do repositório; o comportamento
exato de firmware pode variar. Medições de hardware deste port ainda pendentes.*
