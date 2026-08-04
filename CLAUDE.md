# Geoportal RTA-MSI — Inspeção do Pavimento

WebGIS de página única (Leaflet) que mostra no mapa o resultado das fichas mensais de
inspeção de rodovias do Tocantins — **pavimentadas e não pavimentadas (LEN)**. Todo mês
saem **2 fichas por região × 6 regiões** (R1, R2, R3, R11, R12, R13): uma de trechos
pavimentados, outra de não pavimentados. Usuária: Fernanda (RTA Engenheiros
Consultores). Responder sempre em português.

- **Site**: (ainda não publicado — ver seção "Publicar" abaixo)
- **Repo**: (ainda não publicado)

## Arquitetura

| Arquivo/pasta | Papel |
|---|---|
| `index.html` | O app inteiro (HTML+CSS+JS, sem build). CDN: Leaflet 1.9.4, Turf.js 6 (só usado pra `lineOffset`, o deslocamento visual quando >1 camada está ligada) |
| `converter_fichas.py` | Lê `fichas/*.xlsx` + `camadas/R*_TRECHOS.shp`, corta a geometria de cada S.R.E. no km início/fim de cada linha da ficha (referenciamento linear) e escreve `dados/` |
| `fichas/` | Fichas de inspeção mensais, uma por região+mês, como chegam do campo (`ficha de inspeção_rodovias pavimentadas_R.<região> - <MÊS>.xlsx`) — **não editar**, só adicionar arquivos novos aqui |
| `camadas/` | Cópia de `R<região>_TRECHOS.shp` (uma linha por S.R.E., com `EXT_REAL` em km) — vem do geoportal principal em `../web/camadas/`. Se um S.R.E. novo aparecer numa ficha e não achar o shapefile, é só copiar a versão atualizada de lá |
| `dados/insp_<REGIAO>_<AAAA-MM>.js` | Um GeoJSON (dentro de `window.DADOS_INSPECAO[regiao][competencia]`) por região+competência, gerado pelo converter — **não editar à mão** |
| `dados/manifest.js` | Lista de todas as combinações região/competência disponíveis (`window.MANIFEST_INSPECAO`) + os 5 grupos de condição (`window.GRUPOS_INSPECAO`) — o `index.html` usa isso pra montar os selects e injetar os `<script>` dos arquivos `insp_*.js` sob demanda |
| `logo/` | Logos RTA + MSI (copiados de `web - Mapas/logo/`) |
| `relatorio_qualidade.txt` | Gerado a cada rodada do converter (gitignored) — aponta S.R.E. da ficha que não bateu com o shapefile, geometrias em partes desconexas, extensão inspecionada muito diferente da extensão real etc. |

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
dados): o converter concatena as partes na ordem em que aparecem no shapefile (não usa
`linemerge`, que reordena sem critério), invertendo cada parte se precisar manter a
continuidade com a anterior. Isso preserva o comprimento total (bate com `EXT_REAL`) em
vez de descartar pedaços. Se o vão entre duas partes for grande (>200 m), o
`relatorio_qualidade.txt` avisa com "CONFERIR o shapefile" — pode ser um pedaço do
traçado que falta digitalizar.

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
com exemplo — pediu pra tirar (2026-08-04). **Não recriar sem pedido explícito.** O
mapa da região hoje é sempre colorido pelo Resultado Geral (I.C.M.) só; quem quer o
detalhe por aspecto usa o popup (clique no trecho) ou a tabela do funil (que já mostra
todas as colunas de uma vez). `turf.js` foi removido do projeto (só existia pro offset).

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

Guardado em `properties.icm = {classe, valor}` de cada feature. Cores fixas (paleta de
status, não a escala verde→vinho de severidade): Bom `#0ca30c`, Regular `#fab219`, Ruim
`#ec835a`, Péssimo `#d03b3b`, Sem Informação `#94A3B8` (`CORES_ICM` no `index.html`).

Aparece em 3 lugares:
- **Mapa da aba "Por Região"**: sempre colorido por `icm` (não tem seletor de aspecto).
- **Rosca (donut) + KPI de extensão**: em "Por Região" (escopo região+competência
  selecionadas) e na aba "Visão Geral" (escopo tudo que já foi convertido — carrega os
  `insp_*.js` que faltarem via `carregarTodosOsDados()`). Mesmas funções
  (`somaIcmDe`/`montarDonut`/`montarLegendaIcm`), só o conjunto de features muda.
- **"Visão Geral"** tem também um filtro "Mostrar no mapa" por classe (Bom/Regular/
  Ruim/Péssimo/Sem Informação) — independente do gráfico, que sempre mostra o total.

Clicar em qualquer trecho abre um popup com "Resultado geral" em destaque + o detalhe
dos grupos que existem naquele segmento + tag "Pavimentada"/"Não pavimentada".

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

1. Salvar o(s) `.xlsx` em `fichas/` (nome livre, mas o padrão até agora é
   `ficha de inspeção_rodovias pavimentadas_R.<região> - <MÊS>.xlsx` e
   `ficha de inspeção_rodovias não pavimentadas _R.<região> - <MÊS>.xlsx`)
2. Rodar `python converter_fichas.py`
3. Checar `relatorio_qualidade.txt` — S.R.E. não encontrado, geometria com vão grande,
   extensão inspecionada muito diferente da extensão do shapefile
4. `git add -A && git commit && git push`

Região e competência (mês/ano) são lidos de **dentro da planilha** (célula "REGIÃO:" e
célula "DATA:" de cada aba de trecho), não do nome do arquivo — então o nome do arquivo
pode variar sem quebrar nada.

## Testar local

Servidor `python -m http.server 8768 --directory .` — há config `inspecao-pavimento`
no `.claude/launch.json` do projeto `web` vizinho (`C:\1. Projetos\RTA\web\.claude\launch.json`).

## Publicar

Ainda não publicado. Passos (mesmo fluxo do [[geoportal-levantamento]] em `web - Mapas`):

1. Criar um repositório **novo e vazio** no GitHub (a usuária faz isso pela UI —
   sessões de Claude Code não têm `gh` CLI nem token configurado aqui)
2. `git remote add origin <url>` e `git push -u origin main`
3. Importar o repo no Vercel (vercel.com → Add New Project → escolher o repo) — deploy
   automático a cada push na `main`, igual aos outros dois projetos

## Relação com outros projetos

- `C:\1. Projetos\RTA\web` = geoportal principal (Folium), site
  `rta-msi-rodovias.vercel.app`. É de lá que vêm os shapefiles `R*_TRECHOS.shp`
  (copiados pra `camadas/` aqui).
- `C:\1. Projetos\RTA\web - Mapas` = Geoportal RTA-MSI — Levantamento de Trechos
  (`mapa-levantamento.vercel.app`). Repo/site diferente, mesma família de produtos.
- Este projeto (`web - fichas`) é o terceiro da família: **inspeção de campo do
  pavimento**, separado dos outros dois por pedido da usuária (repo próprio).
