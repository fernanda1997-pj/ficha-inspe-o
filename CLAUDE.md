# Geoportal RTA-MSI — Inspeção do Pavimento

**Este repo hoje tem só UM app: `ficha-inspecao/`** (WebGIS Leaflet com o resultado
das fichas mensais de inspeção de rodovias do Tocantins — pavimentadas e não
pavimentadas/LEN). Todo mês saem **2 fichas por região × 6 regiões** (R1, R2, R3,
R11, R12, R13): uma de trechos pavimentados, outra de não pavimentados.

`camadas/` e `logo/` ficam na RAIZ do projeto — usados pelo `ficha-inspecao/`
(`../camadas`/`../logo/...`).

**Existiu um segundo app aqui, `ordens-servico/`** (as O.S.P. da planilha de
Controle), pasta separada, zero ligação com a ficha — em 2026-08-18 virou **repo
próprio** (`C:\1. Projetos\RTA\web - OS`, a usuária pediu explicitamente "vou criar
[repo] só para as OS"), com cópia própria de `camadas/`/`logo/` (não depende mais
deste repo). Ver `CLAUDE.md` de lá pra detalhes da arquitetura/histórico daquele
app — não duplicar aqui.

Usuária: Fernanda (RTA Engenheiros Consultores). Responder sempre em português.

- **Site**: `ficha-inspe-o.vercel.app` (deploy automático a cada push na `main`)
- **Repo**: `github.com/fernanda1997-pj/ficha-inspe-o`

## Arquitetura — ficha de inspeção (`ficha-inspecao/`)

| Arquivo/pasta | Papel |
|---|---|
| `ficha-inspecao/index.html` | App da ficha (HTML+CSS+JS, sem build). CDN: Leaflet 1.9.4 |
| `ficha-inspecao/converter_fichas.py` | Lê `fichas/*.xlsx` (dentro de `ficha-inspecao/`) + `../camadas/R*_TRECHOS.shp` (raiz, compartilhado), corta a geometria de cada S.R.E. no km início/fim de cada linha da ficha (referenciamento linear) e escreve em `ficha-inspecao/dados/` |
| `ficha-inspecao/fichas/` | Fichas de inspeção mensais, uma por região+mês, como chegam do campo (`ficha de inspeção_rodovias pavimentadas_R.<região> - <MÊS>.xlsx`) — **não editar**, só adicionar arquivos novos aqui |
| `camadas/` (raiz, compartilhado) | Cópia de `R<região>_TRECHOS.shp` (uma linha por S.R.E., com `EXT_REAL` em km) — vem do geoportal principal em `../web/camadas/`. Se um S.R.E. novo aparecer numa ficha e não achar o shapefile, é só copiar a versão atualizada de lá. Tem também `Base_Rods_2023.shp` — ver seção própria abaixo |

## Fallback Base_Rods_2023.shp (provisório — trocar quando sair a versão oficial)

`camadas/Base_Rods_2023.shp` é uma camada **estadual** (o Tocantins inteiro, 1063
trechos) que a usuária pegou de `C:\2. Banco de Dados\Banco de Dados - Shapes\
2023_RODOVIA\` — é o que o **pessoal de campo está produzindo agora**, ainda **não é
a versão oficial final**. Usado só como **fallback**: `linha_do_sre()` em
`converter_fichas.py` busca primeiro no shapefile da própria região
(`R<n>_TRECHOS.shp`) e só cai pro Base_Rods_2023 se o S.R.E. não estiver lá — nunca
substitui o que já funciona. Cobre 9 dos 13 S.R.E. que faltavam nos shapefiles
regionais (`relatorio_qualidade.txt` avisa quando um S.R.E. veio do fallback).

Esquema de colunas diferente dos `R<n>_TRECHOS.shp`: `CODIGO` (não `SRE`) e `Ext_Km`
(não `EXT_REAL`) — já cobertos pelos candidatos de `_achar_coluna()`/detecção de coluna
SRE em `_carregar_linhas_de_shapefile()`. Não tem colunas de coordenada início/fim
(tipo `X_LONG`/`Y_LAT`), então não dá pra checar o sentido da linha nos S.R.E. que vêm
de lá — assume a ordem original do shapefile (aviso único, não por feição).

**Quando a usuária avisar que o pessoal de campo já está usando a versão oficial**:
substituir `camadas/Base_Rods_2023.shp` pela nova (mesmo nome de arquivo, ou trocar o
caminho em `BASE_RODS_2023` no `converter_fichas.py`) e rodar `python converter_fichas.py`
de novo — não precisa mexer em mais nada.
| `ficha-inspecao/dados/insp_<REGIAO>_<AAAA-MM>.js` | Um GeoJSON (dentro de `window.DADOS_INSPECAO[regiao][competencia]`) por região+competência, gerado pelo converter — **não editar à mão** |
| `ficha-inspecao/dados/manifest.js` | Lista de todas as combinações região/competência disponíveis (`window.MANIFEST_INSPECAO`) + os 5 grupos de condição (`window.GRUPOS_INSPECAO`) — o `index.html` usa isso pra montar os selects e injetar os `<script>` dos arquivos `insp_*.js` sob demanda |
| `logo/` (raiz, compartilhado) | Logos RTA + MSI (copiados de `web - Mapas/logo/`) |
| `ficha-inspecao/relatorio_qualidade.txt` | Gerado a cada rodada do converter (gitignored) — aponta S.R.E. da ficha que não bateu com o shapefile, geometrias em partes desconexas, extensão inspecionada muito diferente da extensão real etc. |

## Como funciona o referenciamento linear (o coração do projeto)

A ficha lista, para cada S.R.E., uma sequência de sub-trechos por **km início/fim**
(ex.: S.R.E. `070ETO0230`, km 0–1, 1–1.3, 1.3–1.8...). O shapefile `R<região>_TRECHOS.shp`
tem **uma linha por S.R.E.**, com a extensão real batendo com o km final da ficha. O
converter:

1. Carrega a linha do S.R.E. em EPSG:31982 (métrico, SIRGAS 2000 / UTM 22S — mesmo EPSG
   do geoportal principal).
2. Descobre se o primeiro vértice da linha corresponde ao km 0 (comparando com os
   atributos `X_LONG/Y_LAT` vs `X_FIM_LONG/Y_FIM_LAT` do shapefile) e inverte a linha se
   necessário — senão os trechos saem com km início/fim trocados.
3. Corta a linha com `shapely.ops.substring(geom, km_ini*1000, km_fim*1000)`.
4. Reprojeta o pedaço cortado pra WGS84 (EPSG:4326) pro GeoJSON final.

**Shapefiles com geometria em partes desconexas** (emenda de digitalização, comum nesses
dados): o converter primeiro **descarta partes minúsculas (≤50m)** — vértice solto/
duplicado, ruído de digitalização, não estrada de verdade. Sem isso o algoritmo de
encadeamento (abaixo) desenha um "espeto" reto até esse pontinho perdido (às vezes
vários km de distância) e volta, criando um laço sem sentido no mapa — caso real
encontrado pela usuária: R3/020ETO0210 (2026-08-27). Das partes que sobram, concatena
sempre pela **parte mais próxima de uma das pontas** da linha já montada (guloso:
começa pela mais longa, gruda a mais perto em qualquer ponta, invertendo se precisar)
— preserva o comprimento total (bate com `EXT_REAL`) sem o zigue-zague feio que dava
concatenar "na ordem do arquivo". Se o vão entre duas partes *significativas* for
grande (>200 m), o `relatorio_qualidade.txt` avisa com "CONFERIR o shapefile" — pode
ser um pedaço do traçado que falta digitalizar (exemplo real corrigido em 2026-08-27:
R2/420ETO0030 tinha ~12km faltando; a usuária atualizou o shapefile no projeto
principal e copiou pra cá).

## Os dois modelos de ficha e os 7 grupos de condição

Existem **dois modelos de ficha**, com grupos de condição diferentes. O converter
detecta automaticamente qual é (por sheet, procurando o cabeçalho do 1º grupo — não usa
o nome do arquivo, que é livre): `TEMPLATES` em `converter_fichas.py`.

| Modelo | S.R.E. tipo (SITUAÇÃO) | Grupos |
|---|---|---|
| Pavimentada | PPS/PSU/PDU/EOP... | `pavimento`, `vegetacao`, `drenagem`, `sinalizacao_horizontal`, `sinalizacao_vertical` |
| Não pavimentada | `LEN` | `plataforma`, `drenagem_superficial` |

Cada linha da ficha pode ter mais de uma marcação (X) dentro do mesmo grupo (ex.: um km
com "Remendo em lâmina" **e** "Buraco em lâmina" ao mesmo tempo). Pra cor no mapa, vale a
regra **pior marcação vence**: a severidade é a posição da coluna dentro do grupo
(0 = melhor, a ficha sempre desenha da esquerda/melhor pra direita/pior) — por isso o
converter não depende do texto exato do rótulo (tem, inclusive, um erro de digitação na
ficha original de pavimentada: "INADED." em vez de "INADEQ.").

| Grupo (`id`) | Severidades (0→pior) | Ficha |
|---|---|---|
| `pavimento` | Bom · Remendo isolado · Remendo em lâmina · Buraco isolado · Buraco em lâmina | Pavimentada |
| `vegetacao` | Adequada · Inadequada | Pavimentada |
| `drenagem` | Limpos · Sujos · Danificados | Pavimentada |
| `sinalizacao_horizontal` | Bom · Regular · Inexistente | Pavimentada |
| `sinalizacao_vertical` | Bom · Poucas · Inexistente | Pavimentada |
| `plataforma` | Bom · Regular (até 10 irreg./km) · Ruim (+10 irreg./km) · Péssima (atoleiro/pto. crítico) | Não pavimentada |
| `drenagem_superficial` | Limpa · Obstruída · Ausente | Não pavimentada |

Cada segmento (feature do GeoJSON) só carrega as chaves do grupo do SEU modelo — um
trecho pavimentado nunca tem `plataforma`/`drenagem_superficial` e vice-versa. Isso é
usado pra filtrar colunas na tabela do funil (`f.properties[grupoId] !== undefined`).

**Histórico (revertido):** os 5/2 grupos já foram camadas independentes do mapa
(checkbox por aspecto, com `turf.lineOffset` deslocando as linhas ~6m pra comparar
lado a lado). A usuária achou confuso mesmo depois de numerar as camadas e explicar
com exemplo — pediu pra tirar (2026-08-04). `turf.js` foi removido do projeto (só
existia pro offset). Não recriar esse formato de camadas sobrepostas/deslocadas.

**⚠️ Tudo do "Colorir mapa por" abaixo (até a linha do mockup escuro) foi
REMOVIDO em 2026-09-03** (mesmo dia, mais tarde — "essa parte de 'colori mapa
por' pode tirar tbm") — `aspectoAtual` virou constante fixa `'icm'`,
`selecionarAspecto`/`montarGradeAspectos` não existem mais, não tem mais
select nem grade nenhuma no painel. Ver a entrada "As 2 abas viraram 1
cabeçalho" mais abaixo pro estado atual. Fica descrito aqui só pra explicar
POR QUE `classeDoAspecto`/`somaPorAspecto`/`estiloDoSegmento` continuam
genéricos por baixo do capô (não foram revertidos, só perderam o controle de
UI) — e pra não recriar isso de novo sem pedido explícito.

**"Colorir mapa por" (2026-09-02, virou grade de cards em 2026-09-03):** a usuária
reclamou que o Resultado Geral (média de todos os aspectos) escondia detalhe
importante — "os dados ficavam muito vagos". Criou-se um controle compartilhado
pelas duas abas que recolore o mapa INTEIRO por um aspecto de cada vez (nunca duas
camadas ao mesmo tempo, sem offset) — `GRUPOS` = `GRUPOS_INSPECAO` + `{id:'icm',
nome:'Resultado Geral'}`. Selecionar um aspecto muda `aspectoAtual` (função
`selecionarAspecto(id)`), reseta os filtros de checkbox (`ativosAspectoGeral`/
`ativosAspectoRegiao` — os níveis de um aspecto não têm relação com os de outro) e
redesenha Visão Geral + Por Região. A generalização do que antes só existia pro
I.C.M.: `classeDoAspecto(props, grupoId)` / `somaPorAspecto(feats, grupoId)`
(paralelo a `somaIcmDe`, mas descobre os níveis a partir dos dados de verdade em vez
de uma lista fixa tipo `CLASSES_ICM`) e `ordemClasses` (`[{chave,nome,cor}]`) como
formato comum que alimenta a barra, legenda e filtro tanto pro I.C.M. quanto pra
qualquer aspecto. Trecho que não tem aquele aspecto (ex.: `vegetacao` numa via não
pavimentada) cai em `'sem_info'`, cinza `#94A3B8`. `montarComparativoRegioes` (a
barra "Comparativo por região" na Visão Geral) ficou de propósito só no Resultado
Geral — não segue o seletor.

