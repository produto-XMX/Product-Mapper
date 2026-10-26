# CLAUDE.md

Este arquivo fornece orientações para o Claude Code (claude.ai/code) ao trabalhar com código neste repositório.

## O que é isto

Uma ferramenta HTML autocontida (`index.html`) — "Gerador de Documento de Funil" — para funis de venda baseados na BuyGoods. Não tem backend, não tem etapa de build e não tem dependências: tudo (HTML, CSS, JS) vive em um único arquivo e roda inteiramente no navegador. Não há `package.json`, test runner ou linter neste repositório.

A ferramenta permite que o usuário descreva um ou mais funis de venda (página de venda front + uma sequência de etapas de upsell/downsell) e gera um documento em texto puro — com scripts de tracking, scripts de conversão e links de checkout para cada etapa — pronto para copiar ou baixar e repassar a outra equipe.

## Executando / verificando alterações

Não há servidor de desenvolvimento nem etapa de build. Para testar mudanças, basta abrir o `index.html` no navegador (ou usar um workflow de "run", se houver um configurado). Depois de editar o `<script>` inline, use a skill `verify-inline-script` para checar a sintaxe do JS.

## Arquitetura

Tudo vive em uma única IIFE no final do `index.html`. O modelo de dados é uma árvore, mantida em memória e espelhada no `localStorage` (ver "Persistência", abaixo):

```
documentAccount                    — account_id (a=) único para TODO o documento — front e todos os funis usam este mesmo valor
frontGroups[]                      — nível superior, compartilhado por TODOS os funis (não pertence a nenhuma aba específica)
  .token                           — token de conversão deste grupo front (independente de qualquer funil)
  .urls[]                          — { url } — todas as URLs desse grupo recebem o mesmo tracking/conversão (e o disclaimer único do documento, ver abaixo)
  .variants[]                      — { label, codname, nextUrl } — produtos vendidos nas URLs desse grupo; cada produto tem seu próprio redirect de checkout (produtos diferentes do mesmo grupo/página podem levar a funis diferentes, ex.: comprar "6 Bottles" entra num funil, "2/3 Bottles" entra em outro)

funnels[]                          — lista de nível superior, alternável pelas abas ("Funis")
  .name, .trigger                  — trigger = qual compra no front direciona para este funil (texto livre, apenas documentação — a conexão real é o .nextUrl de cada variante do front, acima)
  .token                           — token de conversão da BuyGoods usado nas ETAPAS deste funil (não no front)
  .steps[]                         — sequência de upsell/downsell, em ordem
    .pageUrl                       — URL da própria página desta etapa (opcional; sobrescreve a inferência a partir do declineUrl da etapa anterior — obrigatório preencher na primeira etapa, que não tem etapa anterior)
    .name, .nextUrl, .declineUrl   — destino do redirect de aceite (compartilhado por todas as variantes da etapa) e link de recusa da própria etapa
    .variants[]                    — { label, codname } — ex: ofertas "3/6/9 Bottles" naquela etapa
```

O Front é intencionalmente desacoplado de qualquer funil: como um mesmo produto/página front costuma alimentar vários funis diferentes (dependendo de qual variante o cliente compra), duplicar o front em cada aba geraria retrabalho e risco de divergência. Por isso `frontGroups` vive fora de `funnels[]`. O `account_id` (`documentAccount`) é único para o documento inteiro — na prática é sempre a mesma conta BuyGoods em todo lugar, então não há motivo pra repetir o campo em cada grupo front e em cada funil. Já o `token` (`t=`) varia de verdade: cada grupo front e cada funil rastreiam eventos de conversão diferentes no BuyGoods, então cada um mantém o seu próprio, nunca herdado de outro escopo.

Não há nada de BuyGoods hardcoded referente a URLs, codnames de produto ou Account ID: tudo isso é dado inserido pelo usuário nos grupos front e nas etapas. As únicas partes fixas são os templates dos scripts BuyGoods em si (ver abaixo). Páginas front (`buildFrontConversionScript`) nunca recebem `&conversion=1`/`&o=`/`&ut=` no iframe de conversão e sempre envolvem a chamada em `setTimeout(..., 1000)`; etapas de upsell/downsell (`buildUpsellConversionScript`) sempre recebem `&conversion=1`/`&o=`/`&ut=` e nunca usam `setTimeout` — essa distinção depende do tipo de bloco (front vs. etapa), nunca do texto da URL. O disclaimer aparece **uma única vez** no documento (não por grupo front), logo depois do título "Página(s) de Venda (Front)" — já que seu conteúdo depende só do `account_id`, que agora é compartilhado por todo o documento, repeti-lo por grupo seria puramente redundante; a nota no bloco já instrui a equipe a colar em todas as páginas front listadas. Cada grupo front também gera seus próprios links de checkout (`/secure/checkout.html`, um por produto do grupo), com `redirect=` calculado a partir do `.nextUrl` de cada **variante** (não do grupo) — isso é intencional: produtos diferentes na mesma página front costumam alimentar funis de upsell diferentes. Já as etapas geram os seus links (`/secure/upsell/`) a partir do `.nextUrl` da **etapa** (compartilhado por todas as variantes daquela etapa) — não confundir os dois níveis ao editar `buildFrontBlocks`/`buildFunnelBlocks`.

Apenas um funil está "ativo" por vez (`activeFunnelId`); os cards "Dados do funil selecionado" e "Etapas deste funil" sempre editam o funil ativo, e trocar de aba re-renderiza esses cards a partir dos dados do funil recém-ativado — não existe um DOM separado por funil. Já o card "Páginas de venda (Front)" é renderizado uma única vez a partir de `frontGroups` e não é afetado por trocar de aba.

