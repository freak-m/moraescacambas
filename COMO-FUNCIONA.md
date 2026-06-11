# Como o Sistema foi Construído

Documentação técnica completa do painel administrativo e do site estático.

---

## Visão Geral

O sistema não tem servidor backend. É composto por:

```
Navegador do usuário
├── GitHub Pages          → index.html (site público)
│                         → admin/index.html (painel admin)
│                         → content.json (todo o conteúdo do site)
│
└── Cloudflare Worker     → consulta a API de analytics da Cloudflare
                          → lê/grava eventos no KV (WhatsApp, fotos, tempo)
```

Tudo roda no navegador. O painel fala diretamente com a API do GitHub e com o Worker — sem intermediários.

---

## 1. Autenticação

### Como funciona o login

A senha do painel é o **GitHub Personal Access Token (PAT)**.

```
Usuário digita o PAT
    ↓
GET https://api.github.com/repos/{REPO}
    com header: Authorization: token {PAT}
    ↓
Se 200 OK → token válido → abre o painel
Se 401/404 → exibe erro
```

Após login bem-sucedido:
```javascript
sessionStorage.setItem('gh_token', token);
```

**Por que sessionStorage e não localStorage?**
- `sessionStorage` é limpo quando a aba fecha ou o usuário faz logout
- `localStorage` persiste para sempre no navegador
- PAT é uma credencial sensível — não deve ficar armazenada permanentemente

### Auto-login

Se já existe um token em `sessionStorage` (aba reaberta na mesma sessão), o painel pula a tela de login automaticamente:

```javascript
(function autoLogin() {
  const existing = sessionStorage.getItem('gh_token');
  if (!existing) return;
  // reconstrói o DOM e abre direto
})();
```

### Logout

```javascript
sessionStorage.removeItem('gh_token');
// esconde o painel, mostra a tela de login
```

---

## 2. Estrutura do Painel

### Layout

```
┌─────────────────────────────────────────────────────┐
│  Sidebar (abas + botões)  │  Editor  │  Preview     │
│  ─ Conteúdo               │          │  (iframe)    │
│  ─ Dashboard              │          │              │
│  ─ Integrações            │          │              │
│                           │          │              │
│  [Publicar] [Sair]        │          │              │
└─────────────────────────────────────────────────────┘
```

Nas abas **Dashboard** e **Integrações** o preview é escondido e o editor expande para tela cheia:

```javascript
const NO_PREVIEW_TABS = new Set(['dashboard', 'integrations']);

if (NO_PREVIEW_TABS.has(tab)) {
  preview.classList.add('hidden');
  editor.classList.add('expand');
}
```

### Sequência de inicialização (após login)

```
login(token)
  ↓
buildBenefitFields()   → cria os 4 campos de benefícios dinamicamente no DOM
buildStatsFields()     → cria os 4 campos de estatísticas dinamicamente no DOM
initEditors()          → inicializa todas as instâncias do Quill.js
setupPreview()         → monta o iframe com o site
setupPreviewToggle()   → botão de mostrar/esconder preview no mobile
wirePreviewListeners() → conecta cada campo de input ao debouncedSync()
  ↓
loadContent()          → busca content.json do GitHub
  ↓
populateForm(content)  → preenche todos os campos com os dados
  ↓
syncPreview()          → envia os dados para o iframe (após 800ms)
```

---

## 3. Editor de Conteúdo

### Abas do editor (accordion)

Cada seção do site tem um accordion. Ao abrir, `focusPreviewSection()` destaca a seção correspondente no iframe.

| Accordion | Seção no site |
|---|---|
| hero | `#hero` |
| service | `#services` |
| how | `#how` |
| benefits | `#benefits` |
| stats | `#stats` |
| contact | `#contact` / `#site-footer` |

### Campos de texto rico (Quill.js)

Campos com formatação (negrito, listas, links) usam Quill.js 1.3.7 via CDN:

