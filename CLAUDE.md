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

**Seletor "Colorir mapa por" (2026-09-02):** a usuária reclamou que o Resultado Geral
(média de todos os aspectos) escondia detalhe importante — "os dados ficavam muito
vagos". Substituiu-se por um `<select id="sel-aspecto">` (populado a partir de
`GRUPOS` = `GRUPOS_INSPECAO` + `{id:'icm', nome:'Resultado Geral'}`) compartilhado
pelas duas abas: ele recolore o mapa INTEIRO por um aspecto de cada vez (nunca duas
camadas ao mesmo tempo, sem offset). Trocar o select muda `aspectoAtual`, reseta os
filtros de checkbox (`ativosAspectoGeral`/`ativosAspectoRegiao` — os níveis de um
aspecto não têm relação com os de outro) e redesenha Visão Geral + Por Região. A
generalização do que antes só existia pro I.C.M.: `classeDoAspecto(props, grupoId)` /
`somaPorAspecto(feats, grupoId)` (paralelo a `somaIcmDe`, mas descobre os níveis a
partir dos dados de verdade em vez de uma lista fixa tipo `CLASSES_ICM`) e
`ordemClasses` (`[{chave,nome,cor}]`) como formato comum que alimenta donut, legenda
e filtro tanto pro I.C.M. quanto pra qualquer aspecto. Trecho que não tem aquele
aspecto (ex.: `vegetacao` numa via não pavimentada) cai em `'sem_info'`, cinza
`#94A3B8`. `montarComparativoRegioes` (a barra "Comparativo por região" na Visão
Geral) ficou de propósito só no Resultado Geral — não segue o seletor.

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
- **Mapa das duas abas**: sempre colorido por `icm` (não tem seletor de aspecto).
- **Donut + legenda com checkbox**: em "Por Região" (escopo = o que estiver
  selecionado no funil Tipo/Trecho/S.R.E., ou a região inteira se nada escolhido —
  `atualizarResumoRegiao()`) e em "Visão Geral" (escopo = tudo que já foi convertido
  — carrega os `insp_*.js` que faltarem via `carregarTodosOsDados()`). A legenda
  **dobra de filtro do mapa** — desmarcar uma classe some com ela do gráfico e do
  mapa ao mesmo tempo (`ativosIcmRegiao`/`ativosIcm`); não existe mais uma seção
  separada de "Mostrar no mapa", foi unificada com a legenda (2026-08-26).
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