Era um `<select>` simples; virou `montarGradeAspectos()` — uma esteira horizontal
de `.card-aspecto` (um por `GRUPOS`, ícone + nome + `% coberto` + mini-barra),
pedido da usuária: "não estou gostando desse geoportal... queria algo mais dinâmico,
bonito". Cada card já mostra a composição daquele aspecto de relance (sem precisar
abrir um dropdown e trocar um por um) e clicar nele chama `selecionarAspecto()`; o
card do `aspectoAtual` fica com destaque (`.ativo`). O número do card é cobertura
(`100 - % 'sem_info'`), não km total — km total é ~igual em todo card (mesmo
universo de trechos) e não diferenciava nada; cobertura já separa visualmente os
aspectos exclusivos de pavimentada (`pavimento`/`vegetacao`/`drenagem`/
`sinalizacao_*`, cobertura ≈ % pavimentada) dos exclusivos de não pavimentada
(`plataforma`/`drenagem_superficial`, cobertura ≈ % não pavimentada). Chamada em
dois pontos: uma vez no carregamento do script (pintura placeholder, ainda sem
dado) e de novo dentro de `desenharVisaoGeral()` (com os totais reais e o card
ativo certo) — só essa função dispara recálculo de todos os 8 grupos de uma vez,
então não precisa ser chamada em mais lugar nenhum.

**Donut → barra empilhada (2026-09-03):** o gráfico de composição (km por
classe/nível) era um donut/pizza; a usuária pediu um gráfico melhor. Virou
`montarBarra()` — uma barra horizontal 100% empilhada com o total em destaque acima
(`.barra-total` + `.barra-empilhada`), a mesma família visual que "Comparativo por
região" já usava. Motivo: comparar comprimento numa reta é mais rápido de ler que
comparar ângulo/arco de fatia, principal queixa de clareza da usuária nessa mesma
conversa. Assinatura da função não mudou (`somaPorClasse, total, idAlvo,
ordemClasses`), só o nome (era `montarDonut`) e os ids/classes CSS (`geral-barra`/
`regiao-barra`, antes `geral-donut`/`regiao-donut`; `.barra-wrap-central`, antes
`.donut-wrap-central`).

**Polish visual geral (2026-09-03):** junto com a grade de cards, um passe de
retoque no painel todo pra parecer menos "cru" (mesmo pedido: "mais dinâmico,
bonito") — cabeçalho com gradiente (`linear-gradient(135deg, --azul, --azul-claro)`
em vez de cor chapada), hover com leve elevação (`translateY`/`translateX` +
sombra) nos cards de KPI, `.comp-regiao` e `.card-aspecto`, transição suave nas
abas/legenda/select, anel de foco azul em select/busca, e scrollbar fina
customizada no painel. Só CSS — nenhuma mudança de dado ou comportamento.

