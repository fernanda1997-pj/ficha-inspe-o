# Geoportal RTA-MSI — Inspeção do Pavimento

WebGIS de página única (Leaflet) que mostra no mapa o resultado das fichas mensais de
inspeção de rodovias pavimentadas do Tocantins. Usuária: Fernanda (RTA Engenheiros
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

## Os 5 grupos de condição

Cada linha da ficha pode ter mais de uma marcação (X) dentro do mesmo grupo (ex.: um km
com "Remendo em lâmina" **e** "Buraco em lâmina" ao mesmo tempo). Pra cor no mapa, vale a
regra **pior marcação vence**: a severidade é a posição da coluna dentro do grupo
(0 = melhor, a ficha sempre desenha da esquerda/melhor pra direita/pior) — por isso o
converter não depende do texto exato do rótulo (tem, inclusive, um erro de digitação na
ficha original: "INADED." em vez de "INADEQ.").

| Grupo (`id`) | Severidades (0→pior) |
|---|---|
| `pavimento` | Bom · Remendo isolado · Remendo em lâmina · Buraco isolado · Buraco em lâmina |
| `vegetacao` | Adequada · Inadequada |
| `drenagem` | Limpos · Sujos · Danificados |
| `sinalizacao_horizontal` | Bom · Regular · Inexistente |
| `sinalizacao_vertical` | Bom · Poucas · Inexistente |

No mapa, cada grupo é uma camada independente (checkbox próprio). Com mais de uma
marcada ao mesmo tempo, as linhas aparecem deslocadas ~6 m uma da outra (via
`turf.lineOffset`) pra dar pra comparar lado a lado — só fica visível de perto (zoom de
rua), o que é esperado. **Cuidado**: `turf.lineOffset` pode devolver coordenadas `NaN`
sem lançar exceção para linhas muito curtas/degeneradas — por isso `offsetMetros()` no
`index.html` valida cada coordenada e volta pra geometria original se der `NaN` (um
único ponto inválido derruba a camada inteira no Leaflet, silenciosamente, se não tratar
isso).

Clicar em qualquer trecho (não importa qual camada está colorindo) abre um popup com o
detalhe completo dos 5 grupos daquele km.

## Fluxo de trabalho — ficha nova chegou

1. Salvar o `.xlsx` em `fichas/` (nome livre, mas o padrão até agora é
   `ficha de inspeção_rodovias pavimentadas_R.<região> - <MÊS>.xlsx`)
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
