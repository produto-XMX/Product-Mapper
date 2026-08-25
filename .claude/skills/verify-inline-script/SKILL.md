---
name: verify-inline-script
description: Verifica a sintaxe do <script> inline do index.html após uma edição, extraindo pelos limites da IIFE em vez de pela tag <script>/<\/script> (o arquivo contém uma string literal <script>...<\/script> que confundiria uma extração naive). Use sempre depois de editar o bloco <script> do index.html.
---

Não há servidor de desenvolvimento nem etapa de build neste repositório. Para testar mudanças, basta abrir o `index.html` no navegador (ou usar um workflow de "run", se houver um configurado).

Como o bloco `<script>` da página contém uma string literal `<script>...<\/script>` (o snippet de tracking da BuyGoods embutido como string JS), ferramentas ingênuas que buscam por tags `<script>`/`</script>` para extrair o script real acabam pegando o trecho errado. Para verificar a sintaxe do JS depois de uma edição, extraia pelos limites da IIFE em vez de pela tag, normalizando as quebras de linha antes (o arquivo pode estar em CRLF):

```bash
node -e "
const fs = require('fs');
let html = fs.readFileSync('index.html', 'utf8').replace(/\r\n/g, '\n');
const start = html.indexOf('(function () {') + '(function () {'.length;
const end = html.lastIndexOf('})();\n</script>');
fs.writeFileSync('check.js', html.slice(start, end));
"
node --check check.js && rm check.js
```

Execute isso depois de qualquer edição no `<script>` inline para detectar erros de sintaxe antes de entregar o arquivo.