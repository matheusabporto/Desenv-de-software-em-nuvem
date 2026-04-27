# Comandos — Projeto 2 Cloud

## EC2 — preparar ambiente

```bash
sudo su

curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.3/install.sh | bash

source ~/.bashrc

. ~/.nvm/nvm.sh

nvm install --lts

node --version

npm --version
```

## EC2 — clonar e rodar o projeto

```bash
git clone https://github.com/seu-usuario/nome-do-repo.git

cd nome-do-repo

node server.js
```

## Git — alterar e subir nova versão

```bash
git add .

git commit -m "atualiza rotas"

git push origin main
```