**Painel virou "dashboard" — 460px, grade que quebra linha em vez de esteira de
scroll (2026-09-03, mesmo dia):** a versão acima da grade de cards ("esteira
horizontal, arraste ← →" dentro do painel de 330px) foi recebida como "horrível".
Mostrei um mockup mais largo (ferramenta `visualize`, fora do site) com KPIs em
linha, grade 2 colunas e "Resultado geral" num cartão com borda — a usuária gostou
do formato e escolheu a opção mais simples de encaixar isso no site de verdade:
alargar o `#painel` inteiro (não um modal, não uma 3ª aba). Mudanças:
- `#painel{width:460px}` (era 330px) — só a REGRA DESKTOP; o `@media (max-width:
  768px)` continua cravando 85%/340px por cima, então o celular não muda em nada.
  `#drawer{left:...}` teve que acompanhar o mesmo valor (460px), senão a gaveta de
  detalhe do S.R.E. abre por baixo do painel.
- `.grade-aspectos`: `display:flex; overflow-x:auto` virou `display:grid;
  grid-template-columns:repeat(auto-fit, minmax(180px, 1fr))` — quebra em 2
  colunas (4 linhas pros 8 `GRUPOS`) em vez de rolar; no celular (painel ainda
  340px) cai sozinho pra 1 coluna, também sem precisar rolar.
- `.barra-wrap-central` + `.legenda-filtro` (Resultado Geral / Por Região) viraram
  UM cartão só (borda compartilhada, cantos arredondados só nas pontas de fora) em
  vez de ficarem soltos no painel — visual "hero" de dashboard, mesma ideia do
  mockup.
- `.kpis-grade-4` foi de 2 pra 4 colunas (cabe numa linha só com o painel mais
  largo).
Se pedir pra alargar mais ou mudar pra modal/aba cheia depois, o registro de
alternativas consideradas está na resposta que ofereceu as 3 opções — a usuária
escolheu explicitamente "alargar o painel" em vez de modal ou 3ª aba.

**Tema escuro + filtro por região + tabela heatmap (2026-09-03, ainda o mesmo
dia):** a grade de cards larga ainda não agradou — a usuária mandou print de um
dashboard SaaS de referência (fundo escuro, pílulas de filtro, cards de KPI,
tabela com célula colorida por intensidade) e disse "algo mais assim". Mostrei
outro mockup reproduzindo esse visual com os dados reais do site (sem os banners
de alerta — pediu pra tirar) e ela confirmou tema escuro + gostou do filtro por
região do topo; a pergunta em aberto foi só "como encaixar o mapa" — escolheu
**adicionar uma opção de mapa escuro** (não trocar o padrão nem manter só claro).

- **Tema escuro só no `#painel`:** variáveis `--p-bg`/`--p-bg-2`/`--p-bg-3`/
  `--p-borda`/`--p-texto`/`--p-texto-2`/`--p-azul`/`--p-trilha` declaradas
  DENTRO do seletor `#painel` (não em `:root`) — os tokens globais
  (`--azul`/`--fundo`/`--borda`/`--cinza`) continuam intactos e usados por
  `#drawer`/`.popup-insp`/controles do Leaflet, que ficam CLAROS de propósito
  — o basemap "Padrão" continua claro por padrão, e o mapa escuro é uma opção
  que a usuária escolhe (ver basemap "Escuro" abaixo), não o padrão forçado.
  Cores de status (verde/amarelo/laranja/vermelho) não mudaram — já liam bem em
  fundo escuro.
- **Basemap "Escuro":** `baseEscuro` usa
  `Canvas/World_Dark_Gray_Base` do ArcGIS REST (mesma família Esri do
  `World_Light_Gray_Base` já usado antes pro "Padrão" em algum momento) — não
  precisa de API key, ao contrário do CARTO Dark Matter (CARTO já deu problema
  de API key em produção nesse projeto, ver histórico de basemap). Registrado
  como 3º radio em `L.control.layers`, ao lado de Padrão/Satélite; Padrão
  continua OpenStreetMap (bate com mapa-levantamento, isso não mudou).
- **Filtro por pílulas de região (`.pills-regiao`):** "gostei da parte de cima
  que separa por região" — pedido novo, não é só CSS. `regiaoFiltroGeral`
  (`''` = Todas) recorta `todasAsFeatures()` via `featuresDaVisaoGeral()`,
  usado por `desenharVisaoGeral()` — filtra KPIs, Resultado Geral E O MAPA
  (reaproveita `redesenharMapaGeral`, que já dá `fitBounds` sozinho).
  `montarComparativoRegioes()` continua recebendo `todasAsFeatures()` SEM
  filtro de propósito — ela existe pra comparar regiões entre si, filtrar pra
  uma só a esvaziaria. **Nesse mesmo dia isso ainda vivia dentro da aba "Visão
  Geral"** (com uma aba "Por Região" separada) — ver entrada seguinte pra como
  isso virou a navegação principal do site.
- **"Comparativo por região" virou tabela heatmap:** era uma lista de cartões
  com barrinha (`.comp-regiao`, removido); virou `<table class="tabela-heatmap">`
  — uma linha por região, uma coluna por classe do Resultado Geral, célula
  colorida pela cor de STATUS da própria classe (`corHeatmap()`: mistura a cor
  com transparência proporcional ao %) em vez de uma escala neutra genérica tipo
  a referência — mantém a mesma linguagem de cor (verde=Bom, vermelho=Péssimo)
  do resto do site. Clicar na linha ainda leva pra "Por Região" com a região
  certa (mesmo comportamento de antes, só mudou de `<div>` pra `<tr>`).
- **Não implementado (recusado explicitamente):** os banners de alerta/insight
  do mockup ("Cobertura baixa em vias não pavimentadas", "5 S.R.E. sem
  geometria") eram ilustrativos pra mostrar o estilo — a usuária disse "deixa
  sem alertas". Não recriar sem pedido explícito; se pedir depois, envolve
  lógica nova (regras de quando/o que virar alerta), não é só visual.

**As 2 abas viraram 1 cabeçalho de região; seletor "Colorir mapa por" foi
removido (2026-09-03, ainda o mesmo dia):** pedido final da usuária depois de
ver as pílulas funcionando: "um cabeçário horizontal, aonde vamos manter essa
parte da imagem das regiões... não vai mais ter as 2 abas, e quando a gente
seleciona a região aparece as opções de filtrar". Ou seja: as pílulas de
região deixam de ser um filtro *dentro* da aba "Visão Geral" e viram A
navegação do site inteiro — "Todas" = dashboard agregado, qualquer região
específica = funil detalhado (Competência → Tipo → Trecho → S.R.E.), sem
aba nenhuma. No mesmo fôlego ela pediu pra tirar a grade "Colorir mapa por"
também ("essa parte... pode tirar tbm") — o mapa voltou a ser sempre colorido
pelo Resultado Geral, como era antes de 2026-09-02.

- **`selecionarRegiao(regiao)`** é a função central nova, substituindo
  `trocarAba(aba)` (removida). `''` mostra `#painel-geral` (dashboard) e
  chama `desenharVisaoGeral()`; qualquer código de região mostra
  `#painel-regiao` (funil) e chama `desenharRegiao()` + `montarCompetencias()`
  + `redesenharMapa()` — mesma limpeza de camadas do mapa que `trocarAba` já
  fazia (fechar drawer, tirar destaque, remover a camada da visão que estava
  saindo). Chamada de `montarPillsRegiao()` (destaca a pílula ativa),
  `montarComparativoRegioes()` (clique na linha da tabela), `irParaTrecho()`
  (busca e link direto) e no boot (`carregarTodosOsDados(...selecionarRegiao(''))`).
- **`<select id="sel-regiao">` continua existindo, só escondido**
  (`style="display:none"` no HTML) — MUITO código antigo (`atualizarURL`,
  `redesenharMapa`, `aoCarregarDados`, `historicoDoSre`...) lê `selRegiao.value`
  como fonte da verdade pra qual região está aberta; `selecionarRegiao()`
  mantém esse `<select>` sincronizado (`selRegiao.value = regiao`) toda vez
  que muda. Reescrever esse código todo pra uma variável solta não valia o
  risco — o `<select>` é só um "estado interno" agora, ninguém vê ele.
  `montarSelects()` só monta as `<option>` (precisa existir pra `.value =`
  funcionar) — não desenha mais nada no mapa no boot (antes desenhava a
  região 1 e a "Visão Geral" tinha que desfazer isso; virou desperdício sem
  sentido já que o padrão agora é "Todas" de verdade).
- **`GRUPOS`/`estiloDoSegmento`/`classeDoAspecto`/`somaPorAspecto` continuam
  genéricos** (aceitam qualquer `grupoId`) por baixo do capô — só o CONTROLE
  de cor do mapa (a grade de cards) saiu. `aspectoAtual` virou uma constante
  fixa `'icm'`, nunca mais muda. O detalhe por aspecto (Pavimento/Vegetação/
  Drenagem/Sinalização/Plataforma/Drenagem Superficial) continua 100%
  disponível no popup do trecho e na tabela do funil — só parou de ser
  como o MAPA é colorido. **Não recriar a grade de cards (clicável, muda o
  mapa) sem pedido explícito** — foi construída, testada, publicada e
  removida no mesmo dia; a usuária quer menos CONTROLE nessa área, não
  menos DADO — ver entrada seguinte, que é sobre dado, não controle.

**"Por aspecto avaliado" — Resultado Geral saiu do dashboard, virou lista por
aspecto (2026-09-03, ainda o mesmo dia):** pouco depois de pedir a remoção da
grade de cards, a usuária pediu o oposto na direção do DADO (não do
controle): "invés de coloca dados do resultado geral, deixa especifico de
cada um: vegetação, condição etc" — confirmou que era tanto no card grande
quanto na tabela "Comparativo por região". Ou seja: ela não queria a grade de
cards CLICÁVEL que mudava a cor do mapa (isso continua fora — mapa sempre
Resultado Geral), mas queria sim ver o dado de cada aspecto separado, só que
como LEITURA, não como controle.

- **Card "Resultado geral" → `montarAspectosPorGrupo(feats, alvoId)`:** troca
  a barra única (Bom/Regular/Ruim/Péssimo/Sem Informação) por uma
  `.aspecto-linha` pra cada um dos 7 aspectos reais (`GRUPOS` sem o `icm`) —
  ícone, km total, barra + legenda própria (níveis de severidade daquele
  aspecto, via `somaPorAspecto`). Sem checkbox, sem clique — só consulta.
  `ICONE_ASPECTO` voltou a existir só pra isso (tinha sido removido junto
  com a grade). **`alvoId` genérico desde o pedido seguinte no mesmo dia**
  ("faça aspecto avaliado para cada região tbm") — a mesma função alimenta
  `#aspectos-geral` (chamada de `desenharVisaoGeral()`, com
  `featuresDaVisaoGeral()`) e `#aspectos-regiao` (chamada de
  `atualizarResumoRegiao()`, com `featuresParaResumo()` — já recortado pelo
  funil Tipo/Trecho/S.R.E., então escolher "Não pavimentada" faz Pavimento/
  Vegetação/Drenagem virarem 100% Sem Informação ali, é o esperado).
- **Tabela "Comparativo por região" → colunas viraram aspectos:** era
  Região × classe do Resultado Geral; virou Região × aspecto, célula = % na
  MELHOR classe daquele aspecto naquela região (`ordemClasses[0]` que não é
  `'sem_info'` — vem ordenado 0→pior, então a primeira é sempre a melhor).
  Cor da célula sempre verde (`corHeatmap('#0ca30c', pct)`) — diferente de
  antes (cor por classe), porque agora É sempre "quanto maior, melhor"
  (% em boa condição), não faz sentido variar o matiz por coluna. Nomes de
  coluna abreviados (`NOME_CURTO_ASPECTO` — "Sinal. H", "Dren. Superf."...)
  porque 7 colunas + região não cabem nos 460px do painel; `#comparativo-regioes{overflow-x:auto}`
  deixa rolar na horizontal em vez de espremer o texto.
- **O que NÃO mudou:** o mapa continua sempre Resultado Geral, em Todas E em
  qualquer região — isso é sobre os CARDS/TABELA/LISTA, não o mapa (a grade
  de cards clicável que trocava a cor do mapa continua removida, ver entrada
  anterior). A barra/legenda COM checkbox do Resultado Geral
  (`#regiao-barra`/`#regiao-legenda`, que filtra o mapa da região) também não
  mudou nem sumiu — "Por aspecto avaliado" entrou como uma lista A MAIS
  logo abaixo dela dentro da região, não em troca.
- **Ficou órfão e foi removido junto:** `montarLegendaFiltroIcm`,
  `atualizarDonutGeralFiltrado`, `ativosAspectoGeral`, `geralSomaPorClasse`/
  `geralTotal`/`geralOrdemClasses` — eram só pro checkbox-filtro da barra
  única que não existe mais. `redesenharMapaGeral` simplificou (sempre
  desenha todas as features recebidas, sem filtrar por classe marcada).

**"O que mostrar" — usuária escolhe quais seções aparecem (2026-09-03, ainda
o mesmo dia):** mandou print da barra/legenda do Resultado Geral **dentro de
uma região** (a mesma coisa que `montarLegendaFiltroRegiao` desenha, ver
acima) e pediu "pode tirar isso e eu poder escolher o que eu quero ativado".
Perguntei se era só tirar o gráfico (manter a legenda) ou remover a seção
inteira com um menu de controle — escolheu a segunda.

- **`#config-wrap`** (botão "⚙️ O que mostrar" + `#popover-config`) fica no
  mesmo nível de `#pills-regiao` — global, sempre visível, não dentro de
  `.conteudo` nenhum. 4 checkboxes, cada um controlando uma seção POR
  CONCEITO (não por tela): `kpis` esconde tanto `#kpis-geral-wrap` (Todas)
  quanto `#kpis-regiao-wrap` (região) de uma vez só — é a mesma decisão
  "não quero ver KPI" nas duas telas, não duas preferências separadas.
  `resultadoGeral` só existe em `#resultado-regiao-wrap` (a barra+legenda
  com checkbox que filtra o mapa da região — não tem equivalente em
  "Todas", que já virou "Por aspecto avaliado" antes hoje).
  `porAspecto` esconde `#aspectos-geral-wrap` E `#aspectos-regiao-wrap`.
  `comparativo` só existe em `#comparativo-wrap` (só em "Todas").
- **`WRAPPERS_SECAO`** (`index.html`) é o mapa chave→lista de ids de
  `<div>`-wrapper — pra adicionar uma seção nova ao menu, envolve o
  HTML dela num wrapper com id e adiciona uma entrada aqui + um
  `<label class="linha-config">` novo no popover (o JS descobre a chave a
  partir do id do checkbox automaticamente, não precisa editar mais nada).
- **Preferência salva em `localStorage`** (chave `rta_fichas_secoes_visiveis`)
  — sobrevive a reload, é por navegador/dispositivo (não sincroniza entre
  máquinas). Padrão (`carregarPreferenciasSecoes()`, sem nada salvo ainda,
  ou `localStorage` bloqueado) é **tudo visível MENOS `resultadoGeral`**, que
  já nasce desligado.
- Aplicado por `aplicarVisibilidadeSecoes()` — só mexe em `style.display`
  dos wrappers (que existem fixos no HTML); as funções que já preenchiam o
  CONTEÚDO desses wrappers (`atualizarResumoRegiao`, `desenharVisaoGeral`
  etc.) não precisaram mudar nada — continuam escrevendo normalmente,
  escondido ou não.

**`resultadoGeral` virou padrão DESLIGADO (2026-09-03, minutos depois) —
SUPERADO, ver "removida de vez" mais abaixo:** a usuária mandou o MESMO
print de novo (a barra/legenda do Resultado Geral dentro de uma região) só
que agora "PODE TIRAR ISSO" em caixa alta — não queria só a opção de
desligar, queria que já viesse desligado (o controle continuava existindo,
só o padrão mudou). Não resolveu de vez: quem já tinha aberto o site antes
(mesmo sem nunca ter mexido no checkbox) podia ter `resultadoGeral: true`
salvo no `localStorage` de uma sessão anterior, e isso pesa mais que o
padrão do código — ela continuou vendo a seção e voltou a pedir, ver
abaixo.

**Bug real: "Sem Informação" inflado por trecho do tipo de via ERRADO
(2026-09-03, ainda o mesmo dia):** a usuária estranhou "Por aspecto
avaliado" mostrando uns 40–50% "Sem Informação" em quase todo aspecto — não
entendeu o que era. Causa raiz: `somaPorAspecto()` jogava no balde
`sem_info` tanto (a) trecho que devia ter o dado e a ficha não marcou
QUANTO (b) trecho do tipo de via ERRADO pra aquele aspecto (ex.: uma via
não pavimentada não tem `vegetacao` — o campo nem existe nela, por design,
ver a tabela de grupos mais acima). Como pavimentada/não pavimentada é
quase meio a meio, isso inflava TODO aspecto pra ~50% "Sem Informação" só
por causa da metade da malha que era do tipo errado — mascarava totalmente
os gaps de dado reais (que eram bem menores, 1–5%).

Corrigido com `TIPO_VIA_DO_ASPECTO` (`index.html`, perto de
`somaPorAspecto`): mapa aspecto → tipo de via que ele pertence (mesma tabela
do CLAUDE.md, "Grupo (`id`)" acima). `somaPorAspecto()` agora PULA (nem
soma no total) trecho do tipo errado — só quem é do tipo certo e mesmo
assim não tem o dado vira "Sem Informação". Efeito colateral bom: o "km
total" de cada card em "Por aspecto avaliado" deixou de ser sempre
12.591 km pra todo aspecto — agora é o km da malha realmente aplicável
(ex.: Pavimento/Vegetação/Drenagem/Sinalização ~6.543 km = só via
pavimentada; Plataforma/Drenagem Superficial ~6.048 km = só não
pavimentada), e a tabela "Comparativo por região" (que já usava
`somaPorAspecto` por baixo) também ficou mais precisa sem precisar mexer
nela. `classeDoAspecto()` (usada só quando `aspectoAtual` != `'icm'`, hoje
sempre `'icm'` — ver histórico do seletor removido) NÃO foi alterada, seria
o mesmo ajuste se algum dia o mapa voltar a colorir por aspecto específico.

**Resultado Geral da região removida DE VEZ, não só desligada (2026-09-03,
mais tarde ainda):** terceira vez que a usuária mandou o mesmo print —
"TIRA POR REGIÃO ISSO". Como desligar por padrão (entrada acima) não
resolveu por causa do `localStorage` de sessões antigas com `resultadoGeral:
true` salvo, a solução definitiva foi tirar a seção do CÓDIGO, não só mudar
uma preferência que pode ter sido salva com o valor errado antes:

- **HTML removido:** `#resultado-regiao-wrap` (a `.barra-wrap-central` +
  `.legenda-filtro` da região) e a linha `chk-resultado-geral` do popover
  "O que mostrar" — a chave `resultadoGeral` saiu de `WRAPPERS_SECAO` e do
  `padrao` de `carregarPreferenciasSecoes()`. Se alguém ainda tiver
  `resultadoGeral` salvo no `localStorage` de antes, a chave só fica lá
  sem efeito nenhum (nada mais lê ela) — inofensivo, não precisa migração.
- **JS removido (ficou órfão):** `atualizarDonutRegiao`,
  `montarLegendaFiltroRegiao`, `montarBarra` (só existia pra essas duas
  barras, geral E região — as duas já tinham saído, então virou 100% morto),
  `ativosAspectoRegiao`. `redesenharMapa()` perdeu o filtro por
  `ativosAspectoRegiao[classe] === false` — o mapa da região sempre mostra
  tudo agora, sem checkbox nenhum pra esconder classe (só o filtro por Tipo
  de via continua, esse é outro mecanismo). `atualizarResumoRegiao()` não
  chama mais nenhuma das duas funções removidas, só
  `montarAspectosPorGrupo`.
- **CSS removido:** `.barra-wrap-central`/`.barra-total`/`.barra-empilhada`/
  `.legenda-filtro` (e descendentes) — não sobrou nenhum elemento que use
  essas classes em lugar nenhum do site.
- **Lição pra próxima vez que uma preferência "por padrão desligada" não
  bastar:** mudar o padrão de código só ajuda quem NUNCA salvou nada; quem
  já tinha uma versão anterior rodando (ou testou o checkbox) carrega o
  valor salvo pra sempre, já que não há expiração/versão no
  `rta_fichas_secoes_visiveis`. Se a peça é pra sumir de vez (não é
  realmente uma preferência que faz sentido religar), tirar do código é
  mais confiável que mexer no padrão.

**Toggle POR ASPECTO + múltiplas regiões juntas (2026-09-03, mais um
pedido):** "ME [dê] A OPÇÃO DE LIGAR E DESLIGAR O ASPECTO AVALIADO DE
ACORDO COM O QUE EU PRECISO E NA PARTE DA REGIÃO PODER LIGAR 2 REGIÕES
JUNTAS" — dois pedidos numa mensagem só.

- **Cada aspecto liga/desliga individualmente**, não só a seção "Por
  aspecto avaliado" inteira de uma vez (aquele toggle único saiu —
  substituído por isto). `aspectosVisiveis` (chave própria em localStorage,
  `rta_fichas_aspectos_visiveis`, separada de `rta_fichas_secoes_visiveis`)
  é um mapa `{grupoId: bool}`, todos `true` por padrão. `montarCheckboxesAspectos()`
  gera um checkbox por `GRUPOS` (nunca hardcoded — puxa nome/ícone de
  `GRUPOS`/`ICONE_ASPECTO`) dentro de `#config-aspectos`, numa sub-seção do
  popover "O que mostrar" (`.config-secao` + `.linha-config-sub`).
  `montarAspectosPorGrupo(feats, alvoId)` filtra `GRUPOS` por
  `aspectosVisiveis[g.id] !== false` antes de desenhar, e esconde o
  `<alvoId>-wrap` inteiro (label + cards) se a usuária desmarcar todos —
  evita título "Por aspecto avaliado" sem nada embaixo.
  **Cuidado ao mexer no popover**: o loop genérico de `secoesVisiveis`
  (`document.querySelectorAll('#popover-config input[type=checkbox]')`)
  precisa do `:not([data-aspecto-vis])` pra não capturar os checkboxes de
  aspecto (que não têm `id`, só `data-aspecto-vis`) — sem isso ele tentava
  ler `chk.id` vazio e criava uma chave `""` bugada em `secoesVisiveis`.
- **Pílulas de região viraram multi-seleção** — antes era rádio (só uma
  ativa, clicar em outra trocava); agora é checkbox (`alternarRegiao(regiao)`
  entra/sai do array `regioesFiltroGeral`, substituindo a antiga variável
  string `regiaoFiltroGeral`). "Todas" continua especial: sempre limpa a
  seleção inteira (`regioesFiltroGeral = []`), não entra na lista. O NÚMERO
  de regiões selecionadas decide o painel (`aplicarSelecaoDeRegioes()`):
  - **0 (Todas) ou 2+** → dashboard agregado (`#painel-geral`), recortado
    pro conjunto escolhido via `featuresDaVisaoGeral()` — com 2+ regiões é
    a MESMA lógica de "Todas", só filtrando por
    `regioesFiltroGeral.indexOf(regiao) !== -1` em vez de "sem filtro".
  - **Exatamente 1** → funil detalhado (`#painel-regiao`, igual sempre
    foi) — só faz sentido pra UMA região porque o funil carrega um pacote
    região+competência específico (`window.DADOS_INSPECAO[regiao][competencia]`),
    não dá pra ter Tipo/Trecho/S.R.E. de duas regiões misturadas.
  - `selecionarRegiao(regiao)` (nome antigo, mantido) agora força
    exatamente uma região (`[regiao]`) ou nenhuma (`[]`) — usada por busca,
    link direto (`irParaTrecho`) e clique na tabela "Comparativo por
    região", que sempre quiseram "abrir só isso aqui", nunca somar à
    seleção. `alternarRegiao(regiao)` (nova) é só das pílulas — dá pra
    somar/tirar uma região da seleção sem mexer nas outras.
  - Números batendo: o dashboard de 2+ regiões soma TODAS as competências
    já convertidas daquelas regiões (igual "Todas" sempre fez) — não é o
    km de uma competência só, que é o que a `#resumo` do funil mostra pra
    UMA região. Comparar os dois é comparar coisas diferentes de propósito
    (dashboard = tudo já levantado; funil = o mês escolhido no momento).

**Cards de "Por aspecto avaliado" viraram clicáveis — controlam o mapa
(2026-09-03, mais um pedido no mesmo dia):** "ASPECTO AVALIADO, TBM ME A
[O]PÇÃO DE LIGA E DESLIGA NO MAPA". Perguntei que efeito ela esperava no
mapa ao ligar um aspecto — resposta: "ATIVA IGUAL EU ATIVEI POR REGIÕES,
DEIXA POR ASPECTO AVALIADO" — ou seja, mesmo padrão de interação das
pílulas de região (clique tipo checkbox, várias podem ficar "ativas" ao
mesmo tempo, não é rádio).

- **`aspectosNoMapa`** (array de `grupoId`) guarda quais cards estão
  "ligados pro mapa" — independente de `aspectosVisiveis` (que só controla
  se o card APARECE na lista; um aspecto pode estar visível na lista sem
  estar ligado no mapa, e vice-versa não existe — se não tá visível não dá
  pra clicar nele). `recalcularAspectoAtual()` resolve a regra de fallback:
  **exatamente 1** aspecto ligado → `aspectoAtual` vira aquele grupoId, o
  mapa inteiro colore por ele; **0 ou 2+** → `aspectoAtual` volta pra
  `'icm'` (Resultado Geral). Motivo do fallback em 2+: não dá pra colorir
  uma linha por dois aspectos ao mesmo tempo sem confundir — mesmo
  problema de fundo das camadas deslocadas já rejeitadas antes, só que
  agora resolvido como regra de fallback (deixa clicar em quantos quiser,
  só avisa visualmente — ícone 🗺️ no card — qual efetivamente está valendo)
  em vez de proibir o multi-clique.
- **`aspectoAtual` deixou de ser constante** (tinha virado fixo `'icm'`
  quando a grade de cards foi removida) — voltou a ser dinâmico, só que
  agora o gatilho é `alternarAspectoNoMapa(grupoId)` (chamado pelo clique
  no `.aspecto-linha`) em vez de um seletor dedicado. Toda a maquinaria
  genérica que já existia por baixo (`estiloDoSegmento`, `classeDoAspecto`,
  `redesenharMapaGeral`, `redesenharMapa`) não precisou mudar nada — só
  passou a receber um `aspectoAtual` que de fato varia de novo.
- **Não confundir com a barra/legenda com checkbox da região**
  (`#regiao-barra`/`#regiao-legenda`, com filtro por CLASSE) — são
  mecanismos diferentes: aquela filtra quais CLASSES do aspecto atual
  aparecem no mapa (dentro do mesmo aspecto); esta troca QUAL ASPECTO
  colore o mapa inteiro. Os dois convivem: escolha o aspecto clicando no
  card, depois filtre as classes daquele aspecto na barra de cima.
- Clicar num card sempre chama `alternarAspectoNoMapa` e re-renderiza via
  `montarAspectosPorGrupo` (que já roda dentro de `desenharVisaoGeral()`/
  `atualizarResumoRegiao()`) — por isso não precisa recriar os cards à
  parte, o destaque `.ativo` e o ícone 🗺️ vêm de graça na próxima
  renderização.

**"Comparativo mês a mês" (2026-09-04, dia seguinte):** "como podemos fazer
o comparativo mes a mes?" — perguntei o formato (tabela mês×região estilo
heatmap, evolução só da região aberta, ou por aspecto) e ela escolheu "por
aspecto, não só Resultado Geral". Virou uma tabela heatmap gêmea da
"Comparativo por região" — mesmas colunas (aspectos, `NOME_CURTO_ASPECTO`),
só que as LINHAS agora são os meses (`competencia_label` do `MANIFEST`,
"Julho/2026", "Agosto/2026"...) em vez das regiões. Célula = % na melhor
classe daquele aspecto naquele mês (`pctMelhorClasse()`, extraído de dentro
de `montarComparativoRegioes` pras duas tabelas heatmap reaproveitarem —
antes a lógica de "acha a primeira classe que não é sem_info" tava
duplicada ali dentro).

