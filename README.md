# IOSPRIVATE API

API de gerenciamento de licenças com Node.js, Express e SQLite (`better-sqlite3`). O banco de produção é o único arquivo `/var/data/iosprivate.sqlite`, em Persistent Disk do Render. Não há Supabase, Firebase, MongoDB, PostgreSQL ou serviço externo de armazenamento.

## Executar localmente

Requer Node.js 20+. Copie `.env.example` para `.env`, defina uma senha forte em `ADMIN_PASSWORD` e um `JWT_SECRET` com pelo menos 32 caracteres. Depois:

```bash
npm ci
npm start
# http://localhost:10000/health
```

O banco local padrão fica em `data/iosprivate.sqlite`; em produção use sempre `/var/data/iosprivate.sqlite`. O primeiro administrador é configurado exclusivamente por variáveis de ambiente. Não existe credencial padrão segura embutida no código.

## Publicar no GitHub

```bash
git init
git add .
git commit -m "feat: initial IOSPRIVATE license API"
gh repo create iosprivate-api --public --source=. --remote=origin --push
```

Nunca faça commit de `.env`, tokens, senhas ou arquivos `.sqlite`.

## Render com Persistent Disk

O `render.yaml` declara um Web Service Node no plano `starter` (necessário para Persistent Disk), build `npm ci`, start `npm start`, health check `/health`, variável `SQLITE_PATH=/var/data/iosprivate.sqlite` e disco de 1 GB montado em `/var/data`. O plano Free não mantém disco persistente.

1. No Render, escolha **New > Blueprint** e conecte o repositório GitHub; ou crie um Web Service apontando para este repositório.
2. Confirme o nome `iosprivate-api`, plano pago compatível com Persistent Disk e a região desejada.
3. Se não usar Blueprint, configure Build Command `npm ci`, Start Command `npm start`, Health Check Path `/health` e adicione um Persistent Disk montado em `/var/data`.
4. Em Environment, configure `ADMIN_USERNAME`, `ADMIN_PASSWORD` e um `JWT_SECRET` aleatório forte. `SQLITE_PATH=/var/data/iosprivate.sqlite` não deve ser alterado. `CORS_ORIGIN` pode ser ajustado para a origem do seu site; use `*` somente quando necessário.
5. Faça o primeiro deploy. A URL será fornecida pelo Render como `https://<nome-do-servico>.onrender.com`; não há URL fixa neste repositório até que o serviço seja criado.
6. Teste: `curl https://<seu-servico>.onrender.com/health`.

A inicialização usa migrações idempotentes e nunca apaga o banco. Em atualizações, mantenha o mesmo Persistent Disk e não use scripts que removam `/var/data`.

## API pública

`POST /v1/bind`, `POST /v1/validate` e `POST /v1/unbind` recebem `{ "key": "...", "device": "identificador-do-cliente" }`. O identificador é compatível com navegador/PWA, mas não é um HWID físico inviolável e pode ser falsificado; o servidor não confia em estado enviado pelo cliente.

Exemplo:

```bash
curl -X POST https://SEU-DOMINIO.onrender.com/v1/bind \
  -H 'Content-Type: application/json' \
  -d '{"key":"IOSPRIVATE-DAILY-...","device":"pwa-device-id"}'
```

Clientes HTML/JavaScript, PWA, Node.js, Scriptable e outros clientes HTTP devem usar somente estes endpoints públicos. Nunca inclua o token administrativo em código cliente.

## Painel e endpoints administrativos

Abra `https://SEU-DOMINIO.onrender.com/`, entre com `ADMIN_USERNAME`/`ADMIN_PASSWORD` e gere, pesquise, bloqueie, reative, desvincule ou exclua licenças. O painel usa a API real, sem dados fictícios.

Endpoints protegidos por `Authorization: Bearer <token>`: `GET /admin/keys`, `POST /admin/keys`, `GET /admin/keys/:id`, `PATCH /admin/keys/:id/status`, `POST /admin/keys/:id/unbind`, `DELETE /admin/keys/:id`, `GET /admin/stats`. O login é `POST /admin/auth/login`.

Criação: `{ "type":"daily", "duration":7, "username":"cliente" }`. Tipos: `hourly`, `daily`, `weekly`, `monthly` (30 dias) e `perm`. As keys usam `crypto.randomBytes`; expiração é calculada no servidor. A vinculação é protegida por transação SQLite, impedindo ativação concorrente em dispositivos diferentes.

## Backup e restauração

Faça backup com a aplicação parada, ou em janela controlada, para garantir um snapshot consistente:

```bash
npm run backup -- ./backups
# restauração: pare a aplicação, valide o arquivo e execute:
npm run restore -- ./backups/iosprivate-AAAA.sqlite
```

No Render, copie o arquivo de backup para armazenamento seguro fora do serviço. Nunca substitua o banco de produção durante uma escrita e nunca restaure sem manter uma cópia `.before-restore-*`. O Persistent Disk não substitui backups externos.

## Testes

Os testes usam `:memory:` e nunca o banco de produção:

```bash
npm test
```

Os testes cobrem inicialização, criação, validação, expiração, bloqueio/reativação, bind/unbind, autenticação, entradas inválidas, concorrência lógica e persistência dentro do ciclo do banco. Antes de cada deploy, execute a suíte e verifique `/health`.
