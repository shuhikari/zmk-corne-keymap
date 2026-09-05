# zmk-corne-keymap — Corne XIAO (shield `corny`)

Config ZMK do meu Corne split sem fio: **Seeed XIAO nRF52840** + shield `corny`
(PCB [corne-xiao](https://github.com/shuhikari/corne-xiao) rev2, 6 colunas).

- 44 posições por camada: 42 teclas + 2 centrais (`SW22`, botão do encoder).
- LED RGB embutido do XIAO indicando **bateria** e **conexão BLE**.
- **ZMK Studio** habilitado: dá pra remapear tecla ao vivo por USB, sem recompilar.

## Como a estrutura funciona

ZMK é uma aplicação [Zephyr RTOS](https://zephyrproject.org/). O firmware sai da
combinação de três coisas:

| Peça | O que é | Onde fica |
|---|---|---|
| **board** | o microcontrolador | `xiao_ble//zmk` (upstream do ZMK) |
| **shield** | a PCB do teclado: matriz, pinos, layout | `config/boards/shields/corny/` |
| **keymap** | o que cada tecla faz | `config/corny.keymap` |

O `west.yml` declara as dependências (o ZMK em si + o módulo do widget de LED), e o
`build.yaml` é a matriz de builds que o GitHub Actions consome.

```
build.yaml                       matriz: quais board+shield compilar
config/
├── west.yml                     dependencias (zmk + zmk-rgbled-widget)
├── corny.conf                   Kconfig: bateria, LED, energia
├── corny.keymap                 <- o arquivo que voce edita no dia a dia
└── boards/shields/corny/
    ├── Kconfig.shield           declara os shields corny_left / corny_right
    ├── Kconfig.defconfig        nome do teclado, quem e o central do split
    ├── corny.dtsi               physical layout + matrix transform + kscan
    ├── corny_left.overlay       colunas do lado esquerdo
    └── corny_right.overlay      colunas do lado direito (+ col-offset 6)
```

### Por que o build é no GitHub e não na máquina

Não é uma trava — é conveniência. Compilar ZMK localmente exige CMake, Ninja, o
devicetree compiler, ARM GCC e o `west` baixando ~2 GB de árvore Zephyr + HALs. Pior
no macOS, onde [o Zephyr SDK não tem build nativo](https://zmk.dev/docs/development/local-toolchain/setup/native)
e é preciso usar o GNU ARM Embedded no lugar.

Por isso o ZMK publica um workflow reutilizável
(`zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`) que este repo chama em
3 linhas: você commita o keymap, o Actions devolve os `.uf2`.

Build local é totalmente suportado — veja
[Building and Flashing](https://zmk.dev/docs/development/local-toolchain/build-flash).

### Physical layout vs. matrix transform

O shield **não** usa `chosen zmk,matrix-transform`. Com um chosen matrix-transform, o
ZMK ignora os physical layouts e o firmware **fica incompatível com o ZMK Studio**
([doc oficial](https://zmk.dev/docs/features/studio)). Aqui o `corny_layout` declara a
propriedade `keys` com as coordenadas das 44 posições, e aponta pro transform e pro
kscan internamente.

## Ordem das posições (importa pros combos)

```
linha 0:  0..11
linha 1: 12..23
linha 2: 24..29 (esq) | 30 31 (centrais SW22) | 32..37 (dir)
linha 3: 38..40 (esq) | 41..43 (dir)
```

## LED de bateria e conexão

O `rgbled_adapter` usa o **LED RGB embutido do XIAO nRF52840**: P0.26 vermelho,
P0.30 verde, P0.06 azul (todos active-low). Vem do módulo
[zmk-rgbled-widget](https://github.com/caksoylar/zmk-rgbled-widget).

Faixas configuradas em `config/corny.conf`:

| Nível | Cor | Kconfig |
|---|---|---|
| acima de 30% | 🟢 verde | `BATTERY_LEVEL_HIGH=30` |
| 10% a 30% | 🟡 amarelo | (entre LOW e HIGH) |
| abaixo de 10% | 🔴 vermelho | `BATTERY_LEVEL_LOW=10` |
| abaixo de 5% | 🔴 vermelho piscando | `BATTERY_LEVEL_CRITICAL=5` |

Conexão BLE: 🔵 conectado, 🟡 anunciando, 🔴 desconectado.

**Quando acende:** no boot (bateria, depois conexão), a cada mudança de estado, e sob
demanda pelas teclas `&ind_bat` / `&ind_con` — camada **System** (`&mo 3`), canto
superior direito e o de baixo dele.

## Editar as teclas

### ZMK Studio (ao vivo, sem reflashear)

1. Plugue o **lado esquerdo** (central) no USB.
2. Abra <https://zmk.studio>.
3. Acione `&studio_unlock` — camada **System** (`&mo 3`), canto superior esquerdo.
4. Edite e salve. Aplica na hora.

> [!IMPORTANT]
> Depois de usar o Studio, mudanças no `corny.keymap` **não se aplicam** até você
> fazer um *Restore Stock Settings* no próprio Studio. O Studio grava o keymap nas
> settings do dispositivo, e elas têm precedência sobre o arquivo.

### Pelo arquivo (fonte da verdade versionada)

Edite `config/corny.keymap`, commite, e pegue os `.uf2` no artifact do Actions.

## Flashear

1. Baixe o artifact `firmware` do Actions (`corny_left...uf2` e `corny_right...uf2`).
2. Conecte **um lado** por USB.
3. **Duplo-toque no botão de reset** do XIAO. Monta um volume USB (`XIAO-SENSE`).
4. Arraste o `.uf2` do lado correspondente pro volume. Ele desmonta e reinicia sozinho.
5. Repita no outro lado.

**Confirmação de que pegou:** o dispositivo passa a aparecer como **`Corny`** /
**`Corny Right`** no USB. Se ainda aparecer o nome antigo, o flash não pegou.

```sh
ioreg -p IOUSB -l -w 0 | grep '"USB Product Name"'
```

### Se algo der errado

- **Voltar ao firmware anterior:** os `.uf2` pré-compilados do
  [corne-xiao](https://github.com/shuhikari/corne-xiao) rev2 6-col servem de fallback.
- **Metades não pareiam / comportamento estranho:** flashe `settings_reset.uf2` nos
  **dois** lados, depois reflashe o firmware normal nos dois.
- **Nunca reflashe o bootloader** (`Adafruit_nRF52_Bootloader`) sem necessidade — é a
  única operação que pode brickar o XIAO de verdade.