```javascript
const QUILL_TOOLBAR = [
  ['bold', 'italic', 'underline'],
  [{ list: 'ordered' }, { list: 'bullet' }],
  ['link', 'clean']
];

quills['q-hero-subtitle'] = new Quill('#q-hero-subtitle', {
  theme: 'snow',
  modules: { toolbar: QUILL_TOOLBAR }
});
```

Campos que usam Quill: hero subtitle, descrição do serviço, 3 passos do "Como Funciona", 4 benefícios, subtítulo do contato, painéis do WhatsApp.

Campos comuns usam `<input type="text">` ou `<textarea>`.

---

## 4. Preview em Tempo Real

### Como funciona

O preview é um `<iframe>` que carrega o `index.html` do site. Quando o usuário edita um campo, o editor injeta os novos valores diretamente no DOM do iframe via `contentDocument`:

```javascript
function syncPreview() {
  const doc = iframe.contentDocument;

  // Exemplo: atualiza o subtítulo do hero
  const el = doc.getElementById('hero-subtitle');
  el.innerHTML = quills['q-hero-subtitle'].root.innerHTML;
}
```

O sync é chamado com debounce de 250ms para não travar enquanto o usuário digita:

```javascript
let syncTimer;
function debouncedSync() {
  clearTimeout(syncTimer);
  syncTimer = setTimeout(syncPreview, 250);
}
```

Todos os inputs e os editores Quill disparam `debouncedSync` ao mudar:

```javascript
// Inputs comuns
document.querySelectorAll('.adm-editor input, .adm-editor textarea')
  .forEach(el => el.addEventListener('input', debouncedSync));

// Quill editors
Object.values(quills).forEach(q => q.on('text-change', debouncedSync));
```

### Highlight de seção

Ao trocar de aba ou abrir um accordion, a seção correspondente no iframe recebe um flash amarelo:

```javascript
function focusPreviewSection(sectionId) {
  const section = doc.getElementById(sectionId);
  section.scrollIntoView({ behavior: 'smooth', block: 'start' });
  setTimeout(() => {
    section.classList.add('cms-focus-flash');  // animação CSS
  }, 700);
}
```

Um badge `"✏ Editando esta seção"` aparece no canto superior direito do iframe durante a edição.

---

## 5. Publicar

### Fluxo completo

```
Usuário clica em "Publicar"
    ↓
GET /repos/{REPO}/contents/content.json
    → obtém o SHA atual do arquivo (necessário para o PUT)
    ↓
collectForm()
    → lê todos os inputs e editores Quill
    → monta o objeto JSON completo
    ↓
JSON.stringify(content, null, 2)
    → serializa com indentação
    ↓
btoa(unescape(encodeURIComponent(jsonStr)))
    → codifica em base64 (requisito da API do GitHub)
    ↓
PUT /repos/{REPO}/contents/content.json
    body: { message: "Admin: update content", content: base64, sha: sha }
    ↓
GitHub Pages faz rebuild automático (~30 segundos)
```

### Por que precisa do SHA?

A API do GitHub exige o SHA do arquivo atual para evitar conflitos. Se duas pessoas editarem ao mesmo tempo, o segundo PUT falha com `409 Conflict`.

---

## 6. Upload de Imagens

As imagens são enviadas diretamente para o repositório via GitHub API:

```
Usuário seleciona o arquivo
    ↓
FileReader.readAsDataURL(file)
    → converte o arquivo em base64
    ↓
GET /repos/{REPO}/contents/{filename}
    → verifica se o arquivo já existe (para obter o SHA)
    ↓
PUT /repos/{REPO}/contents/{filename}
    body: { message: "Admin: upload image", content: base64, sha? }
    ↓
URL pública: https://moraescacambas.com.br/{filename}
```

**Preview instantâneo:** enquanto o GitHub ainda não publicou o arquivo, o preview usa `URL.createObjectURL(file)` — uma URL temporária local — para mostrar a imagem na hora.

---