- **Aparece nas DUAS telas**, com escopo diferente (ao contrário de
  "Comparativo por região", que é sempre TODAS as regiões de propósito):
  - `#comparativo-mensal-geral` (Todas/múltiplas regiões) — usa
    `featuresDoMesGeral(competencia)`, que respeita `regioesFiltroGeral`
    (filtra pelas pílulas ativas, ou todas as regiões carregadas se
    nenhuma). Diferente da "Comparativo por região": aqui faz sentido
    escopar pelo filtro, porque a pergunta é "como esse conjunto evoluiu",
    não "como as regiões se comparam entre si".
  - `#comparativo-mensal-regiao` (região específica) — usa
    `featuresParaResumo(competencia)`, que agora aceita um `competencia`
    OPCIONAL (por padrão pega `selCompetencia.value`, a que estiver
    escolhida no dropdown — mas o comparativo passa cada mês explicitamente,
    pra pegar o funil Tipo/Trecho/S.R.E. aplicado a TODOS os meses, não só
    ao selecionado). Um comparativo mês a mês do trecho/S.R.E. que estiver
    filtrado no momento, não só da região inteira.
  - `montarComparativoMensal(getFeats, alvoId)` é genérica — recebe a
    função de escopo como parâmetro, não hardcoded, exatamente pra servir
    as duas telas com a diferença de escopo acima.
