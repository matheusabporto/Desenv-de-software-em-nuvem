# Guia Completo — Projeto 3 de Desenvolvimento de Software em Nuvem

> Stack: **Node.js + Express + Supabase + AWS EC2** (duas VMs)
> Objetivo: CRUD completo de produtos com arquitetura distribuída em nuvem.

---

## Índice

1. [Antes de começar: a arquitetura na sua cabeça](#1-antes-de-começar-a-arquitetura-na-sua-cabeça)
2. [Conceitos que você precisa entender de verdade](#2-conceitos-que-você-precisa-entender-de-verdade)
3. [Fase 1 — Setup do repositório](#fase-1--setup-do-repositório)
4. [Fase 2 — Banco de dados no Supabase](#fase-2--banco-de-dados-no-supabase)
5. [Fase 3 — Backend local](#fase-3--backend-local)
6. [Fase 4 — Frontend local](#fase-4--frontend-local)
7. [Fase 5 — Deploy do backend na EC2](#fase-5--deploy-do-backend-na-ec2)
8. [Fase 6 — Deploy do frontend na EC2](#fase-6--deploy-do-frontend-na-ec2)
9. [Fase 7 — Testes e apresentação](#fase-7--testes-e-apresentação)
10. [Apêndices](#apêndices)

---

## 1. Antes de começar: a arquitetura na sua cabeça

Antes de qualquer linha de código, você precisa **enxergar** o que está construindo. Esse é o desenho mental:

```
┌───────────────────┐         ┌───────────────────┐         ┌───────────────────┐
│                   │  HTTP   │                   │  HTTPS  │                   │
│   VM 2 - EC2      │ ──────► │   VM 1 - EC2      │ ──────► │     SUPABASE      │
│   FRONTEND        │ ◄────── │   BACKEND         │ ◄────── │   (PostgreSQL)    │
│   HTML/CSS/JS     │  JSON   │   Node + Express  │   SQL   │   Tabela produtos │
│                   │         │                   │         │                   │
└───────────────────┘         └───────────────────┘         └───────────────────┘
       Porta 80                    Porta 3000                  Gerenciado pela
   (servindo arquivos)         (API REST)                     Supabase
```

**Três peças, três responsabilidades, três localizações diferentes na internet.** Cada uma só sabe da existência da próxima.

### Por que separar assim?

A pergunta que provavelmente vai aparecer na sua cabeça: "por que não jogar tudo numa máquina só?" Resposta curta: **acoplamento**. Quanto mais separado, mais resiliente e escalável.

Imagina que amanhã seu frontend bomba e precisa aguentar 10x mais acessos. Você sobe mais máquinas só de frontend, sem mexer no backend. Imagina que o banco precisa de manutenção — você troca o banco, o backend continua igual. Essa é a lógica de produção real.

No seu caso o objetivo é didático: **provar que você sabe orquestrar peças distribuídas**. É exatamente isso que o trabalho avalia.

---

## 2. Conceitos que você precisa entender de verdade

Vou explicar rápido, mas o suficiente para você ter base sólida. Se você dominar esses 6 conceitos, o resto é mecânica.

### 2.1 REST e CRUD

**CRUD** é o nome bonito para as 4 operações básicas de qualquer sistema com banco de dados: **Create, Read, Update, Delete**. Criar, ler, atualizar, deletar. É o feijão com arroz.

**REST** é um *estilo arquitetural* para construir APIs HTTP. Ele diz, basicamente:

- Cada coisa do seu sistema (recurso) tem uma URL identificadora. Ex: `/produtos`, `/produtos/42`
- O **verbo HTTP** define o que você quer fazer com aquele recurso, não a URL

| Operação CRUD | Verbo HTTP | Rota típica       | O que faz                      |
| ------------- | ---------- | ----------------- | ------------------------------ |
| Create        | POST       | `/produtos`       | Cria um produto novo           |
| Read (todos)  | GET        | `/produtos`       | Lista todos os produtos        |
| Read (um)     | GET        | `/produtos/:id`   | Busca um produto específico    |
| Update        | PUT        | `/produtos/:id`   | Atualiza um produto existente  |
| Delete        | DELETE     | `/produtos/:id`   | Apaga um produto               |

> **Por que isso importa:** quando você ver no enunciado "rotas para consultar, cadastrar, alterar, deletar", ele está pedindo exatamente as 5 operações REST acima. Saber o nome correto te ajuda a comunicar e a estruturar o código.

### 2.2 Códigos de status HTTP

Quando o backend responde, ele manda um **número** junto que diz se deu certo, deu errado, deu errado por culpa de quem, etc. Os principais para você:

- **200 OK** — Deu tudo certo (resposta padrão para GET/PUT/DELETE bem-sucedidos)
- **201 Created** — Deu certo e criou algo novo (use para POST)
- **400 Bad Request** — O cliente mandou algo errado (faltou campo, formato inválido)
- **404 Not Found** — Não existe o que você pediu (ex: produto com ID 999 que não existe)
- **500 Internal Server Error** — Deu pau no servidor, problema seu (do backend)

> **Por que isso importa:** retornar o status correto não é firula — é o que permite o frontend reagir adequadamente. Se o backend retorna sempre 200, mesmo quando deu erro, o frontend não tem como saber e mostrar mensagem para o usuário.

### 2.3 JSON

É o formato de troca de dados entre frontend e backend. Texto puro com estrutura de objeto/array. Exemplo:

```json
{
  "id": 1,
  "nome": "Notebook Dell",
  "descricao": "Notebook i7 16GB RAM",
  "preco": 4500.00,
  "quantidade": 12
}
```

O Node converte JSON ↔ objeto JavaScript com `JSON.parse()` e `JSON.stringify()`, mas usando Express com o middleware `express.json()`, ele faz isso automaticamente.

### 2.4 Variáveis de ambiente (`.env`)

Coisas como **senha do banco**, **URL do Supabase** e **chaves de API** **não podem ir para o GitHub**. Nunca. É falha de segurança grave.

A solução universal: você coloca essas coisas num arquivo chamado `.env` (sem nome, só extensão) e adiciona `.env` no `.gitignore`. Aí seu código lê dele via `process.env.NOME_DA_VARIAVEL`.

```env
SUPABASE_URL=https://abcdefg.supabase.co
SUPABASE_KEY=eyJhbGciOiJIUzI1NiIsInR5...
PORT=3000
```

No código:

```js
require('dotenv').config();
const supabaseUrl = process.env.SUPABASE_URL;
```

### 2.5 CORS (o inimigo que vai aparecer)

CORS = **Cross-Origin Resource Sharing**. É uma política de segurança do navegador.

**O problema:** quando seu frontend (rodando em `http://ip-da-vm-frontend`) tenta fazer um `fetch` para seu backend (em `http://ip-da-vm-backend:3000`), o navegador **olha torto** porque são origens diferentes. Por padrão, ele bloqueia.

**A solução:** o backend precisa **explicitamente dizer** "tudo bem, pode chamar de outras origens". Isso é feito com o pacote `cors`:

```js
const cors = require('cors');
app.use(cors());
```

Em produção real você configuraria para aceitar só origens específicas. Para o trabalho, `cors()` aberto resolve.

> **Por que isso importa:** esse é o erro número 1 que vai te paralisar na Fase 6. Quando o frontend rodar localmente, *funciona*. Quando você subir as duas VMs e o frontend bater no backend, *quebra*. Já te avisei — quando aparecer "blocked by CORS policy" no console, lembra desse parágrafo.

### 2.6 Async / Await

Antes (estilo antigo, como no código do professor):

```js
fs.readFile('users.json', 'utf8', function(err, data) {
  // ... código aninhado dentro do callback
});
```

Hoje (estilo moderno):

```js
async function listarProdutos() {
  const data = await supabase.from('produtos').select('*');
  return data;
}
```

`await` pausa a execução até a operação assíncrona terminar. `async` marca uma função que contém `await`. Muito mais legível, sem "callback hell".

---

## Fase 1 — Setup do repositório

> **Tempo estimado:** 30-45 minutos
> **Branch:** `main` (direto, este é o setup inicial)

### 1.1 Criar o repositório no GitHub

1. Vai no GitHub, "New repository"
2. Nome sugerido: `projeto3-cloud-crud-produtos` ou similar
3. Adicione descrição: "Aplicação Full Stack com CRUD de produtos, deploy em AWS EC2 e banco no Supabase. Disciplina de Desenvolvimento de Software em Nuvem - Unifor"
4. Marque "Add a README"
5. Adicione `.gitignore` template: **Node**
6. License: opcional (MIT é seguro)
7. Clone localmente:

```bash
git clone https://github.com/matheusabporto/projeto3-cloud-crud-produtos.git
cd projeto3-cloud-crud-produtos
```

### 1.2 Estrutura de pastas

Dentro do repositório clonado, crie a estrutura:

```
projeto3-cloud-crud-produtos/
├── README.md
├── .gitignore
├── backend/
│   └── (vazio por enquanto)
└── frontend/
    └── (vazio por enquanto)
```

Comando:

```bash
mkdir backend frontend
```

### 1.3 `.gitignore` na raiz

O template Node já cobre o essencial, mas confirma que tem essas linhas:

```
node_modules/
.env
.env.local
*.log
.DS_Store
```

> **Por que isso importa:** `node_modules/` é uma pasta gigantesca de dependências que o `npm install` regenera. Não vai pro Git. `.env` tem credenciais, nunca vai pro Git.

### 1.4 README inicial

Substitua o README gerado pelo GitHub por algo assim:

```markdown
# Projeto 3 — CRUD de Produtos em Nuvem

Aplicação Full Stack desenvolvida para a disciplina de Desenvolvimento de Software em Nuvem (Unifor, 2026.1, Prof. Américo Sampaio).

## Arquitetura

- **Frontend:** HTML + CSS + JavaScript puro, servido em VM EC2 separada
- **Backend:** Node.js + Express, API REST rodando em VM EC2 separada
- **Banco de dados:** PostgreSQL gerenciado via Supabase

## Estrutura do repositório

- `/backend` — API REST com Express
- `/frontend` — Interface estática

## Status

🚧 Em desenvolvimento

## Autor

Matheus Porto — Análise e Desenvolvimento de Sistemas, Unifor
```

### 1.5 Primeiro commit

```bash
git add .
git commit -m "chore: estrutura inicial do projeto"
git push origin main
```

> **Padrão de commits:** vou sugerir o padrão **Conventional Commits**. Os prefixos principais:
> - `feat:` nova funcionalidade
> - `fix:` correção de bug
> - `chore:` configuração, setup, manutenção
> - `docs:` documentação
> - `refactor:` melhoria de código sem mudar comportamento
> - `style:` formatação, espaços (não mexe no código)
>
> Use quando puder com escopo: `feat(backend): adiciona rota GET /produtos`

---

## Fase 2 — Banco de dados no Supabase

> **Tempo estimado:** 30 minutos
> **Branch:** crie `feat/setup-supabase`

### 2.1 O que é Supabase

Pensa no Supabase como um **PostgreSQL gerenciado**, com algumas coisas a mais: autenticação pronta, storage, edge functions. Para o trabalho, você só precisa da parte de banco.

Vantagens vs. instalar Postgres na própria EC2:
- Não precisa configurar nem manter servidor de banco
- Tem painel web bonito para criar tabelas, ver dados, rodar SQL
- Tier gratuito generoso
- API REST automática gerada (mas a gente vai usar o cliente JavaScript dele, que é mais limpo)

### 2.2 Criar projeto

1. Vai em [supabase.com](https://supabase.com), cria conta (login com GitHub funciona)
2. "New project"
3. Nome: `projeto3-cloud-unifor`
4. Senha do banco: **gera uma forte e salva num gerenciador de senhas**. Mesmo que você não vá usar diretamente, é credencial do seu projeto.
5. Region: **South America (São Paulo)** (latência menor)
6. Aguarda alguns minutos enquanto o projeto inicializa

### 2.3 Criar tabela `produtos`

No menu lateral, vai em **Table Editor** → "New table".

| Campo       | Tipo                | Configurações                           |
| ----------- | ------------------- | --------------------------------------- |
| id          | `int8` (bigint)     | Primary key, Is Identity (auto-increment) |
| nome        | `text`              | Not null                                |
| descricao   | `text`              | Not null                                |
| preco       | `numeric`           | Not null                                |
| quantidade  | `int4` (integer)    | Not null                                |
| created_at  | `timestamptz`       | Default: `now()`                        |

Desmarque "Enable Row Level Security (RLS)" por enquanto. RLS é uma camada de segurança boa, mas para o trabalho ela só vai complicar — você pode ativar depois se quiser estudar.

### 2.4 Inserir dados de teste

Ainda no Table Editor, "Insert" → "Insert row". Adiciona uns 3-5 produtos para testar:

```
nome: Notebook Dell Inspiron
descricao: i7, 16GB RAM, SSD 512GB
preco: 4500.00
quantidade: 5
```

```
nome: Mouse Logitech MX Master
descricao: Mouse sem fio ergonômico
preco: 450.00
quantidade: 20
```

### 2.5 Pegar as credenciais

Vai em **Project Settings → API**. Você vai precisar de duas coisas:

- **Project URL** — algo como `https://abcdefg.supabase.co`
- **anon public key** — começa com `eyJ...` (chave longa)

> **Sobre as chaves:** o Supabase tem duas chaves principais:
> - `anon` (pública) — usada em apps frontend, respeita RLS
> - `service_role` (secreta) — bypassa RLS, **só** no backend
>
> Como desabilitamos RLS, qualquer uma funciona. Use a `anon` por hábito de boas práticas. **Mesmo a anon, nunca cole direto no código** — sempre via `.env`.

### 2.6 Commit dessa fase

Nessa fase você não escreveu código (foi tudo na interface do Supabase), mas vale documentar no README do `/backend`. Crie um `backend/README.md`:

```markdown
# Backend

## Banco de dados

Banco gerenciado via Supabase (PostgreSQL).

### Tabela `produtos`

| Coluna      | Tipo          | Descrição                          |
| ----------- | ------------- | ---------------------------------- |
| id          | bigint        | Primary key, auto-increment        |
| nome        | text          | Nome do produto                    |
| descricao   | text          | Descrição do produto               |
| preco       | numeric       | Preço (decimal)                    |
| quantidade  | integer       | Quantidade em estoque              |
| created_at  | timestamptz   | Data de criação (automática)       |
```

```bash
git checkout -b feat/setup-supabase
git add backend/README.md
git commit -m "docs(backend): documenta schema da tabela produtos"
git push origin feat/setup-supabase
```

Pode fazer merge para `main` ou abrir um Pull Request (PR) — PRs são bom hábito para portfólio. No GitHub, clica em "Compare & pull request" → "Merge pull request".

---

## Fase 3 — Backend local

> **Tempo estimado:** 2-3 horas
> **Branches:** vamos quebrar em várias — `feat/backend-setup`, `feat/rotas-get`, `feat/rotas-post-put-delete`

### 3.1 Inicializar o projeto Node

Dentro da pasta `/backend`:

```bash
cd backend
npm init -y
```

Isso cria um `package.json`. Agora instale as dependências:

```bash
npm install express @supabase/supabase-js dotenv cors
```

O que cada uma faz:
- **express** — framework web, gerencia rotas
- **@supabase/supabase-js** — cliente oficial do Supabase para Node
- **dotenv** — lê variáveis do arquivo `.env`
- **cors** — middleware para liberar requisições de outras origens

### 3.2 Estrutura mínima

Dentro de `/backend`:

```
backend/
├── node_modules/      (gerado automaticamente, vai pro .gitignore)
├── .env               (vai pro .gitignore)
├── .env.example       (vai pro Git, sem valores reais)
├── package.json
├── package-lock.json
├── index.js           (arquivo principal)
└── README.md
```

### 3.3 Arquivo `.env`

```env
SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_KEY=sua-chave-anon-aqui
PORT=3000
```

E o `.env.example` (esse vai pro Git, serve de template para outras pessoas):

```env
SUPABASE_URL=
SUPABASE_KEY=
PORT=3000
```

### 3.4 Esqueleto do `index.js`

Aqui está a estrutura inicial. Vou te dar comentada para você entender, e depois você implementa cada rota:

```js
// Carrega variáveis do .env
require('dotenv').config();

const express = require('express');
const cors = require('cors');
const { createClient } = require('@supabase/supabase-js');

// Cria cliente do Supabase
const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_KEY
);

const app = express();

// Middlewares
app.use(cors());           // Libera CORS
app.use(express.json());   // Permite ler JSON no body das requisições

const PORT = process.env.PORT || 3000;

// ========== ROTAS ==========

// Rota de teste — útil para checar se o servidor está vivo
app.get('/', (req, res) => {
  res.json({ message: 'API de Produtos rodando!' });
});

// TODO: implementar as 5 rotas de produtos

// ========== INICIA SERVIDOR ==========
app.listen(PORT, () => {
  console.log(`Servidor rodando em http://localhost:${PORT}`);
});
```

### 3.5 Implementando as 5 rotas

Vou te dar **uma rota completa** como referência, e o esqueleto das outras pra você completar. A ideia é você raciocinar, não copiar.

**Rota 1: GET /produtos — listar todos**

```js
app.get('/produtos', async (req, res) => {
  try {
    const { data, error } = await supabase
      .from('produtos')
      .select('*')
      .order('id', { ascending: true });

    if (error) throw error;

    res.status(200).json(data);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Erro ao buscar produtos' });
  }
});
```

Repara nas partes:
- `async (req, res)` — função assíncrona, porque vamos usar `await`
- `try/catch` — captura qualquer erro
- `supabase.from('produtos').select('*')` — equivale a `SELECT * FROM produtos`
- `.order(...)` — ordena por id
- O Supabase retorna sempre `{ data, error }` — você sempre confere o erro antes de usar o data
- `res.status(200).json(data)` — devolve com status correto

**Rota 2: GET /produtos/:id — buscar um produto**

```js
app.get('/produtos/:id', async (req, res) => {
  try {
    const { id } = req.params;
    const { data, error } = await supabase
      .from('produtos')
      .select('*')
      .eq('id', id)
      .single();  // espera exatamente 1 resultado

    if (error) {
      // Se não encontrou, retorna 404
      return res.status(404).json({ error: 'Produto não encontrado' });
    }

    res.status(200).json(data);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Erro ao buscar produto' });
  }
});
```

**Rota 3: POST /produtos — criar produto**

Agora é com você. Pense no que precisa fazer:
1. Pegar os dados do `req.body` (nome, descricao, preco, quantidade)
2. Validar minimamente (todos preenchidos?)
3. Chamar `supabase.from('produtos').insert([{...}]).select()`
4. Retornar status **201** (criado) com o produto criado

Esqueleto:

```js
app.post('/produtos', async (req, res) => {
  try {
    const { nome, descricao, preco, quantidade } = req.body;

    // TODO: validação básica — todos os campos vieram?

    const { data, error } = await supabase
      .from('produtos')
      .insert([{ nome, descricao, preco, quantidade }])
      .select();

    if (error) throw error;

    res.status(201).json(data[0]);
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: 'Erro ao criar produto' });
  }
});
```

**Rota 4: PUT /produtos/:id — atualizar produto**

Pista: usa `.update({...}).eq('id', id).select()`. Pense no que retornar se o ID não existir.

**Rota 5: DELETE /produtos/:id — deletar produto**

Pista: usa `.delete().eq('id', id)`. Retorna 200 com mensagem de sucesso, ou 204 (No Content) sem corpo.

### 3.6 Como testar localmente

Antes de subir pra nuvem, **teste tudo localmente**. Opções:

**Opção A — curl no terminal:**
```bash
curl http://localhost:3000/produtos
curl http://localhost:3000/produtos/1
curl -X POST http://localhost:3000/produtos \
  -H "Content-Type: application/json" \
  -d '{"nome":"Teclado","descricao":"Mecânico","preco":300,"quantidade":10}'
```

**Opção B — Postman ou Insomnia** (mais visual, recomendo): você instala, cria as requisições, salva uma collection.

**Opção C — REST Client do VS Code:** instala a extensão, cria um arquivo `requests.http` e roda direto no editor.

### 3.7 Rodando o servidor

No `package.json`, adiciona um script:

```json
"scripts": {
  "start": "node index.js",
  "dev": "node --watch index.js"
}
```

`--watch` reinicia automaticamente quando você salva o arquivo (útil em desenvolvimento).

Rodar:
```bash
npm run dev
```

### 3.8 Commits sugeridos para essa fase

```bash
git checkout -b feat/backend-setup
git add .
git commit -m "feat(backend): configura express, supabase e dotenv"

git checkout -b feat/rotas-get
# implementa GET / e GET /produtos
git commit -m "feat(backend): adiciona rotas GET de produtos"

# E assim por diante para POST, PUT, DELETE
```

---

## Fase 4 — Frontend local

> **Tempo estimado:** 3-4 horas
> **Branch:** `feat/frontend-basico` e depois `feat/melhorias-frontend`

### 4.1 Estrutura

Dentro de `/frontend`:

```
frontend/
├── index.html        (página principal: lista + formulário)
├── style.css         (visual)
├── app.js            (lógica de fetch e DOM)
└── config.js         (URL do backend — facilita trocar depois)
```

### 4.2 Por que separar `config.js`

Esse é um truque pequeno mas muito importante. No `config.js`:

```js
const API_URL = 'http://localhost:3000';
```

Quando você for fazer deploy, vai mudar **uma única linha**, não vasculhar 50 arquivos atrás de `localhost:3000`.

### 4.3 Estrutura do `index.html`

Pense nas funcionalidades que precisa entregar:
1. **Listar** todos os produtos (tabela ou cards)
2. **Cadastrar** novo produto (formulário com nome, descricao, preco, quantidade)
3. **Buscar por ID** (input + botão + área de resultado)
4. **Editar** produto (formulário que preenche com dados existentes e salva)
5. **Deletar** produto (botão por item da lista)

Pode ser tudo numa página única (recomendo, mais simples) ou em páginas separadas. Sugestão de layout single-page:

```
┌─────────────────────────────────────┐
│         CRUD DE PRODUTOS            │
├─────────────────────────────────────┤
│  [Cadastrar Produto]                │
│   nome: [______]                    │
│   descrição: [______]               │
│   preço: [______]                   │
│   quantidade: [______]              │
│   [Salvar]                          │
├─────────────────────────────────────┤
│  [Consultar por ID]                 │
│   ID: [___] [Buscar]                │
│   Resultado: ...                    │
├─────────────────────────────────────┤
│  [Lista de Produtos]                │
│   ┌─────────────────────────────┐   │
│   │ id | nome | preço | ações   │   │
│   │  1 | ...  | ...   | [E] [D] │   │
│   │  2 | ...  | ...   | [E] [D] │   │
│   └─────────────────────────────┘   │
└─────────────────────────────────────┘
```

### 4.4 Padrão para chamadas `fetch`

A função base que você vai usar muito:

```js
async function listarProdutos() {
  try {
    const response = await fetch(`${API_URL}/produtos`);

    if (!response.ok) {
      throw new Error(`Erro HTTP: ${response.status}`);
    }

    const produtos = await response.json();
    renderizarLista(produtos);
  } catch (erro) {
    console.error('Falha ao listar produtos:', erro);
    alert('Não foi possível carregar a lista de produtos.');
  }
}
```

Para POST/PUT/DELETE, a sintaxe muda um pouco:

```js
async function criarProduto(dados) {
  const response = await fetch(`${API_URL}/produtos`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(dados)
  });
  return response.json();
}
```

### 4.5 Sobre as 4 melhorias avaliadas

Vou ser direto sobre cada uma porque cada décimo importa (lembre: sem as melhorias o teto é 7,0).

**Melhoria 1 (1,0 pt) — Campo descrição:** já vai estar integrado se você seguir o que escrevemos. Garanta que o `<input>` está no formulário, que o `app.js` envia ele no `body`, e que o backend salva no banco.

**Melhoria 2 (1,0 pt) — Botão Update funcional:** essa é a que mais aluno erra. O fluxo correto:

1. Cada linha da lista tem um botão "Editar"
2. Ao clicar, os campos do formulário se preenchem com os dados daquele produto
3. O formulário "muda de modo" — em vez de criar, ele vai atualizar
4. Você precisa guardar em algum lugar o ID do produto sendo editado (variável global, atributo `data-id` num campo escondido, etc.)
5. Ao submeter, faz `PUT /produtos/:id` em vez de `POST`

Truque elegante: ter uma variável `let produtoEditandoId = null` no escopo do app. Se for `null`, o submit faz POST. Se tiver valor, faz PUT. Após salvar, volta pra `null` e limpa o form.

**Melhoria 3 (0,5 pt) — Consulta por ID:** seção separada na página com um input e um botão. Ao clicar, chama `GET /produtos/:id`, mostra o resultado abaixo (ou alerta se não encontrar).

**Melhoria 4 (0,5 pt) — Visual:** não precisa ser obra de arte, mas saia do default do navegador. Sugestões que valem ponto:

- Fonte legível (use uma sans-serif decente: `system-ui`, `Segoe UI`, `Inter`)
- Cores consistentes (escolhe 2-3 cores e segue)
- Espaçamento generoso (padding e margin)
- Inputs com altura confortável (não os minúsculos default)
- Tabela com linhas zebradas
- Botões com hover

Frameworks que te dão visual decente com pouco esforço, se quiser usar:
- **Pico.css** — adicione 1 linha no `<head>` e tudo fica bonito sozinho
- **Tailwind via CDN** — mais flexível, mas exige escrever classes

### 4.6 Como testar localmente

Para servir HTML estático com chamadas para o backend, **não basta abrir o arquivo no navegador** (porque o navegador trata `file://` de jeito diferente e isso pode causar problemas de CORS).

Opções:

**Opção A — Live Server (extensão do VS Code):**
Instala a extensão "Live Server" → clica com botão direito no `index.html` → "Open with Live Server". Abre em `http://localhost:5500`.

**Opção B — http-server:**
```bash
npx http-server frontend -p 5500
```

Com o backend rodando em `:3000` e o frontend em `:5500`, eles vão conseguir conversar (graças ao `cors()` que você já adicionou no backend).

---

## Fase 5 — Deploy do backend na EC2

> **Tempo estimado:** 1-2 horas
> **Branch:** `feat/deploy-backend`

Aqui você revisita coisas que já fez no Projeto 2, mas com mais cuidado.

### 5.1 Lançar a primeira EC2

1. Console AWS → EC2 → "Launch instance"
2. Nome: `projeto3-backend`
3. AMI: **Ubuntu Server 24.04 LTS** (gratuito, familiar)
4. Tipo: **t2.micro** ou **t3.micro** (free tier)
5. Key pair: crie um novo OU use o que você já tem do Projeto 2. Salve o `.pem`.
6. Network settings → **Security Group** (atenção aqui):
   - Permitir **SSH (porta 22)** — do "Meu IP" para mais segurança, ou de qualquer lugar para conveniência
   - Permitir **Custom TCP** porta **3000** — de qualquer lugar (`0.0.0.0/0`), porque sua API vai precisar ser pública
7. Storage: 8GB padrão tá ok
8. Launch instance

### 5.2 Conectar via SSH

```bash
chmod 400 sua-chave.pem    # só na primeira vez, ajusta permissão
ssh -i sua-chave.pem ubuntu@IP-PUBLICO-DA-INSTANCIA
```

### 5.3 Setup da máquina

Dentro da VM:

```bash
sudo apt update
sudo apt install -y nodejs npm git
node --version    # confirma que instalou
```

> Se a versão do Node for muito antiga (Ubuntu costuma trazer versão velha), instale uma versão atual via NodeSource:
> ```bash
> curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
> sudo apt install -y nodejs
> ```

### 5.4 Clonar o repositório

```bash
cd ~
git clone https://github.com/matheusabporto/projeto3-cloud-crud-produtos.git
cd projeto3-cloud-crud-produtos/backend
npm install
```

### 5.5 Configurar variáveis de ambiente na VM

Lembra que o `.env` não foi pro GitHub? Então você precisa criar ele direto na VM:

```bash
nano .env
```

Cola o conteúdo:

```env
SUPABASE_URL=https://seu-projeto.supabase.co
SUPABASE_KEY=sua-chave-aqui
PORT=3000
```

`Ctrl+O`, Enter, `Ctrl+X` para salvar e sair.

### 5.6 Rodar com `pm2` (não com `node` direto)

Se você rodar `node index.js` direto, na hora que você fechar o SSH o servidor cai. Para manter rodando em background, use o **pm2** (process manager para Node).

```bash
sudo npm install -g pm2
pm2 start index.js --name backend-produtos
pm2 save
pm2 startup  # gera comando que você cola para o serviço subir junto com a VM
```

Verifica:

```bash
pm2 list
pm2 logs backend-produtos    # ver os logs em tempo real
```

### 5.7 Testar do mundo externo

Do seu computador local (não da VM):

```bash
curl http://IP-PUBLICO-DA-VM:3000/produtos
```

Deve retornar a lista. Se não retornar:
- Confere o Security Group (porta 3000 liberada?)
- Confere se o `pm2` está rodando (`pm2 list` na VM)
- Confere os logs (`pm2 logs`)
- Confere se o `.env` tá com as variáveis certas

### 5.8 Importante: anote o IP público

Você vai precisar dele para configurar o frontend. Algo como `http://54.123.45.67:3000`.

> **Cuidado:** o IP público da EC2 muda sempre que você para e religa a instância. Para evitar isso em projetos reais, usa-se Elastic IP. Para o trabalho, deixa como está, só não desligue a instância antes da apresentação.

---

## Fase 6 — Deploy do frontend na EC2

> **Tempo estimado:** 1-2 horas (mais a guerra contra o CORS)
> **Branch:** `feat/deploy-frontend`

### 6.1 Lançar a segunda EC2

Mesmo processo da anterior, mas:
- Nome: `projeto3-frontend`
- Security Group precisa liberar:
  - SSH (22) — pra você acessar
  - **HTTP (80)** — para o mundo acessar a página

### 6.2 Atualizar o `config.js` ANTES de subir

No seu computador local, edita o `frontend/config.js`:

```js
const API_URL = 'http://IP-PUBLICO-DA-VM-BACKEND:3000';
```

Faz commit e push:

```bash
git add frontend/config.js
git commit -m "chore(frontend): aponta para backend em produção"
git push
```

### 6.3 Setup da VM de frontend

SSH na nova VM. Você tem duas opções para servir arquivos estáticos:

**Opção A — http-server via Node (mais simples, mas amador):**

```bash
sudo apt update
sudo apt install -y nodejs npm git
sudo npm install -g http-server pm2

git clone https://github.com/matheusabporto/projeto3-cloud-crud-produtos.git
cd projeto3-cloud-crud-produtos/frontend

# Roda na porta 80 (precisa sudo para portas < 1024)
sudo pm2 start "http-server -p 80" --name frontend
sudo pm2 save
```

**Opção B — Nginx (profissional, vale o ponto extra de visual):**

```bash
sudo apt update
sudo apt install -y nginx git

git clone https://github.com/matheusabporto/projeto3-cloud-crud-produtos.git

# Copia os arquivos para o diretório do nginx
sudo cp -r projeto3-cloud-crud-produtos/frontend/* /var/www/html/

# Garante que o nginx está rodando
sudo systemctl start nginx
sudo systemctl enable nginx
```

> Opção B é mais robusta e impressiona mais o professor. Mas se for apertado de tempo, opção A serve.

### 6.4 Acessar do navegador

Abre `http://IP-PUBLICO-DA-VM-FRONTEND` no navegador. Deve carregar a página.

### 6.5 Quando o CORS atacar

Provavelmente a primeira chamada vai falhar. Abre o **DevTools (F12)** → aba **Console**. Vai ter erro vermelho com "blocked by CORS policy" ou similar.

Causas comuns e como corrigir:

1. **Esqueceu o `app.use(cors())` no backend** — adiciona, redeploy o backend.
2. **`config.js` com URL errada** — `http://` ou `https://`? Tem `:3000`? Tem barra no fim?
3. **Security Group bloqueando** — porta 3000 tá aberta?
4. **Backend caiu** — testa direto do navegador: `http://IP-BACKEND:3000/produtos`. Deve mostrar JSON.

> Não saia da Fase 6 enquanto **TODAS** as 5 operações (listar, buscar por id, criar, atualizar, deletar) não funcionarem pelo navegador acessando da VM de frontend. Esse é o momento de paciência.

---

## Fase 7 — Testes e apresentação

### 7.1 Checklist final

Antes de avisar o professor que está pronto, verifica:

- [ ] Frontend acessível pelo IP público da VM 2
- [ ] Listagem de produtos carrega do Supabase via backend
- [ ] Cadastro funciona (POST) e aparece na lista
- [ ] Botão editar preenche o formulário e salva (PUT)
- [ ] Consulta por ID retorna o produto certo
- [ ] Consulta por ID que não existe mostra mensagem de "não encontrado"
- [ ] Deletar remove da lista e do banco
- [ ] Visual minimamente decente
- [ ] README final atualizado com:
  - [ ] Arquitetura (com diagrama, se possível)
  - [ ] Tecnologias usadas
  - [ ] Como rodar localmente
  - [ ] URL pública (ou IP) do frontend e do backend
  - [ ] Print da aplicação rodando

### 7.2 README final caprichado

Vale a pena gastar 20 minutos no README. Para portfólio, isso é o que **decide se um recrutador clica no seu repo de novo**. Adiciona:

```markdown
## Demonstração

![Print da aplicação](docs/print-app.png)

## Tecnologias

- **Frontend:** HTML5, CSS3, JavaScript (vanilla)
- **Backend:** Node.js, Express
- **Banco de dados:** PostgreSQL via Supabase
- **Infraestrutura:** AWS EC2 (Ubuntu 24.04)
- **Versionamento:** Git, GitHub

## Arquitetura

[diagrama de arquitetura aqui]

## Como executar localmente

### Pré-requisitos
- Node.js 20+
- Conta no Supabase

### Backend
\`\`\`bash
cd backend
npm install
cp .env.example .env
# preencha o .env com suas credenciais do Supabase
npm run dev
\`\`\`

### Frontend
\`\`\`bash
cd frontend
npx http-server -p 5500
\`\`\`
```

### 7.3 Para a apresentação

- Tenha as duas VMs ligadas com antecedência (umas 30 minutos antes)
- Tenha o Supabase com dados de teste interessantes (não 50 produtos aleatórios, uns 5 bem feitos)
- Abra abas pré-prontas: frontend, painel do Supabase mostrando a tabela, DevTools para mostrar as chamadas de rede
- Pratique o fluxo: "vou criar um produto agora" → criar no frontend → mostrar que aparece na tabela do Supabase → editar → consultar por ID → deletar
- Se o professor perguntar de CORS, segurança, ou por que separou as VMs, você agora sabe responder

---

## Apêndices

### Apêndice A — Cheatsheet de comandos Git

```bash
# Começar uma nova feature
git checkout main
git pull
git checkout -b feat/nome-da-feature

# Salvar progresso
git add .
git commit -m "feat(escopo): mensagem clara"
git push origin feat/nome-da-feature

# Voltar pra main e atualizar
git checkout main
git pull

# Ver o histórico bonito
git log --oneline --graph --all
```

### Apêndice B — Troubleshooting comum

| Sintoma                                  | Causa provável                          | Solução                                            |
| ---------------------------------------- | --------------------------------------- | -------------------------------------------------- |
| `Cannot find module 'express'`           | Não rodou `npm install`                 | `npm install` na pasta do backend                  |
| `EADDRINUSE: address already in use`     | Porta 3000 ocupada                      | Mata o processo ou troca a porta no `.env`         |
| `blocked by CORS policy`                 | Backend sem `app.use(cors())`           | Adiciona o middleware e redeploy                   |
| Frontend abre mas API não responde       | Security Group bloqueando porta         | Libera porta 3000 no SG do backend                 |
| `relation "produtos" does not exist`     | Tabela não foi criada no Supabase       | Volta no Table Editor e cria                       |
| Servidor cai ao fechar SSH               | Rodou `node` direto, sem pm2            | Usa `pm2 start index.js`                           |
| IP da VM mudou                           | Stop/start na instância EC2             | Pega novo IP no console AWS, atualiza config.js    |

### Apêndice C — Glossário rápido

- **API** — Application Programming Interface. Conjunto de regras para um programa conversar com outro.
- **CRUD** — Create, Read, Update, Delete. Operações básicas com dados.
- **REST** — Estilo arquitetural de API baseado em recursos e verbos HTTP.
- **JSON** — JavaScript Object Notation. Formato de dados em texto.
- **Endpoint** — Cada URL específica que a API expõe (ex: `/produtos/:id`).
- **Middleware** — Função que processa requisições entre o pedido e a resposta (ex: `cors`, `express.json`).
- **CORS** — Política de segurança do navegador sobre requisições entre origens diferentes.
- **Status code** — Número que indica o resultado de uma requisição HTTP (200, 404, 500, etc).
- **Body** — Conteúdo enviado numa requisição (geralmente JSON em POST/PUT).
- **Headers** — Metadados da requisição (tipo de conteúdo, autenticação, etc).
- **Async/await** — Sintaxe moderna para lidar com código assíncrono em JavaScript.
- **EC2** — Elastic Compute Cloud. Serviço de máquinas virtuais da AWS.
- **Security Group** — Firewall virtual da AWS, controla portas abertas.
- **Supabase** — Backend-as-a-service. Principalmente PostgreSQL gerenciado.

---

## Última nota

Não se assuste com o tamanho desse guia. Você não vai usar tudo de uma vez — vai consultando seção por seção, fase por fase. **Vai dar certo.** É exatamente esse tipo de projeto que vai te dar segurança em back-end e cloud, e é um dos primeiros que tem cara de "coisa real" no seu portfólio.

Qualquer dúvida em qualquer fase, me chama. Bora.
