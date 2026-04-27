# Projeto 2 — Desenvolvimento de Software em Nuvem

Aplicação web em Node.js com múltiplas rotas, versionada no GitHub e com deploy realizado na AWS e no Railway.

**Disciplina:** Desenvolvimento de Software em Nuvem  
**Professor:** Américo Sampaio  
**Curso:** Análise e Desenvolvimento de Sistemas — Unifor  

---

## Sobre o projeto

Aplicação web desenvolvida com Node.js no modelo request-response, com pelo menos 3 rotas distintas respondendo com HTML. O projeto contempla versionamento com Git/GitHub e deploy em duas plataformas de nuvem: Amazon AWS e Railway.

---

## Tecnologias utilizadas

- Node.js
- Git e GitHub
- Amazon AWS (EC2)
- Railway

---

## Estrutura do projeto

```
/
├── server.js         # arquivo principal da aplicação
├── .gitignore        # template Node.js
└── README.md
```

---

## Rotas disponíveis

| Rota | Descrição |
|------|-----------|
| `/`  | Página inicial |
| `/sobre` | Página sobre o projeto |
| `/contato` | Página de contato |

---

## Como executar localmente

**Pré-requisitos:** Node.js instalado.

```bash
# 1. Clonar o repositório
git clone https://github.com/seu-usuario/nome-do-repositorio.git

# 2. Entrar na pasta
cd nome-do-repositorio

# 3. Iniciar a aplicação
node server.js
```

Acesse em: `http://localhost:3000`

---

## Deploy

### Amazon AWS (EC2)

A aplicação está disponível em: `http://<ip-publico-da-instancia>:3000`

Passos realizados:
1. Criação de instância EC2
2. Instalação do NVM e Node.js
3. Clone do repositório via Git
4. Execução da aplicação

### Railway

A aplicação está disponível em: `https://<url-gerada-pelo-railway>`

Passos realizados:
1. Conexão do repositório GitHub ao Railway
2. Deploy automático a partir da branch principal

---

## Fluxo Git utilizado

```bash
# Clonar o repositório remoto
git clone https://github.com/seu-usuario/nome-do-repositorio.git

# Após alterações no código
git add .
git commit -m "descrição da alteração"
git push origin main
```

---

## Escopo do projeto (critérios de avaliação)

| Item | Descrição | Pontos |
|------|-----------|--------|
| 1 | Aplicação Node.js com 3+ rotas respondendo em HTML | 2,5 |
| 2 | Operações Git/GitHub: clone, commit e push | 2,5 |
| 3 | Deploy na AWS ou Oracle Cloud | 2,5 |
| 4 | Deploy no Railway a partir do GitHub | 2,5 |
