# citado-indexer

> Motor de auto-indexação para sites na borda (Cloudflare Workers): o site se mantém **indexado e atualizado no Google, no Bing — e nas IAs que consultam esses índices — sozinho**, toda semana, sem ação manual.

Criado pela [Citado](https://citado.app.br) — sites otimizados para buscadores e para IAs generativas (GEO), por Virginia Marçal.

## O problema que ele resolve

Indexar um site uma vez não basta — **índice envelhece**:

- O Bing costuma baixar o sitemap **uma única vez** no cadastro e demorar semanas para voltar (às vezes não volta). Páginas criadas depois simplesmente não entram no índice.
- Conteúdo alterado permanece desatualizado nos buscadores até o robô decidir voltar.
- E as IAs herdam o problema: o **ChatGPT busca no índice do Bing**, o Perplexity combina índice próprio com Bing, o Gemini usa o Google. Índice velho = respostas de IA com a foto velha do seu negócio.

## Como funciona

O motor roda **dentro da própria infraestrutura do site** (Cloudflare Worker com [Cron Trigger](https://developers.cloudflare.com/workers/configuration/cron-triggers/)) — sem servidor para manter, sem mensalidade de ferramenta, dentro do plano gratuito do Cloudflare.

**Toda semana**, para cada site configurado:

1. Lê o `sitemap.xml` e extrai todas as URLs.
2. Notifica todas via [IndexNow](https://www.indexnow.org/) (cobre Bing → ChatGPT, e outros buscadores parceiros).
3. Reenvia o sitemap ao [Bing Webmaster API](https://learn.microsoft.com/en-us/bingwebmaster/getting-access) — matando a armadilha do "sitemap baixado uma vez só".

**Regra de ouro:** o motor nunca pode quebrar nada — cada site roda em `try/catch` e falha vira registro de log, jamais erro.

Este repositório traz a variante **central** (um Worker que varre uma lista de sites — ideal para sites estáticos em Pages/Vercel, que não têm cron próprio). A variante **embutida** (o motor dentro do Worker do próprio site, com ping imediato a cada mudança de conteúdo publicada no painel) segue a mesma lógica; um exemplo dela em produção está descrito em [citado.app.br/indexacao-automatica](https://citado.app.br/indexacao-automatica).

## Deploy em 5 minutos

Pré-requisitos: conta Cloudflare (grátis) e Node.js.

1. **Hospede a key do IndexNow** em cada site: gere uma key (32 hex) e sirva o arquivo `https://seusite.com/SUAKEY.txt` com a key como conteúdo. É pública por design — o protocolo exige.
2. **Edite a lista `SITES`** em [`src/index.ts`](src/index.ts): host canônico (com `www` se for o caso), URL do sitemap, key IndexNow e, se o site estiver registrado no Bing Webmaster, a URL exatamente como registrada.
3. **Deploy:**
   ```bash
   npx wrangler deploy
   ```
4. **Secrets (opcionais):**
   ```bash
   npx wrangler secret put BING_API_KEY   # reenvio de sitemap ao Bing (sem ela, roda só IndexNow)
   npx wrangler secret put RUN_TOKEN      # habilita disparo manual via GET /run?token=...
   ```
   ⚠️ **Armadilha no Windows:** secret cadastrado via pipe do PowerShell carrega `\r\n` invisível no fim — o Bing responde `HTTP 400`. Por isso o código faz `.trim()` em todo secret. Aprendemos em produção. 🙂
5. **Teste sem esperar segunda-feira:**
   ```bash
   curl "https://citado-indexer.SEU-SUBDOMINIO.workers.dev/run?token=SEU_RUN_TOKEN"
   ```

O agendamento (`[triggers]` no [`wrangler.toml`](wrangler.toml)) está para toda segunda 07:30 UTC — ajuste à vontade.

## O que ele garante — e o que não garante

- ✅ **Presença:** as páginas ficam sempre nos índices que os buscadores e as IAs consultam.
- ✅ **Frescor:** página nova ou alterada é descoberta em minutos, não em semanas.
- ❌ **Posição:** ranking vem de conteúdo, relevância e reputação. Quem promete "primeiro lugar garantido" está vendendo fumaça.

## Em produção

- [oficina306.com.br](https://oficina306.com.br/como-este-site-aparece-nas-ias) — variante embutida, com status da última varredura publicado em página aberta.
- [citado.app.br](https://citado.app.br) — coberto pela variante central deste repositório.

## Quem faz

O **citado-indexer** é parte do pipeline da [**Citado**](https://citado.app.br) — criação de sites profissionais com indexação técnica completa e GEO (Generative Engine Optimization) para pequenos negócios brasileiros serem encontrados no Google e citados por ChatGPT, Gemini, Claude e Perplexity.

Guia de GEO em português: [citado.app.br/geo](https://citado.app.br/geo)

## Licença

[MIT](LICENSE) — use, adapte, embuta nos seus sites. Se ele te ajudar, um link para [citado.app.br](https://citado.app.br) é sempre bem-vindo.