- **`competenciasGlobais()`**: lê competências únicas de `MANIFEST`,
  ordena com **sort de string simples** (`"2026-07" < "2026-08"` já ordena
  cronológico certo porque o formato é `YYYY-MM`) — não precisa parsear
  data. Se um dia vier um mês de ANO diferente (ex.: "2027-01"), continua
  ordenando certo pelo mesmo motivo.
- Toggle "Comparativo mês a mês" no popover "O que mostrar"
  (`comparativoMensal` em `WRAPPERS_SECAO`, controla as duas telas juntas,
  igual `kpis`/`porAspecto` já faziam) — ligado por padrão.

**"Comparativo por região" removido DE VEZ (2026-09-04, no dia seguinte):**
"pode tirara: Comparativo por região (% na melhor classe)" — pedido direto,
sem meio-termo. Aprendendo com o episódio do Resultado Geral da região (ver
"removida DE VEZ, não só desligada" mais acima — mudar só o padrão do
toggle não bastou por causa de `localStorage` de sessões antigas), foi
direto pra remoção no código, sem passar por "desliga por padrão" primeiro:

- **HTML removido:** `#comparativo-wrap` (label + `#comparativo-regioes`)
  e a linha `chk-comparativo` do popover.
- **JS removido:** a função `montarComparativoRegioes` inteira (a chamada
  dentro de `desenharVisaoGeral()` também) e a chave `comparativo` de
  `WRAPPERS_SECAO`/`carregarPreferenciasSecoes()`. Se alguém tiver
  `comparativo` salvo no `localStorage` de antes, fica lá sem efeito —
  nada mais lê essa chave.
- **O que NÃO saiu, porque a tabela "Comparativo mês a mês" ainda usa:**
  `pctMelhorClasse()`, `corHeatmap()`, `NOME_CURTO_ASPECTO`, a classe CSS
  `.tabela-heatmap`. Só o `#comparativo-regioes{overflow-x:auto}` virou
  `#comparativo-mensal-geral{overflow-x:auto}` +
  `#comparativo-mensal-regiao{overflow-x:auto}` (esse ajuste de rolagem
  horizontal nunca tinha sido replicado pra tabela mensal — corrigido de
  passagem aqui).
- **Lição confirmada:** pra remover uma seção controlada por
  `localStorage` de vez (não é algo que a usuária vá querer religar), pular
  direto pra tirar do código em vez de só desligar o padrão — economiza um
  ciclo inteiro de "ela ainda vê, pede de novo".

**Card "ligado" no mapa com destaque forte, igual pílula de região
(2026-09-09):** "deixa mais destacado o que esta ligado e desligado" e, na
sequência, a usuária apontou o padrão a seguir: "igual esta na parte da
região". Antes, um card `.aspecto-linha` ativo (clicado pra colorir o mapa
por aquele aspecto) só ganhava borda + sombra sutis — fácil de não notar
entre os outros cards. Agora usa preenchimento sólido:

- `.aspecto-linha.ativo{background:var(--azul); ...}` — usa o **azul escuro
  original** (`--azul`, `#1E3A72`, do `:root` global) como fundo, não o
  `--p-azul` (azul claro, `#60a5fa`) que o painel escuro declara pra si
  mesmo. Motivo: `--p-azul` foi pensado pra texto/acento sobre fundo escuro,
  não pra virar o fundo de um card inteiro (com o texto escuro do card em
  cima ficaria ilegível). `--azul` já era usado como preenchimento das
  pílulas de região ativas — reaproveitar o mesmo token garante o mesmo
  peso visual "igual esta na parte da região".
- Junto, texto/ícones do card viram claros pra continuar legíveis em cima do
  azul escuro: `.nome{color:#fff}`, `.total{color:#cfe0ff}`,
  `.legenda{color:#cfe0ff}`, `.barra{background:rgba(255,255,255,.18)}`.
- Funciona igual nas duas telas onde a lista "Por aspecto avaliado"
  aparece: `#aspectos-geral` (Todas) e `#aspectos-regiao` (dentro do funil
  de uma região específica) — é o mesmo seletor CSS, não precisou duplicar
  nada.
- Testado: computed style confere fundo `rgb(30,58,114)` e cores de texto
  claras, tanto num card da tela "Todas" quanto num card dentro da Região 2
  (`#aspectos-regiao [data-aspecto-mapa="drenagem"]`), sem erro no console;
  conferido também no viewport mobile (375×812).

**Aviso visível quando um arquivo `dados/insp_*.js` falha ao carregar
(2026-09-10):** não foi pedido pela usuária — achado numa revisão de "o que
dá pra melhorar" (ela perguntou, apontei alguns pontos, ela confirmou "pode
corrigir"). `carregarSelecionado()` e `carregarTodosOsDados()` injetam um
`<script src="dados/...">` por região+competência e só tinham `s.onload` —
sem `s.onerror`. Se um arquivo não existir de verdade (manifest.js
desatualizado apontando pra um `.js` renomeado/apagado, deploy incompleto
que subiu o manifest mas não subiu todos os dados), o `onload` nunca
dispara:

- Em `carregarTodosOsDados` (roda no boot, carrega TODOS os arquivos do
  `MANIFEST` de uma vez): o contador `restantes` nunca chegava a zero →
  `callback()` nunca rodava → o dashboard "Todas" nunca desenhava nada. A
  splash inicial (`#gp-loading`) tem uma rede de segurança de 8s que
  esconde o overlay de qualquer jeito, então o resultado prático era pior
  que "travado": a tela parecia carregada, mas vazia/quebrada, sem
  nenhuma pista do que aconteceu.
- Em `carregarSelecionado()` (troca de região/competência manual): o
  funil daquele mês simplesmente não atualizava, sem mensagem nenhuma.

Corrigido com `s.onerror` nas duas funções, mais uma função nova
`avisarFalhaCarregamento(msg)`: loga `console.error` (debug) e mostra um
banner vermelho fixo no topo (`#gp-erro-dados`, HTML logo depois do
`#gp-loading`) com a lista de arquivos que falharam + botão de fechar.
Em `carregarTodosOsDados`, o arquivo que falha **ainda conta como
"terminado"** (`umTerminou()` chamado tanto no `onload` quanto no
`onerror`) — um mês/região com problema não trava mais os outros que
carregaram OK; o dashboard desenha com o que deu certo e avisa o que
faltou. Testado renomeando temporariamente um arquivo real do manifest
(`dados/insp_R2_2026-07.js` → 404 de propósito) e confirmando via
`fetch()`/script injetado isolado que o navegador dispara `onerror`
corretamente pra um recurso 404 nesse setup (o teste end-to-end com o
código do app em si não disparou o banner na mesma aba porque o
navegador já tinha esse arquivo em cache HTTP de um load anterior bem-
sucedido — artefato do método de teste, não do código; a lógica em si
foi conferida por leitura + pelo teste isolado do `onerror`). Banner
também conferido visualmente (conteúdo + botão fechar) em viewport
mobile 400×300.

## Resultado Geral (I.C.M. / I.C.M.N.P.)

Índice único por segmento, combinando todos os aspectos daquele modelo de ficha:
severidade de cada grupo presente ÷ severidade máxima do grupo (normaliza 0–1), tira a
média, enquadra em faixas de 25% (`calcular_icm()` em `converter_fichas.py`):

| Faixa da média | Classe |
|---|---|
| 0–25% | Bom |
| 25–50% | Regular |
| 50–75% | Ruim |
| 75–100% | Péssimo |
| nenhum grupo com marcação | Sem Informação |

Guardado em `properties.icm = {classe, valor}` de cada feature — nome interno no
código continua "I.C.M.", mas a sigla foi tirada de todo rótulo visível na tela
(2026-08-27, pedido da usuária: "sem sigla") — na UI é só **"Resultado Geral"**.
Cores fixas (paleta de status, não a escala verde→vinho de severidade): Bom
`#0ca30c`, Regular `#e05a5a` (era `#fab219`/amarelo até 2026-09-15, ver
entrada abaixo), Ruim `#ec835a`, Péssimo `#d03b3b`, Sem Informação
`#94A3B8` (`CORES_ICM` no `index.html`).

**"Regular" (Resultado Geral) virou vermelho, não é mais amarelo
(2026-09-15):** "quero mudar a cor o que esta em aamarelo deixar em
vermelho" — pedido vago, tinha 2 amarelos idênticos (`#fab219`) em
lugares diferentes: `CORES_ICM['Regular']` (Resultado Geral: Bom·Regular·
Ruim·Péssimo) e `CORES_PONTO_CRITICO['Em execução']` (status de Pontos
Críticos: Crítico·Em execução·Resolvido). Perguntei qual dos dois — "Regular
(Resultado Geral)". Avisei que usar o MESMO vermelho do Péssimo
(`#d03b3b`) deixaria as duas classes idênticas no mapa/legenda/tabela —
ela confirmou querer um vermelho mais claro e distinguível, não o
idêntico.

- Só `CORES_ICM['Regular']` mudou, de `#fab219` pra `#e05a5a` (vermelho
  mais claro que o `#d03b3b` do Péssimo, e com matiz mais "vermelho puro"
  que o `#ec835a` laranja-salmão do Ruim — os três continuam
  distinguíveis a olho). `CORES_PONTO_CRITICO['Em execução']` **não
  mudou**, continua `#fab219` amarelo — não foi o que ela pediu.
- **Cuidado, existe um AMARELO DIFERENTE que não foi tocado**: além dos
  dois `#fab219` acima, cada aspecto individual (Pavimento, Sinalização
  Horizontal, Plataforma etc.) tem sua própria escala de severidade
  genérica `CORES_SEVERIDADE = ['#16A34A','#EAB308','#F97316','#DC2626',
  '#7F1D1D']` (verde·amarelo·laranja·vermelho·vinho, por posição da
  coluna na ficha) — QUALQUER aspecto cuja 2ª severidade se chame
  "Regular" (ex.: `sinalizacao_horizontal`: Bom·Regular·Inexistente;
  `plataforma`: Bom·Regular·Ruim·Péssima) ainda aparece em amarelo
  `#EAB308` na tabela do funil/gaveta — é uma paleta e um "Regular"
  totalmente diferentes do Resultado Geral, não fazem parte deste pedido.
  Se um dia ela reclamar de "ainda tem amarelo" apontando pra uma coluna
  de aspecto específico (não a coluna "Resultado geral"), é esse o lugar.