Todos os campos `id` (`funnelIdCounter`, `stepIdCounter`, `variantIdCounter`, `frontGroupIdCounter`, `frontUrlIdCounter`) são contadores monotônicos independentes usados como chaves de atributos `data-*` no DOM para delegação de eventos — existe exatamente um listener de `input`/`click` por container (`funnelTabsEl`, `stepsContainer`, `frontGroupsContainer`), não um listener por linha. Qualquer nova linha editável deve seguir esse padrão: renderizar com atributos `data-field`/`data-action`/`data-*-id` e tratar no listener delegado já existente do container.

**Fluxo de renderização:** toda mutação chama a função de render mais específica necessária, e sempre termina em `renderDoc()`:
- Editar um valor no lugar → apenas `renderDoc()`.
- Adicionar/remover uma linha (etapa, variante, funil, grupo front, URL front) → `renderStepsDOM()` / `renderFrontGroupsDOM()` / `renderFunnelTabs()` (reconstrói o HTML daquele container a partir do estado) → depois `renderDoc()`.
- Trocar o funil ativo → `renderFunnelTabs()` + `renderActiveFunnelForm()` + `renderStepsDOM()` + `renderDoc()` (não chama `renderFrontGroupsDOM()` — o front não depende do funil ativo).

**A geração do documento é baseada em blocos, não em string/markdown.** `buildFrontBlocks()` (uma vez, para todos os grupos front) e `buildFunnelBlocks(funnel, index)` (uma vez por funil, só as etapas) transformam o estado em um array de blocos tipados (`title`, `heading`, `label`, `text`, `list`, `code`, `divider`); `renderDoc()` concatena `buildFrontBlocks()` seguido de um `buildFunnelBlocks()` por funil, nessa ordem. Essa fonte única de verdade é então renderizada de duas formas:
- `blocksToPlainText(blocks)` → o conteúdo literal do `.txt` (títulos/cabeçalhos em maiúsculas, sem símbolos de markdown) — é isso que é copiado e baixado. Evita `#`/`**`/`` ``` `` de propósito, já que o resultado é colado em ferramentas como o Google Docs, que não interpretam Markdown e mostrariam os símbolos literalmente.
- `renderDocPreview(blocks)` → HTML estilizado exibido em tempo real no painel `#docPreview`, incluindo um botão "Copiar" por bloco de código (cada script pode ser copiado independentemente, já que uma página normalmente precisa do script de tracking e do script de conversão colados em dois lugares diferentes).

Ao alterar a estrutura ou o texto do documento, edite `buildFrontBlocks`/`buildFunnelBlocks` — nunca ajuste manualmente a formatação de string em `blocksToPlainText`/`renderDocPreview` para corrigir conteúdo, já que ambos são derivados do mesmo array de blocos.

Blocos `heading`/`title` aceitam um `meta` opcional: um sufixo " — texto" que NUNCA é maiúsculizado (ao contrário do `text` do bloco), usado para anexar URLs sem alterar seu case. O heading de cada etapa usa isso para mostrar a URL real daquela página, com prioridade: `step.pageUrl` (se o usuário preencheu explicitamente) → senão o `declineUrl` da etapa ANTERIOR (a página de uma etapa costuma ser alcançada pela recusa da etapa antes dela, não pelo `nextUrl` da própria etapa, que aponta para a PRÓXIMA página) → senão sem `meta` (só ocorre na primeira etapa quando `pageUrl` está vazio, já que ela não tem etapa anterior de onde inferir).

**Quatro templates de script da BuyGoods estão hardcoded**: `buildScript()` (pixel de tracking, indexado por `account` + `codname`s unidos por vírgula — o único template compartilhado entre front e etapas, pois só o `product=` varia), `buildFrontConversionScript()` e `buildUpsellConversionScript()` (dois templates de iframe de conversão *distintos e não intercambiáveis*, cada um indexado por `account` + `token` — ver acima a diferença de `setTimeout`/`conversion=1`/`o=`/`ut=`) e `buildDisclaimerScript()` (snippet de disclaimer, indexado só por `account`). Todos são snippets por página extraídos do próprio embed code da BuyGoods — trate o corpo dessas strings como opaco/exato e altere apenas os valores interpolados `acc`/`product`/`tok`.

**Fonte de verdade para copiar/baixar:** `#docOut` é um `<textarea>` real mantido no DOM mas visualmente oculto (`.visually-hidden`, não `display:none`) para que `.select()` continue funcionando no fallback via `execCommand('copy')`. `#docPreview` é somente apresentação — a lógica de clipboard e download de arquivo sempre lê de `docOut.value`, nunca faz scraping do HTML do preview.

**Persistência:** `renderDoc()` termina chamando `persistState()`, que grava `serializeState()` no `localStorage` (chave `funnel-doc-generator:v1`) — como toda mutação já termina em `renderDoc()`, esse é o único ponto de autosave; não adicione chamadas de save espalhadas pelos listeners. O mesmo JSON é o formato do arquivo "Salvar projeto (.json)" / "Abrir projeto". O JSON não guarda os `id`s (são só chaves de DOM) e guarda o funil ativo como índice (`activeFunnelIndex`); `applyState()` reconstrói tudo pelas fábricas `newX()` via `hydrateX()`, tolerando campos ausentes, e depois `renderAll()` re-renderiza todos os containers. Ao adicionar um campo novo ao modelo, inclua-o em `serializeState()` e no `hydrateX()` correspondente.
