# Setup Guide — Novo Cliente

Este arquivo define o fluxo completo para Claude configurar o sistema para um novo cliente.
**No início de cada sessão nova, Claude deve fazer as perguntas abaixo antes de qualquer ação.**

---

## Perguntas que Claude deve fazer antes de começar

> Claude: antes de configurar qualquer coisa, preciso de algumas informações. Vou perguntar em blocos.

---

### Bloco 1 — Sobre o negócio

1. **Qual é o nome da empresa?**
   _(ex: Moraes Caçambas, HermeSpot, etc.)_

2. **Qual é o serviço principal?**
   _(ex: aluguel de caçambas, manutenção de ar condicionado, etc.)_

3. **Qual é o número do WhatsApp com DDI e DDD?**
   _(ex: 5519993277969 — sem espaços, traços ou parênteses)_

4. **Qual é a cidade / região de atendimento?**
   _(ex: Campinas e Região Metropolitana)_

5. **Qual é o horário de funcionamento?**
   _(ex: Segunda a Sábado, 07h às 18h)_

6. **Tem endereço físico para mostrar no site?**
   _(pode deixar em branco se não quiser exibir)_

---

### Bloco 2 — Números para os cards de estatísticas

7. **Quantos clientes atendidos (aproximado)?**
   _(ex: 500)_

8. **Quantas obras / serviços realizados?**
   _(ex: 1200)_

9. **Entrega em quantas horas? Ou qual é o diferencial em números?**
   _(ex: entrega em 24h, 10 anos de experiência)_

10. **Mais algum número relevante para o 4º card?**
    _(ex: anos de experiência, cidades atendidas, etc.)_

---

### Bloco 3 — Repositório GitHub

11. **Qual é o nome de usuário do GitHub?**
    _(ex: freak-m)_

12. **Qual é o nome do repositório?**
    _(ex: moraescacambas — será criado se não existir)_

13. **Você já tem um GitHub Personal Access Token (PAT)?**
    - Se **sim**: qual é a validade? O repositório está no escopo?
    - Se **não**: vou te explicar como criar abaixo

    > **Como criar um PAT:**
    > 1. Acesse github.com → clique na sua foto → **Settings**
    > 2. Role até o fim → **Developer settings**
    > 3. **Personal access tokens → Tokens (classic) → Generate new token (classic)**
    > 4. Nome: `admin-panel-[nome do cliente]`
    > 5. Validade: 1 ano
    > 6. Marque: `repo` (todas as permissões de repositório)
    > 7. Clique em **Generate token** — copie e guarde, não aparece mais

---

### Bloco 4 — Domínio e Cloudflare

14. **Qual é o domínio do site?**
    _(ex: clientedominio.com.br)_

15. **O domínio já está no Cloudflare?**
    - Se **não**: precisa transferir os nameservers para o Cloudflare antes de continuar

16. **Tem acesso ao painel do Cloudflare da conta onde o domínio está?**

17. **Qual subdomínio quer usar para o Worker de analytics?**
    _(ex: `analytics.clientedominio.com.br` — ou pode usar a URL padrão `.workers.dev`)_

---

### Bloco 5 — Analytics e integrações

18. **Tem Google Analytics configurado? Se sim, qual é o Measurement ID?**
    _(formato: `G-XXXXXXXXXX` — encontra em GA4 → Admin → Fluxos de dados → ID de medição)_

19. **Tem Meta Pixel? Se sim, qual é o ID?**
    _(formato numérico, ex: `1234567890123`)_

20. **Tem verificação de domínio do Meta? Se sim, qual é o código?**
    _(encontra em Meta Business Suite → Configurações → Domínios)_

---

### Bloco 6 — Conteúdo e marca

21. **Tem logo do cliente? Em qual formato?**
    _(`.webp` preferido, aceita `.png` ou `.svg`)_

22. **Tem imagem de compartilhamento (OG image)?**
    _(1200×630px — aparece quando o link é compartilhado no WhatsApp/Facebook)_

23. **Qual é o slogan ou frase principal do hero do site?**

24. **Quais são as características do serviço principal?** (lista de bullets)
    _(ex: "Reformas completas de residências", "Construção civil", etc.)_

25. **Como funciona o serviço? (3 passos)**
    _(ex: Passo 1: Entre em contato → Passo 2: Agendamos → Passo 3: Retiramos)_

26. **Quais são os 4 benefícios principais da empresa?**
    _(ex: Entrega Rápida, Empresa Licenciada, etc.)_