- Testado: busquei nos dados reais carregados (`window.DADOS_INSPECAO`)
  um segmento com `icm.classe === 'Regular'` (achado: R1/2026-07,
  S.R.E. `126ETO0130`) e conferi na gaveta que o selo "Resultado geral"
  saiu `rgb(224, 90, 90)` (o `#e05a5a` novo), diferente do `rgb(236,131,90)`
  (Ruim) e `rgb(12,163,12)` (Bom) que apareciam na mesma tabela — sem
  erro no console; conferido também em viewport mobile (375×812).

**O "AMARELO DIFERENTE" avisado acima também virou vermelho, mesmo dia
(2026-09-15, poucos minutos depois):** exatamente o cenário previsto na
entrada anterior — ela mandou print dos cards "Por aspecto avaliado" (as
barras de Pavimento/Vegetação/Drenagem/Sinalização/Plataforma) e confirmou:
"o que eu quero que muda e a cor desse amarelo para vermelho". Dessa vez é
o `CORES_SEVERIDADE`, não o `CORES_ICM` — uma paleta genérica de 5
posições (0=boa condição → 4=pior) COMPARTILHADA por todos os 7 aspectos,
usada tanto nesses cards quanto na cor do MAPA quando um aspecto é
selecionado (`aspectosNoMapa`) e nos badges do popup/gaveta.

- `CORES_SEVERIDADE[1]` mudou de `#EAB308` (amarelo) pra `#e05a5a` — o
  MESMO vermelho claro usado em `CORES_ICM['Regular']` (não inventei um
  terceiro tom; reaproveitar deixa a "linguagem de cor" do site
  consistente: esse tom = "atenção, mas não é o pior").
  `CORES_SEVERIDADE[2]` (`#F97316` laranja) e `[3]`/`[4]` (`#DC2626`
  vermelho / `#7F1D1D` vinho) não mudaram.
- **Motivo de não usar o mesmo `#DC2626` da posição 3**: como essa escala
  é compartilhada, aspectos com 4-5 níveis (`pavimento`: Bom·Remendo
  isolado·Remendo em lâmina·Buraco isolado·Buraco em lâmina; `plataforma`:
  Bom·Regular·Ruim·Péssima) usam a posição 1 E a posição 3 ao mesmo
  tempo — cor idêntica juntaria visualmente "Remendo isolado"/"Regular"
  (defeito leve) com "Buraco isolado"/"Péssima" (defeito grave) na MESMA
  barra. Perguntei e ela confirmou querer um vermelho mais claro,
  distinguível — igual já tinha decidido pro Resultado Geral.
- Aspectos com só 2-3 níveis (`vegetacao`, `drenagem`,
  `sinalizacao_horizontal`, `sinalizacao_vertical`, `drenagem_superficial`)
  não usam a posição 3/4 dessa escala, então pra eles não tinha esse
  risco de colisão — mas a cor é a mesma variável pra todo mundo, não dava
  pra mudar só pra uns.
- Testado: busquei um segmento com `pavimento.severidade === 1` (achado:
  R1/2026-08, S.R.E. `010ETO0650`) e conferi que o badge "REM. I." saiu
  `rgb(224, 90, 90)` — igual ao `Regular` do Resultado Geral, diferente do
  laranja/vermelho-escuro das posições seguintes na mesma barra; conferido
  visualmente no dashboard "Todas" (barras dos 7 cards) e em viewport
  mobile (375×812, gaveta do S.R.E. acima) — sem erro no console.

**`CORES_SEVERIDADE` redesenhada de vez — sem laranja, tudo vermelho
gradativo (2026-09-23):** a correção acima (só a posição 1) não resolveu
o problema de fundo — ela mandou o MESMO print de novo (os 7 cards "Por
aspecto avaliado") e desta vez foi sobre a paleta inteira, não só o
amarelo: "não esta bom essas cores, preciso de cores armonicas que
combinam, verde para o bom, vermelho mais forte, uma combinações
melhores". A escala tinha virado uma mistura sem lógica de matiz (verde ·
vermelho-claro · **laranja** · vermelho · vinho) — o laranja no meio
quebrava a progressão de cor, mesmo a paleta sendo tecnicamente
"correta" (cada posição distinguível da vizinha).

- Mostrei 2 mockups (widget `visualize`, fora do site, reproduzindo o
  card real com HTML/CSS igual ao `index.html`) comparando "Opção A"
  (vermelho suave, transição gradual) com "Opção B" (vermelho mais
  saturado desde a 1ª posição de defeito, salto maior) — ela escolheu
  **Opção B**.
- `CORES_SEVERIDADE` virou `['#16A34A', '#EF4444', '#DC2626', '#B91C1C',
  '#7F1D1D']` — **tirou o laranja de vez** (posição 2 não é mais
  `#F97316`). Da posição 1 em diante é tudo vermelho, só mais escuro/
  saturado a cada posição — uma única família de cor pra "tem defeito",
  intensidade crescente = gravidade crescente. Só a posição 0 (Bom)
  continua verde (`#16A34A`, não mudou).
- **`CORES_ICM['Regular']` (Resultado Geral) NÃO foi tocado nesta
  rodada** — continua `#e05a5a`, o tom mais claro escolhido em
  2026-09-15. As duas paletas ficaram levemente inconsistentes entre si
  (tom de vermelho da posição 1 diferente entre "Por aspecto avaliado" e
  "Resultado Geral") — se ela reclamar de novo apontando pro Resultado
  Geral, é só alinhar `CORES_ICM['Regular']` pro mesmo `#EF4444` desta
  paleta.
- Testado: `getComputedStyle` nas barras dos cards confirma
  `rgb(22,163,74)` (Bom) → `rgb(239,68,68)` → `rgb(220,38,38)` →
  `rgb(185,28,28)` → `rgb(127,29,29)` (Buraco em lâmina/pior), sem laranja
  em lugar nenhum; conferido visualmente no dashboard "Todas" (desktop e
  mobile 375×812); sem erro no console.

**3º ajuste na MESMA paleta, minutos depois — "muito tudo vermelho" não
era só intensidade (2026-09-23):** a Opção B acima (vermelho Tailwind
puro — 500/600/700/900, todos o MESMO matiz, só variando de claro pra
escuro) ainda não agradou: "agora esta muio tudo vermelho, me da uma
opção sobre deixa isso bonito, pelo amor de deus". Diagnóstico: o
problema nunca foi só "vermelho demais" — era a paleta ter 4 posições no
mesmo matiz exato ("vermelho de sinal de trânsito"), sem nenhuma variação
de TOM entre elas, só de claridade. Isso lê como "tudo igual, só mais
escuro", cansativo visualmente, mesmo cada posição sendo tecnicamente
distinguível.

Dessa vez não mostrei mockup de novo (ela já tinha escolhido uma vez e
não resolveu — pedir pra escolher de novo entre opções seria repetir o
ciclo) — implementei direto uma progressão com variação de MATIZ, não só
de claridade, dentro da mesma família quente (sem reintroduzir amarelo/
laranja, que ela já tinha pedido pra tirar 2x):

- `CORES_SEVERIDADE` virou `['#16A34A', '#E2725B', '#C0392B', '#922B21',
  '#641E16']` — terracota → vermelho-romã (o "pomegranate" da paleta
  Flat UI Colors, tom bem conhecido por ficar elegante) → tijolo → vinho
  escuro. Cada posição tem um matiz levemente diferente da anterior
  (não só a mesma cor mais escura), o que cria variedade visual mesmo
  todas sendo "vermelho".
- Continua sem amarelo/laranja — só a posição 0 (Bom) é verde, igual
  sempre foi.
- **Cuidado de contraste**: `.badge-tab` (selos na gaveta/funil) tem
  `color:#fff` fixo — testei que mesmo a posição 1 (`#E2725B`, a mais
  clara das 4 novas) mantém texto branco legível (contraste ~3:1,
  mesma faixa que a posição 1 já aprovada da rodada anterior); não dá
  pra usar tons muito mais claros que isso sem trocar a cor do texto
  do badge também.
- `CORES_ICM['Regular']` continua `#e05a5a`, sem mudar — a
  inconsistência com o Resultado Geral (nota da entrada anterior)
  segue de pé.
- Testado: `getComputedStyle` confirma `rgb(226,114,91)` →
  `rgb(192,57,43)` → `rgb(146,43,33)` → `rgb(100,30,22)`; texto branco
  dos badges "INAD."/"SUJO"/"POUC." conferido legível visualmente;
  dashboard "Todas" e gaveta de S.R.E. conferidos em desktop e mobile
  (375×812); sem erro no console.

**4º ajuste — só a posição 1 (terracota), minutos depois (2026-09-23):**
feedback direto e específico dessa vez: "esse terracorta que não esta
bom". Diagnóstico: `#E2725B` puxava pra ALARANJADO (canal G alto demais
pra ser "vermelho de verdade"), destoando das outras 3 posições (romã/
tijolo/vinho, que são vermelho puro, sem nada de laranja) — o problema
não era a paleta inteira de novo, era só essa cor específica quebrando a
família.

- Posição 1 trocada de `#E2725B` (terracota) pra `#CD5C5C` ("indian
  red" — tom clássico, R e B/G equilibrados o bastante pra não ter viés
  de matiz pra laranja nem pra rosa). `CORES_SEVERIDADE` final:
  `['#16A34A', '#CD5C5C', '#C0392B', '#922B21', '#641E16']`. Posições
  0 e 2-4 não mudaram.
- Contraste do texto branco do badge melhorou nessa troca (mais escuro
  que o terracota anterior), sem precisar verificar de novo com cautela
  extra.
- Testado: `getComputedStyle` confirma `rgb(205,92,92)`; conferido
  visualmente no dashboard "Todas" e na gaveta de S.R.E., desktop e
  mobile (375×812); sem erro no console.

**Paleta virou categórica — uma cor por TIPO de achado, não mais um
gradiente compartilhado (2026-09-23, mesmo dia, 5º e último ajuste dessa
sequência):** a usuária mandou uma paleta pronta, com hex exato de cada
categoria: BOM verde `#2E7D32`, REGULAR âmbar `#FBC02D`, RUIM laranja
escuro `#E65100`, PESSIMO vermelho `#B71C1C`, INAD roxo `#8E24AA`, SUJO
marrom `#6D4C41`, POUC ciano `#00ACC1`, BUR.I. vermelho vivo `#FF1744`,
BUR.L. vinho `#880E4F` — ela mesma descreveu como "mantendo a lógica de
gradiente — do melhor estado ao pior estado, seguido pelas categorias de
ocorrência pontual". Isso é uma mudança de FILOSOFIA, não só de tom: os 4
ajustes anteriores (2026-09-15 a mais cedo hoje) eram todos variações de
"clarear/escurecer o mesmo vermelho"; esta é "cada tipo de problema tem
sua própria identidade visual".

- **De array plano pra objeto por aspecto**: `CORES_SEVERIDADE` deixou de
  ser `['cor0','cor1',...]` (uma escala, compartilhada por todos os 7
  aspectos) e virou `{pavimento:[...], vegetacao:[...], drenagem:[...],
  ...}` — um array de cores PRÓPRIO por `grupoId`, mesmo tamanho da
  paleta daquele aspecto. `corDoSegmento()` e `somaPorAspecto()` (os 2
  únicos lugares que liam `CORES_SEVERIDADE[nivel]`) passaram a ler
  `CORES_SEVERIDADE[grupoId][nivel]`.
