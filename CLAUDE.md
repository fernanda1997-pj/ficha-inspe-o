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

**`resultadoGeral` virou padrão DESLIGADO (2026-09-03, minutos depois):** a
usuária mandou o MESMO print de novo (a barra/legenda do Resultado Geral
dentro de uma região) só que agora "PODE TIRAR ISSO" em caixa alta — não
queria só a opção de desligar, queria que já viesse desligado (ela pode
religar em "O que mostrar" se quiser; o controle continua existindo, só o
padrão mudou). `localStorage` é por navegador/aparelho — então mudar o
padrão em código é o que garante que ISSO comece escondido em qualquer
lugar que ela abra o site, não só onde ela já tinha desmarcado manualmente
antes.

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
`#0ca30c`, Regular `#fab219`, Ruim `#ec835a`, Péssimo `#d03b3b`, Sem Informação
`#94A3B8` (`CORES_ICM` no `index.html`).

Aparece em 3 lugares:
- **Mapa inteiro** (Todas ou qualquer região): sempre colorido por `icm` — o
  seletor "Colorir mapa por" que trocava isso existiu por um dia (2026-09-02/03)
  e foi removido a pedido da usuária, ver seção acima.
- **Barra + legenda com checkbox**: só numa região específica agora
  (`#regiao-barra`/`#regiao-legenda`, escopo = o que estiver selecionado no
  funil Tipo/Trecho/S.R.E., ou a região inteira se nada escolhido —
  `atualizarResumoRegiao()`). A legenda **dobra de filtro do mapa** —
  desmarcar uma classe some com ela do gráfico e do mapa ao mesmo tempo
  (`ativosAspectoRegiao`). Em "Todas" isso não existe mais desde
  2026-09-03 — virou a lista "Por aspecto avaliado" (sem checkbox, sem
  filtro de mapa, ver seção acima); quem quer filtrar o mapa por classe do
  Resultado Geral abre uma região.
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