## 7. Dashboard de Analytics (Cloudflare)

### Origem dos dados

O painel não consulta a Cloudflare diretamente. Isso seria inseguro (exporia o API Token). Em vez disso, um **Cloudflare Worker** atua como intermediário:

```
admin/index.html
    ↓  GET https://analytics.moraescacambas.com.br/?days=7
Cloudflare Worker
    ↓  POST https://api.cloudflare.com/client/v4/graphql
       Authorization: Bearer {CF_API_TOKEN}  ← variável de ambiente, nunca no código
    ↓
Retorna JSON para o painel
```

### Cache

Para evitar abusar da API da Cloudflare e tornar o painel rápido:

```
loadDashboardData()
    ↓
Existe cache em localStorage?  (cf_cache_7d, cf_cache_30d, cf_cache_90d)
    ├── SIM → renderiza imediatamente
    │         ↓
    │         Cache tem mais de 5 minutos?
    │             ├── SIM → busca em background + mostra banner amarelo "Atualizando..."
    │             └── NÃO → nada a fazer
    │
    └── NÃO → mostra spinner → busca → renderiza → guarda no cache
```

**Hard expiry:** 2 horas. Após isso sempre rebusca mesmo que o usuário não peça.

### Métricas exibidas

| Métrica | Endpoint Cloudflare | Restrição |
|---|---|---|
| Visitas diárias (gráfico) | `httpRequests1dGroups` | Suporta qualquer período |
| Tipos de dispositivo (donut) | `httpRequestsAdaptiveGroups` | **Máximo 1 dia** |
| Sistema operacional | `webAnalyticsRumPageloadEventsAdaptiveGroups` | **Máximo 1 dia** |
| Eventos (WhatsApp, fotos) | KV Namespace | — |

Por causa da restrição de 1 dia, **ambas as datas** dos endpoints adaptativos são sempre hoje:
```javascript
filter: { date_geq: "${fmt(now)}", date_leq: "${fmt(now)}" }
```

### Gráfico de Dispositivos (donut SVG)

O gráfico é gerado sem nenhuma biblioteca — apenas matemática e SVG:

```javascript
const R  = 42;  // raio externo
const ri = 26;  // raio interno (cria o buraco central)
let angle = -Math.PI / 2;  // começa do topo (12h)

data.forEach(seg => {
  const ratio = seg.count / total;
  const end   = angle + ratio * 2 * Math.PI;

  // Calcula os 4 pontos do arco (externo início, externo fim, interno fim, interno início)
  const path = `M ... A${R},${R} ... L ... A${ri},${ri} ... Z`;

  angle = end;  // próxima fatia começa onde esta terminou
});
```

Cada fatia é um `<path>` SVG com:
- Arco externo no sentido horário (raio R)
- Linha para dentro
- Arco interno no sentido anti-horário (raio ri)
- Fechamento do path

### Gráfico de visitas (sparkline)

Barras + linha poligonal geradas em SVG puro também:

```javascript
function sparklineSVG(values, dates, color) {
  // Calcula a posição de cada barra proporcionalmente ao valor máximo
  // Gera <rect> para cada barra e <polyline> para a linha
}
```

---

## 8. Cloudflare Worker

### Responsabilidades

1. Recebe `GET /?days=N` do painel
2. Consulta a API GraphQL da Cloudflare com as credenciais secretas
3. Lê eventos do KV Namespace
4. Retorna JSON combinado

### Variáveis de ambiente (nunca no código-fonte)

| Variável | O que é |
|---|---|
| `CF_API_TOKEN` | Token da API Cloudflare (permissão: Zone Analytics Read) |
| `CF_ZONE_ID` | ID da zona (domínio) no Cloudflare |

### KV Binding

| Nome no Worker | Namespace KV |
|---|---|
| `CACAMBAS_KV` | Namespace criado no painel do Cloudflare |

### O que é gravado no KV