- **Por que indexar por posição numérica (`severidade`) e não por texto
  exato (`status`)**: cheguei a considerar um dicionário `{texto: cor}`
  direto, mas o texto de `info.status` vem do cabeçalho da própria
  planilha da ficha e VARIA — o mesmo achado aparece como "REGULAR",
  "REG." ou "REG. (ATÉ 10 IRR./KM)" dependendo do arquivo/região/mês;
  "BUR. I", "BUR. I." ou "REM. I."; tem até o typo conhecido "INADED."
  (ver `TIPO_VIA_DO_ASPECTO` mais acima). Confirmei rodando um grep em
  todos os `dados/insp_*.js` reais — só pra "bom/regular/ruim" já
  existiam 8+ variantes de texto. Um dicionário de texto exato quebraria
  (caía no cinza de "sem cor") toda vez que uma ficha nova escrevesse o
  mesmo achado com uma palavra ligeiramente diferente. `severidade` é a
  posição numérica (0,1,2...) que o converter sempre preenche de forma
  limpa, não importa a palavra exata — MUITO mais robusto pra essa
  variação real dos dados.
- **Só as posições que ela deu cor específica usam a cor dela**:
  Inadequada (`vegetacao[1]`), Sujos (`drenagem[1]`), Poucas
  (`sinalizacao_vertical[1]`), Buraco isolado (`pavimento[3]`), Buraco em
  lâmina (`pavimento[4]`). **Posições que ela NÃO cobriu, preenchidas com
  escolha própria** (avisado na entrega, não é pedido explícito):
  - `pavimento[1]`/`pavimento[2]` (Remendo isolado/em lâmina) → reusam
    âmbar/laranja do gradiente genérico dela (são a versão "leve"/"média"
    do mesmo aspecto, cabem na lógica "melhor→pior" que ela descreveu).
  - `drenagem[2]` (Danificados, pior nível desse aspecto de 3 níveis) →
    marrom mais escuro (`#3E2723`), mesma família de "Sujos" mas mais
    grave — mesma técnica que ela já usou em Buraco isolado→lâmina
    (mesma família, mais escuro = mais grave).
  - `sinalizacao_horizontal[2]`/`sinalizacao_vertical[2]` (Inexistente,
    pior nível dos dois) e `drenagem_superficial[2]` (Ausente) → cinza-
    azulado escuro `#37474F` (`COR_INEXISTENTE`), COMPARTILHADO entre os
    3 — não é alerta "quente" (a coisa simplesmente não existe/não foi
    encontrada), e precisava ser diferente do cinza claro `#94A3B8` já
    usado pra "Sem Informação" (que é FALTA DE DADO, não um achado real
    — misturar os dois confundiria "não sei" com "não existe").
  - `sinalizacao_horizontal[1]`/`drenagem_superficial[1]` (Regular,
    Obstruída) e todo `Bom`/`Adequada`/`Limpos`/`Limpa` → âmbar/verde do
    gradiente genérico (são literalmente a palavra "Regular"/"Bom" ou o
    equivalente "está tudo bem" daquele aspecto).
- **`CORES_ICM` (Resultado Geral) alinhada junto**: como Bom/Regular/
  Ruim/Péssimo do Resultado Geral são exatamente as mesmas 4 palavras
  que abrem a lista dela, virou `{Bom:#2E7D32, Regular:#FBC02D,
  Ruim:#E65100, Péssimo:#B71C1C}` — resolve de vez a inconsistência
  registrada nas entradas anteriores (Regular e Péssimo eram 2 vermelhos
  diferentes ali, sobra dos ajustes de 2026-09-15/23 feitos só nessa
  paleta separada).
- **Bug de contraste descoberto e corrigido nesta mesma rodada**: `.badge`/
  `.badge-tab` (selos na gaveta/funil/popup) tinham `color:#fff` FIXO no
  CSS. Âmbar (`#FBC02D`) e ciano (`#00ACC1`) são claros o bastante pra
  texto branco em cima ficar quase ilegível (contraste ~1.7:1, bem abaixo
  do mínimo de acessibilidade ~3:1) — só apareceu porque a paleta nova
  introduziu essas 2 cores CLARAS; toda a paleta anterior (vermelhos/
  verde escuro) era escura o bastante pra nunca ter dado pra notar.
  Função nova `corTextoContraste(hex)` (perto de `corDoSegmento`) calcula
  luminância relativa (fórmula WCAG) e escolhe branco ou `#1a1a1a`,
  o que der mais contraste — aplicada nos 5 lugares que montam um selo
  colorido (drawer, histórico do S.R.E., popup do trecho, popup de ponto
  crítico). Resolve de vez pra qualquer cor futura, não só as de hoje —
  não precisa lembrar de checar contraste manualmente da próxima vez que
  uma cor mudar.
- Testado: `getComputedStyle` confirma as 7 paletas por aspecto
  renderizando as cores certas (inclusive roxo em Vegetação e ciano em
  Sinalização Vertical, visíveis nas barras); confirmado que "Regular"
  (âmbar) e "Poucas" (ciano) saem com texto ESCURO (`rgb(26,26,26)`)
  enquanto "Bom"/"Inadequada"/"Sujo" seguem com texto branco — a função
  de contraste escolhendo certo caso a caso; conferido visualmente no
  dashboard "Todas", na gaveta de S.R.E. e no mapa (colorido por
  Resultado Geral, mostrando o âmbar novo), desktop e mobile (375×812);
  sem erro no console.

**Paleta categórica refinada pra versão definitiva, mesmo dia
(2026-09-23, 6º e último ajuste desta sequência):** a usuária mandou uma
versão mais completa e já organizada em 4 FAIXAS por ela mesma —
"Verde/Ok", "Alertas leves e médios", "Defeitos e danos", "Crítico e
ausente" — cobrindo posições que o ajuste anterior tinha preenchido só
com escolha própria (Remendo isolado/em lâmina, Sujos/Obstruída,
Inexistente/Ausente) e até redefinindo o cinza de "Sem Informação".

- **Novas cores** (substituem `CORES_SEVERIDADE` inteira):
  `COR_BOM='#2E7D32'`, `COR_REGULAR='#FBC02D'`, `COR_DEFEITO='#E53935'`
  (RUIM/INAD/DANIF, unificados numa cor só — antes eram 3 cores
  diferentes), `COR_CRITICO='#4A148C'` (só Péssima-atoleiro da
  Plataforma), `COR_AUSENTE='#424242'` (INEX/AUSENTE, antes era
  `#37474F` azulado, agora cinza neutro), `COR_SEM_INFO='#9E9E9E'`
  (era `#94A3B8` — trocado em TODOS os lugares que liam esse valor:
  `CORES_ICM`, `corDoSegmento`, `somaPorAspecto`, EXCETO
  `CORES_PONTO_CRITICO['Sem atualização']`, que é outra funcionalidade,
  não fazia parte do pedido). Pavimento ganhou 2 cores próprias novas só
  dele: `#FFB74D` (Remendo isolado) e `#FB8C00` (Remendo em lâmina) —
  antes reusavam âmbar/`COR_RUIM` antigo. Drenagem/Drenagem Superficial
  ganharam `#A1887F` (marrom-taupe) pra Sujos/Obstruída, antes eram
  marrons diferentes cada um. Buraco isolado mudou de `#FF1744` pra
  `#D81B60` (rosa-vermelho, pedido dela).
- **1 inferência não 100% explícita**: ela deu cor só pra "PÉSSIMA
  (ATOLEIRO)" (a pior classe da Plataforma), não repetiu pro "Péssimo"
  GENÉRICO do Resultado Geral (`CORES_ICM`). Aplicado `COR_CRITICO`
  (roxo) pros dois — como ela separou "defeito" (vermelho) de "crítico"
  (roxo) como 2 FAIXAS diferentes (não só 2 tons do mesmo vermelho),
  fez sentido o Péssimo genérico cair na faixa "crítico" igual à
  Péssima-atoleiro. Se ela achar que Péssimo devia ser outra cor, é só
  trocar `CORES_ICM['Péssimo']`.
- **Tabela final por aspecto** (posição 0→pior):
  - `pavimento`: Bom·Remendo isolado·Remendo em lâmina·Buraco isolado·
    Buraco em lâmina → verde·`#FFB74D`·`#FB8C00`·`#D81B60`·`#880E4F`
  - `vegetacao`: Adequada·Inadequada → verde·`COR_DEFEITO`
  - `drenagem`: Limpos·Sujos·Danificados → verde·`#A1887F`·`COR_DEFEITO`
  - `sinalizacao_horizontal`: Bom·Regular·Inexistente →
    verde·`COR_REGULAR`·`COR_AUSENTE`
  - `sinalizacao_vertical`: Bom·Poucas·Inexistente →
    verde·`#00ACC1`·`COR_AUSENTE`
  - `plataforma`: Bom·Regular·Ruim·Péssima →
    verde·`COR_REGULAR`·`COR_DEFEITO`·`COR_CRITICO`
  - `drenagem_superficial`: Limpa·Obstruída·Ausente →
    verde·`#A1887F`·`COR_AUSENTE`
