# Revisão: o software cumpre o que promete? + Melhorias de design

## 1. O software cumpre o que promete?

Revisei o código-fonte inteiro (pipeline de filtro, captura de Raw Input, hook
de supressão) contra as afirmações do README. Conclusão: **sim, a arquitetura
descrita corresponde ao código real**, com uma ressalva importante sobre teste.

- **"Não lê tela, não lê memória de outro processo, não tem noção de
  alvo/jogo"** — confirmado. `MouseFilterPipeline::Process` só recebe
  `(dx, dy, dt)` (três `double`s) e devolve `(dx, dy)`. Não há nenhuma chamada
  a `GetPixel`, `BitBlt`, `ReadProcessMemory`, captura de tela, ou qualquer
  API de foco de janela em todo o projeto.
- **"SetCursorPos em vez de SendInput relativo, para evitar a curva de
  aceleração do Windows"** — confirmado em `InputEngine::HandleRawInput`.
- **"O hook WH_MOUSE_LL só suprime, nunca re-filtra eventos injetados
  (LLMHF_INJECTED)"** — confirmado; isso evita o loop de realimentação que o
  README promete evitar.
- **Filtro adaptativo (1-Euro Filter)** — a matemática em `OneEuroFilter.h` e
  `MouseFilterPipeline.cpp` está correta: o ganho DC do filtro passa-baixa é
  exatamente 1.0 para qualquer alpha (é uma média ponderada convexa), então a
  suavização de fato não introduz aceleração/desaceleração de ganho — só
  atraso, exatamente como a documentação afirma.
- **Anti-spike nunca "prediz"** — confirmado; `SpikeGuard::Process` só limita
  magnitude (clamp), nunca substitui ou extrapola um valor.
- Um bug real (não relacionado a design) que encontrei e corrigi: `main.cpp`
  chamava `InitCommonControlsEx` sem a flag `ICC_TAB_CLASSES`. A aba
  (`WC_TABCONTROLW`) usada em `CreateTabs` depende dessa flag para a classe
  ser registrada; sem ela, em algumas configurações `CreateWindowExW` para o
  controle de abas pode falhar silenciosamente. Corrigido.

### O que eu **não** consigo confirmar

Este ambiente é Linux, sem Windows e sem mouse físico — o mesmo limite que o
próprio README já assume ("não testado em hardware físico"). Não há como
compilar com MinGW aqui (sem acesso à internet para instalar o
cross-compiler) nem rodar o `.exe` para validar o comportamento do hook em
tempo real. Revisei o código estaticamente, linha a linha, e a lógica bate
com a documentação — mas o "compila e roda de fato num Windows real com
mouse" continua sendo algo que só você pode confirmar aí.

## 2. Melhorias de design feitas

O próprio código já admitia a limitação (comentário original em
`ApplyTheme()`): os controles nativos do Win32 (botões, sliders) ignoram o
tema escuro, então o app ficava com uma mistura de fundo escuro e botões
brancos "colados". Refiz a camada visual:

- **Fonte real do sistema** (Segoe UI, via `SPI_GETNONCLIENTMETRICS`) em vez
  da fonte "System" padrão do GDI — aplicada a todos os controles com
  `EnumChildWindows` + `WM_SETFONT`.
- **Botões com desenho próprio** (`BS_OWNERDRAW` + `WM_DRAWITEM`): cantos
  arredondados, categorias visuais (ação primária em azul, "Excluir" com
  contorno vermelho, o botão Ativado/Desativado muda para verde/cinza
  conforme o estado) — consistentes em tema claro ou escuro, já que não
  dependem mais do desenho nativo do Windows.
- **Sliders com trilho e alça customizados** (`NM_CUSTOMDRAW`): a parte já
  percorrida do slider é preenchida com a cor de destaque, o resto fica em
  cinza neutro, e a alça vira um círculo colorido — visual mais claro do que
  a barra cinza padrão do Windows.
- **Tema escuro de verdade**: além do fundo/texto que já existia, agora
  cobre também caixas de edição e listas dos comboboxes
  (`WM_CTLCOLOREDIT`/`WM_CTLCOLORLISTBOX`), aplica `SetWindowTheme` nos
  controles nativos que respeitam isso (abas, sliders, scrollbars), e ativa a
  barra de título escura via DWM quando o Windows suporta.
  Uma ressalva honesta: caixas de combo (`CBS_DROPDOWNLIST`) e alguns
  detalhes de chrome do Windows têm suporte parcial e não-oficial a dark mode
  mesmo em apps nativos da própria Microsoft — o resultado deve ficar bem
  mais coerente que antes, mas não é 100% garantido sem testar num Windows
  real.
- Aumentei um pouco o espaçamento entre linhas (`kRowHeight` 50→54) e o
  tamanho da janela (480×680 → 500×720) para dar mais respiro ao layout mais
  denso de texto/controles.

Nenhuma dessas mudanças toca a lógica do pipeline de filtro — são puramente
de apresentação (`MainWindow.h/.cpp`), então o comportamento do filtro em si
continua exatamente o mesmo.