O `index.html` público grava eventos cada vez que:
- Usuário clica no botão do WhatsApp → incrementa `wa`
- Usuário abre a foto do serviço → incrementa `photo`
- Usuário passa tempo na página → registra em `time`

O Worker retorna esses dados para o painel no campo `events`.

---

## 9. Integrações

Configuradas na aba **Integrações** e salvas em `content.json`:

| Campo | Onde fica | Como é usado |
|---|---|---|
| Google Analytics ID (`G-XXXXXXX`) | `content.json → integrations.google_analytics` | `index.html` injeta o script do GA4 |
| Meta Pixel ID | `content.json → integrations.meta_pixel` | Script do Pixel na `<head>` |
| Meta verificação | `content.json → integrations.meta_verify` | `<meta name="facebook-domain-verification">` |
| Worker URL | `localStorage['cf_worker_url']` + `content.json → integrations.worker_url` | Dashboard busca analytics nessa URL |

---

## 10. content.json

É o único arquivo de dados do site. O `index.html` público o busca ao carregar e monta toda a página dinamicamente. O painel admin o lê, edita e escreve de volta.

```
index.html público
    ↓
fetch('/content.json')
    ↓
Monta SEO (title, description, schema.org)
Monta seções (hero, services, how, benefits, stats, contact, footer)
Injeta scripts de integração (GA, Meta Pixel)
```

Estrutura principal:

```
content.json
├── seo          → title, description, keywords, og_image, schema_local, faq
├── hero         → subtitle
├── services     → tag, title, title_highlight, subtitle, size, name, description, features[]
├── how          → subtitle, steps[3]
├── benefits     → subtitle, items[4]
├── stats        → [4]{ value, suffix, label }
├── contact      → tag, title, title_highlight, subtitle, cards[], wa_panel_label, wa_panel_sub
├── footer       → location, hours
└── integrations → google_analytics, meta_pixel, meta_verify, worker_url
```

---

## 11. Segurança

| O que | Onde fica | Por quê |
|---|---|---|
| GitHub PAT | `sessionStorage` | Limpo ao fechar a aba |
| CF API Token | Worker env var | Nunca chega ao navegador |
| CF Zone ID | Worker env var | Nunca chega ao navegador |
| Worker URL | `localStorage` + `content.json` | Não é segredo — é uma URL pública |
| Painel admin | `admin/index.html` com `noindex, nofollow` | Não indexado por buscadores |

**Cloudflare Security (recomendado para todos os domínios):**
- Bot Fight Mode → On
- Block AI Bots → Block on all pages

---

## 12. Dependências Externas

| Dependência | Versão | Usado para |
|---|---|---|
| Quill.js | 1.3.7 (CDN) | Campos de texto rico |
| GitHub API | v3 | Ler/gravar content.json, upload de imagens |
| Cloudflare GraphQL API | — | Dados de analytics |
| Cloudflare Workers KV | — | Eventos do site |

Zero dependências de build. Zero npm. Zero bundler. Tudo roda diretamente no navegador.

---

## 13. Fluxo Completo de uma Edição

```
1. Usuário abre admin/index.html
2. Digita o GitHub PAT → login verifica com a API → token vai para sessionStorage
3. buildBenefitFields() + buildStatsFields() criam campos dinâmicos no DOM
4. initEditors() inicializa os editores Quill
5. loadContent() busca content.json do GitHub (decodifica base64 → JSON)
6. populateForm() preenche todos os campos com os dados atuais
7. setupPreview() carrega index.html num iframe ao lado
8. syncPreview() envia os valores para o iframe (com 800ms de delay inicial)
9. Usuário edita um campo → debouncedSync() → syncPreview() atualiza o iframe em 250ms
10. Usuário clica "Publicar":
    - collectForm() lê todos os campos → monta JSON
    - ghGet() busca o SHA atual do content.json
    - ghPut() envia o novo JSON em base64 com o SHA
    - GitHub Pages faz rebuild (~30s)
    - Toast "Publicado com sucesso"
```