- Testado: `getComputedStyle` confere as 7 paletas (`rgb(255,183,77)`,
  `rgb(251,140,0)`, `rgb(216,27,96)` no Pavimento; `rgb(74,20,140)` roxo
  em Plataforma; `rgb(158,158,158)` no "Sem Informação" novo, em vez do
  cinza-azulado antigo) e o texto certo (claro/escuro) nos selos —
  inclusive "INAD." em cima do novo vermelho `#E53935` saiu com texto
  ESCURO (a função calculou mais contraste assim: ~4.9:1 contra ~4.3:1
  no branco — resultado correto, ainda que menos usual que "vermelho com
  texto branco"); conferido visualmente no dashboard "Todas", gaveta de
  S.R.E. e mapa, desktop e mobile (375×812); sem erro no console.

Aparece em 2 lugares (⚠️ **desatualizado até 2026-09-10**: esta seção ainda
citava `#regiao-barra`/`#regiao-legenda`/`ativosAspectoRegiao` como se
existissem — mas esses foram removidos DE VEZ em 2026-09-03, ver "Resultado
Geral da região removida DE VEZ" mais acima. Corrigido aqui pra não repetir
a confusão em sessão futura):
- **Mapa inteiro** (Todas ou qualquer região): sempre colorido por `icm` — o
  seletor "Colorir mapa por" que trocava isso existiu por um dia (2026-09-02/03)
  e foi removido a pedido da usuária, ver seção acima. Não tem mais nenhuma
  barra/legenda com checkbox filtrando por classe do Resultado Geral em
  lugar nenhum (nem "Todas" nem região específica) — quem quer o detalhe por
  classe usa "Por aspecto avaliado" (lista de leitura, sem filtro de mapa).
- Clicar em qualquer trecho abre um popup com "Resultado geral" em destaque + o
  detalhe dos grupos que existem naquele segmento + tag "Pavimentada"/"Não
  pavimentada".

## Camadas de contexto (mapa)

Adicionadas em 2026-08-27, todas com dados gerados por funções em
`converter_fichas.py` (chamadas no `main()`) que escrevem `dados/*.js` própios:

| Camada | Fonte | Liga por padrão? | Gerado por |
|---|---|---|---|
| Malha viária (cinza, fundo) | `camadas/Base_Rods_2023.shp`, simplificado (~50m) | Sim, sempre | `gerar_malha_contexto()` → `dados/malha_contexto.js` |
| Limites municipais | `camadas/LimiteMunicipal_AGM_TO_2022_A.shp` (139 municípios TO, fonte AGM/2022) | Não (checkbox) | `gerar_limites_municipais()` → `dados/limites_municipais.js` |
| Pontos Críticos | `camadas/R<n>_Pontos_Criticos.shp` (geometria) + planilha "Controle Pontos Críticos" (status/descrição/link) | Não (checkbox) | `gerar_pontos_criticos()` → `dados/pontos_criticos.js` |
| Satélite (basemap) | Esri `World_Imagery` | Não (troca com "Padrão" no controle de camadas) | — |

**Pontos Críticos** é o mais elaborado: lê a planilha **`C:\1. Projetos\RTA\web\pontos
criticos\Controle Pontos Críticos .xlsx`** (do OUTRO projeto, `../../web/` a partir de
`ficha-inspecao/` — caminho absoluto em `PLANILHA_PONTOS_CRITICOS`; lida direto de lá
pra sempre pegar a versão mais atual, nunca copiada pra cá). Uma aba por região
("REGIÃO 1"..."REGIÃO 13"), colunas de mês variam MUITO entre regiões (nome, se tem
ano junto tipo "Novembro/2025", se tem espaço em "Mapas de Out"/"MapaAbril") —
`_mes_abrev_de()` casa tudo pela abreviação de 3 letras, ignorando o resto do texto.
Cada mês tem uma coluna de status ("Crítico"/"Em execução"/"Recuperado"/"-") e,
depois da coluna "Status Final / Situação", uma coluna "Mapa de \<mês\>" com um
**hyperlink pro Google Drive** (ficha do mês em PDF, privada) — vira o botão "Ver
mapa" no popup (nunca embutir a imagem, é privada e não é URL direta de imagem —
mesmo comportamento do geoportal principal, `gerar_mapa.py`). Ponto com status mais
recente = "Recuperado"/similar **não entra na camada** (`_classificar_situacao`
'Resolvido' → `continue` em `gerar_pontos_criticos()`) — a usuária só quer ver o que
ainda precisa de atenção.

Tentativa revertida: embutir foto de campo real (`fotos-pontos-criticos/`, copiada
do projeto principal, só existe pra algumas regiões) direto no popup como `<img>` —
a usuária pediu pra tirar ("deixa sem as fotos, apenas com o link"). Os arquivos de
foto continuam no repo (não fazem mal), só não são mais referenciados pelo código.

`camadas/R<n>_Pontos_Criticos.shp`, `LimiteMunicipal_AGM_TO_2022_A.shp` e
`Base_Rods_2023.shp` — igual aos `R<n>_TRECHOS.shp`, são cópias de fora deste repo
(a primeira do projeto principal `../web/camadas/`, a segunda de
`MAPAS OSP/SHAPEFILES UTEIS/` — ver "Relação com outros projetos" no fim deste
arquivo). Se precisar atualizar, copiar de novo de lá e rodar o conversor.

## Funil Região → Competência → Tipo de via → Trecho → S.R.E.

Segundo jeito de navegar, pra achar um trecho específico (ou ver tudo de uma vez):
Região (desenha o contorno tracejado — `camadas/R<n>_REGIÃO.shp` → `dados/regioes.js`
— e dá fit nele) → Competência → Tipo de via (só os tipos que existem naquele
região+mês) → **Trecho** (agrupa vários S.R.E. sob o mesmo lote/rodovia — equivale à
coluna `Id` do shapefile `R<n>_TRECHOS.shp`, bate com o número da aba da ficha) →
S.R.E. Tanto Trecho quanto S.R.E. têm uma opção **"Todos"** (`TODOS = '__todos__'`
no JS) — escolher "Todos os trechos" já mostra tudo direto, sem precisar escolher o
S.R.E. depois; escolher um trecho específico ainda oferece "Todos os S.R.E. deste
trecho". `featuresDoFunil()` centraliza esse filtro (tipo é obrigatório; trecho/sre em
"Todos" viram no-op).

Ao escolher (S.R.E. ou "Todos"), o mapa dá zoom com destaque (casing branco + cor pelo
aspecto principal — `pavimento` pra via pavimentada, `plataforma` pra não pavimentada)
e abre uma gaveta embaixo do mapa com uma tabela de **todos** os sub-trechos de km e
as condições marcadas em cada um (só as colunas do modelo de ficha correspondente,
mais a coluna "Resultado Geral (I.C.M.)"). Quando mostra mais de um S.R.E. de uma vez
("Todos"), a tabela ganha colunas extras de Trecho e S.R.E. no início.

## Fluxo de trabalho — ficha nova chegou

Todo mês chegam **até 12 arquivos** (pavimentada + não pavimentada × 6 regiões), mas não
precisa esperar todos — o converter processa o que tiver em `fichas/` e funde por
região+competência (dá pra ir soltando os arquivos conforme chegam e rodar de novo).

1. Salvar o(s) `.xlsx` em `ficha-inspecao/fichas/` (nome livre, mas o padrão até
   agora é `ficha de inspeção_rodovias pavimentadas_R.<região> - <MÊS>.xlsx` e
   `ficha de inspeção_rodovias não pavimentadas _R.<região> - <MÊS>.xlsx`)
2. Rodar `python converter_fichas.py` de dentro de `ficha-inspecao/`
3. Checar `ficha-inspecao/relatorio_qualidade.txt` — S.R.E. não encontrado, geometria
   com vão grande, extensão inspecionada muito diferente da extensão do shapefile
4. `git add -A && git commit && git push`

Região e competência (mês/ano) são lidos de **dentro da planilha** (célula "REGIÃO:" e
célula "DATA:" de cada aba de trecho), não do nome do arquivo — então o nome do arquivo
pode variar sem quebrar nada.

## Testar local

Servidor `python -m http.server 8768 --directory .` na RAIZ do projeto — há config
`inspecao-pavimento` no `.claude/launch.json` do projeto `web` vizinho
(`C:\1. Projetos\RTA\web\.claude\launch.json`). URL:
`http://localhost:8768/ficha-inspecao/index.html`.

## Publicar (já feito — ver abaixo se precisar refazer/entender)

Publicado em 2026-08-27: `github.com/fernanda1997-pj/ficha-inspe-o` →
`ficha-inspe-o.vercel.app`, deploy automático a cada `git push` na `main`.

**`vercel.json` (raiz do repo) é essencial — não apagar.** O `index.html` fica em
`ficha-inspecao/`, não na raiz do repo, e o Root Directory do projeto no Vercel
ficou no padrão (raiz) — sem o rewrite, o link raiz do site dá 404. Ele redireciona
`/` → `/ficha-inspecao/index.html` e `/dados/*` → `/ficha-inspecao/dados/*` (as duas
únicas referências relativas que "saem" da pasta; `../logo/*` já resolve sozinho
porque o navegador clampa `..` na raiz).

Passos originais (mesmo fluxo do [[geoportal-levantamento]] em `web - Mapas`), pra
publicar um projeto novo do zero:

1. Criar um repositório **novo e vazio** no GitHub (a usuária faz isso pela UI —
   sessões de Claude Code não têm `gh` CLI nem token configurado aqui)
2. `git remote add origin <url>` e `git push -u origin main`
3. Importar o repo no Vercel (vercel.com → Add New Project → escolher o repo) — deploy
   automático a cada push na `main`, igual aos outros dois projetos

### Basemap "Padrão" — histórico de tentativas (não repetir sem checar antes)

CARTO (`basemaps.cartocdn.com/rastertiles/voyager`) passou a exigir API key em
produção (marca d'água "API KEY REQUIRED"). Tentativas seguintes, na ordem:
Esri `World_Street_Map` (carregado/colorido demais) → Esri `Canvas/World_Light_Gray_Base`
+ `World_Light_Gray_Reference` juntas (2 camadas sobrecarregaram o carregamento
inicial — tiles levando 10s+ cada, provável limite de rajada) → OpenStreetMap padrão
(mostra rio/estrada de terra/contorno de propriedade, poluído demais em cima dos
trechos coloridos) → **atual: só `Canvas/World_Light_Gray_Base`, uma única camada**
(sem a `_Reference`) — visual limpo, sem chave, rápido. Se precisar trocar nesse
mesmo problema, testar tempo de resposta de verdade antes (a Esri variou de
"instantâneo" a "10s+" pro mesmo endpoint em momentos diferentes — provável
throttling de rajada, não indisponibilidade permanente).

`map` usa `preferCanvas:true` (Leaflet) — Visão Geral soma milhares de trechos de
uma vez (+ a malha viária de contexto), SVG individual por feição pesava demais.

## Histórico de tentativas de ligar ficha × O.S. (não repetir sem pedido explícito)

Quando `ordens-servico/` ainda vivia dentro deste repo (antes de virar
`C:\1. Projetos\RTA\web - OS`), a usuária pediu pra ligar as duas fontes de dados
mais de uma vez e desistiu toda vez:
1. Mostrar O.S. dentro do popup/gaveta da ficha (bloco stacked) — achou confuso os
   dois conteúdos juntos na mesma gaveta pequena.
2. Link cruzado (botão "N O.S.P. neste trecho → ver", chip de filtro, selo
   "📋 ficha: classe" na lista de O.S., histórico de competências da ficha na gaveta
   da O.S.) — ainda achou confuso / gerava dúvida tipo "por que O.S. concluída e
   ficha negativa" (resposta: ficha é mensal, O.S. não tem periodicidade fixa, então
   uma piora depois do "Concluída" é desgaste normal, não erro — mas mesmo com essa
   explicação preferiu não ligar).
3. Duas páginas HTML na mesma pasta, depois duas PASTAS separadas, e por fim
   **repositórios separados** — cada nível de separação foi pedido explicitamente.
**Se um dia pedir de novo pra ligar as duas**, agora é cross-repo (não dá mais pra
só referenciar um `window.DADOS_*` do outro app — teria que ser via API/arquivo
publicado) — vale confirmar bem o que ela quer antes de reimplementar algo do
histórico acima.

## Relação com outros projetos

- `C:\1. Projetos\RTA\web` = geoportal principal (Folium), site
  `rta-msi-rodovias.vercel.app`. É de lá que vêm os shapefiles `R*_TRECHOS.shp`
  (copiados pra `camadas/` aqui).
- `C:\1. Projetos\RTA\web - Mapas` = Geoportal RTA-MSI — Levantamento de Trechos
  (`mapa-levantamento.vercel.app`). Repo/site diferente, mesma família de produtos.
- `C:\1. Projetos\RTA\web - OS` = Geoportal RTA-MSI — Ordens de Serviço. Repo/site
  próprio desde 2026-08-18 (era `ordens-servico/` aqui dentro) — ver `CLAUDE.md` de
  lá.
- Este projeto (`web - fichas`) é o quarto da família: **inspeção de campo do
  pavimento**, repo próprio.