---

## Após coletar todas as respostas

Com as informações acima, Claude deve:

1. **Criar o repositório** no GitHub com os arquivos base (copiar deste repositório)
2. **Editar `admin/index.html`**: mudar `const REPO = 'owner/repo'` e a URL padrão do Worker
3. **Configurar o Cloudflare Worker** (ver seção abaixo)
4. **Preencher o `content.json`** com todos os dados do cliente
5. **Atualizar branding**: logo, favicon, nome no `<title>` e na sidebar
6. **Ativar GitHub Pages**: Settings → Pages → branch `main` → pasta `/`
7. **Configurar segurança Cloudflare** (Bot Fight Mode + Block AI Bots)

---

## Configuração do Cloudflare Worker

### Passo 1 — Criar o Worker
1. Acesse [dash.cloudflare.com](https://dash.cloudflare.com)
2. **Workers & Pages → Create → Create Worker**
3. Nomeie (ex: `analytics-[nomecliente]`)
4. Clique em **Deploy** (código temporário)
5. Clique em **Edit code** e cole o código abaixo
6. Clique em **Deploy**

### Passo 2 — Criar o KV Namespace
1. **Workers & Pages → KV → Create namespace**
2. Nome: `[nomecliente]-kv`
3. Clique em **Add**

### Passo 3 — Vincular KV ao Worker
1. Abra o Worker → **Settings → Bindings → Add**
2. Tipo: **KV Namespace**
3. Variable name: `CACAMBAS_KV` _(deve ser exatamente esse nome)_
4. KV namespace: selecione o que criou
5. **Save**

### Passo 4 — Adicionar variáveis de ambiente
1. No Worker → **Settings → Variables → Add variable**
2. Adicione:
   - `CF_ZONE_ID` = _(Zone ID do domínio — veja abaixo como encontrar)_
   - `CF_API_TOKEN` = _(Token da API — veja abaixo como criar)_
3. Marque ambos como **Encrypt** → **Save**

### Como encontrar o Zone ID
> Cloudflare Dashboard → clique no domínio → Overview → coluna direita → **Zone ID** (copiar)

### Como criar o CF API Token
> 1. Cloudflare → sua foto → **My Profile → API Tokens → Create Token**
> 2. Use **Custom Token**
> 3. Permissions: `Zone → Analytics → Read`
> 4. Zone Resources: `Include → Specific zone → [domínio do cliente]`
> 5. **Continue to summary → Create Token** — copie, não aparece mais

### Passo 5 — Domínio personalizado (opcional)
1. Worker → **Settings → Triggers → Custom Domains → Add Custom Domain**
2. Informe: `analytics.[dominiocliente]`
3. O Cloudflare cria o registro DNS automaticamente

### Código do Worker

```javascript
export default {
  async fetch(request, env) {
    const headers = {
      'Content-Type': 'application/json',
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'GET, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type',
    };

    if (request.method === 'OPTIONS') return new Response(null, { headers });

    const url   = new URL(request.url);
    const days  = parseInt(url.searchParams.get('days') || '7');
    const from  = url.searchParams.get('from');
    const to    = url.searchParams.get('to');

    const now   = new Date();
    const fmt   = d => d.toISOString().split('T')[0];
    const end   = to   ? new Date(to)   : now;
    const start = from ? new Date(from) : new Date(now.getTime() - (days - 1) * 86400000);
    const prevEnd   = new Date(end.getTime()   - days * 86400000);
    const prevStart = new Date(start.getTime() - days * 86400000);

    // ATENÇÃO: httpRequestsAdaptiveGroups e webAnalyticsRumPageloadEventsAdaptiveGroups
    // só aceitam range de 1 dia. Ambos date_geq E date_leq devem ser fmt(now).
    const query = `{
      viewer {
        zones(filter: { zoneTag: "${env.CF_ZONE_ID}" }) {
          current: httpRequests1dGroups(
            limit: ${days + 5}
            orderBy: [date_ASC]
            filter: { date_geq: "${fmt(start)}", date_leq: "${fmt(end)}" }
          ) {
            dimensions { date }
            sum { requests cachedRequests pageViews bytes }
            uniq { uniques }
          }
          prev: httpRequests1dGroups(
            limit: ${days + 5}
            orderBy: [date_ASC]
            filter: { date_geq: "${fmt(prevStart)}", date_leq: "${fmt(prevEnd)}" }
          ) {
            dimensions { date }
            sum { requests pageViews }
            uniq { uniques }
          }
          devices: httpRequestsAdaptiveGroups(
            limit: 10
            orderBy: [count_DESC]
            filter: { date_geq: "${fmt(now)}", date_leq: "${fmt(now)}" }
          ) {
            count
            dimensions { clientDeviceType }
          }
          os: webAnalyticsRumPageloadEventsAdaptiveGroups(
            limit: 10
            orderBy: [count_DESC]
            filter: { date_geq: "${fmt(now)}", date_leq: "${fmt(now)}" }
          ) {
            count
            dimensions { operatingSystem }
          }
        }
      }
    }`;

    let cfData = {};
    try {
      const cfRes = await fetch('https://api.cloudflare.com/client/v4/graphql', {
        method: 'POST',
        headers: {
          'Authorization': `Bearer ${env.CF_API_TOKEN}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({ query }),
      });
      cfData = await cfRes.json();
    } catch (e) {
      return new Response(JSON.stringify({ error: e.message }), { status: 500, headers });
    }

    // Lê eventos do KV
    let events = null;
    try {
      events = await env.CACAMBAS_KV.get('events', { type: 'json' });
    } catch (_) {}

    return new Response(JSON.stringify({ ...cfData, events }), { headers });
  },
};
```

---

## Sobre os cards do dashboard e o que cada um precisa

| Card | Dados | O que precisa no Worker |
|---|---|---|
| Total Visitas | `uniq.uniques` do `httpRequests1dGroups` | Worker configurado com `CF_ZONE_ID` e `CF_API_TOKEN` |
| Page Views | `sum.pageViews` do `httpRequests1dGroups` | Idem |
| Tráfego Total | `sum.bytes` do `httpRequests1dGroups` | Idem (`bytes` na query GraphQL) |
| Taxa de Cache | `sum.cachedRequests / sum.requests` | Idem (`cachedRequests` e `requests` na query) |
| Cliques WhatsApp | `events.wa` do KV | KV criado, vinculado e o site gravando eventos |
| Fotos Abertas | `events.photo` do KV | KV criado, vinculado e o site gravando eventos |
| Tempo na Página | `events.time_avg` do KV | KV criado, vinculado e o site gravando eventos |
| Dispositivos | `httpRequestsAdaptiveGroups` | Worker com `CF_ZONE_ID` e `CF_API_TOKEN` |
| Sistema Operacional | `webAnalyticsRumPageloadEventsAdaptiveGroups` | Idem |
| Países | `countryMap` do `httpRequests1dGroups` | Idem |

---

## Segurança Cloudflare (fazer para todos os clientes)

**Security → Bots:**
- Bot Fight Mode → **On**
- Block AI Bots → **On** → Scope: **Block on all pages**

---

## Arquivos a modificar ao clonar para novo cliente

| Arquivo | Linha | O que mudar |
|---|---|---|
| `admin/index.html` | ~1491 | `const REPO = 'owner/repo-name';` |
| `admin/index.html` | ~1552 | `const CF_WORKER_DEFAULT = 'https://analytics.newdomain.com';` |
| `admin/index.html` | sidebar | Nome da empresa |
| `admin/index.html` | logo `src` | Caminho do logo |
| `admin/index.html` | `<title>` | Título da página |
| `content.json` | tudo | Conteúdo do cliente (editável pelo painel) |
| `index.html` | `<head>` | Meta tags de SEO e schema.org |

---

## Erros comuns e como resolver

| Erro | Causa | Solução |
|---|---|---|
| `zone cannot request time range wider than 1d` | `httpRequestsAdaptiveGroups` com range > 1 dia | Mude `date_geq` e `date_leq` para `fmt(now)` no Worker |
| `Sem dados. Verifique CF_ZONE_ID` | Zone ID errado ou em branco | Workers → Settings → Variables → verificar `CF_ZONE_ID` |
| `401` da API do GitHub | PAT expirado ou escopo errado | Gerar novo PAT com escopo `repo` |
| Cards de Cache/Bandwidth mostram `—` | Worker sem `bytes`/`cachedRequests` na query | Atualizar código do Worker com a query acima |
| Cards de WhatsApp/Foto mostram `—` | KV não vinculado ou site não gravando eventos | Verificar binding `CACAMBAS_KV` no Worker |
| Dashboard mostra spinner infinito | Sem cache + erro no Worker | Corrigir Worker primeiro; cache cria na 1ª requisição bem-sucedida |
