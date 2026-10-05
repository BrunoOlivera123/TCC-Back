# SOSCAR — Backend (API)

API do **SOS Car** (TCC "Sistema de SOS para Veículos Automotores"): Node.js + Express + MySQL.
Cuida de autenticação, autorização, regras de negócio, banco de dados, e-mail (código de recuperação de senha) e validação do Google Login.
A interface fica no projeto separado **SOSCAR-FRONTEND**, que conversa com esta API por HTTP/REST.

```
SOSCAR-FRONTEND  ──HTTP/REST (JSON, Bearer token)──▶  SOSCAR-BACKEND  ──▶  MySQL
```

## Requisitos

- Node.js 18 ou superior
- MySQL 8 (opcional em desenvolvimento: sem banco o servidor usa modo em memória e os dados somem ao reiniciar)

## Instalação

```bash
npm install
cp .env.example .env     # Windows: copy .env.example .env
npm run dev              # ou: npm start
```

Por padrão a API sobe em **https://localhost:3000** com um certificado autoassinado de desenvolvimento (criado em `.certs/`). Na primeira vez, abra `https://localhost:3000/health` no navegador e aceite o aviso do certificado; senão o frontend não consegue chamar a API. Para HTTP puro (por exemplo atrás de um proxy que já faz HTTPS), use `USAR_HTTPS=0`.

## Configuração (`.env`)

Veja `.env.example` (todas as variáveis são comentadas). Resumo:

| Variável | Para quê |
|---|---|
| `PORT`, `HOST`, `USAR_HTTPS`, `TLS_CERT_FILE`, `TLS_KEY_FILE`, `TRUST_PROXY` | Servidor |
| `CORS_ORIGINS` | Endereços do frontend autorizados (lista separada por vírgulas, sem `*`) |
| `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` | Banco MySQL |
| `ADMIN_EMAIL`, `ADMIN_SENHA` / `ADMIN_SENHA_HASH` | Conta do administrador (padrão `admin@soscar.com` / `admin123` — **troque antes de publicar**) |
| `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM` | Envio do código de recuperação de senha |
| `CODIGO_REDEFINICAO_SEGREDO` | Chave (HMAC) que protege os códigos de 6 dígitos no banco |
| `MOSTRAR_CODIGO_REDEFINICAO_NO_CONSOLE` | Só desenvolvimento: sem SMTP, imprime o código no terminal |
| `GOOGLE_CLIENT_ID`, `GOOGLE_DOMINIO_PERMITIDO` | Google Login (o ID do cliente é público; não há Client Secret) |
| `OSM_USER_AGENT`, `BUSCARCEP_CHAVE` | Integrações externas opcionais |

**CORS:** só as origens listadas em `CORS_ORIGINS` podem ler as respostas. Sem a variável, em desenvolvimento ficam liberados `http(s)://localhost:5173` e `http(s)://127.0.0.1:5173`; com `NODE_ENV=production` nenhuma origem é liberada até você configurá-la. A autenticação usa `Authorization: Bearer`, sem cookies.

## Banco de dados

- As tabelas são **criadas e atualizadas automaticamente** por `src/db.js` na primeira execução.
- `database/schema.sql` é o modelo conceitual do TCC (não precisa ser executado).
- `database/migrations/` documenta as alterações de esquema (bancos antigos podem precisar aplicar `001_integridade.sql` uma vez, com o servidor parado e backup feito).
- Use uma instância da API por banco.

## Estrutura

```
src/
  server.js            rotas e regras de negócio
  db.js                conexão/persistência MySQL (com fallback em memória)
  auth-middleware.js   sessão por token (cliente, prestador, admin)
  google-auth.js       validação local do ID token do Google (+ nonce)
  email.js             envio do código de redefinição de senha
  geocodificacao.js    endereço -> coordenadas (Nominatim) com cache
  buscarcep.js         consulta de CEP (opcional)
  utils/               CPF e distância (Haversine)
database/              schema.sql e migrations/
tests/                 testes (node --test)
docs/                  anotações do projeto
```

## API

Todas as rotas de dados ficam sob `/api` e respondem JSON (`{ "erro": "mensagem" }` em caso de erro). Rotas protegidas exigem `Authorization: Bearer <token>`, obtido no login/cadastro (sessão em memória, 24 h).

| Área | Rotas |
|---|---|
| Saúde | `GET /health`, `GET /` |
| Público | `GET /api/categorias` |
| Autenticação | `POST /api/auth/registrar`, `POST /api/auth/login`, `POST /api/auth/logout`, `GET /api/auth/me`, `PATCH /api/auth/atualizar`, `POST /api/auth/completar-perfil`, `POST /api/auth/login-admin` |
| Google | `GET /api/auth/google/config`, `GET /api/auth/google/nonce`, `POST /api/auth/google` |
| Recuperação de senha | `POST /api/auth/esqueci-senha` → `POST /api/auth/validar-codigo` → `POST /api/auth/redefinir-senha` |
| Localização (cliente) | `GET /api/localizacao/geocodificar`, `GET /api/localizacao/reverse` |
| Chamados | `POST /api/chamados`, `GET /api/chamados/atual`, `GET /api/chamados/historico`, `GET /api/chamados/disponiveis`, `PATCH /api/chamados/:id/localizacao`, `POST /api/chamados/:id/{aceitar,iniciar,concluir,cancelar,cancelar-prestador,cancelamento-visto,avaliacao}` |
| Prestador | `PATCH /api/prestador/disponibilidade`, `GET /api/prestador/me/avaliacoes` |
| Notificações | `GET /api/notificacoes`, `POST /api/notificacoes/lidas`, `POST /api/notificacoes/:id/lida` |
| Administração | `GET /api/admin/{estatisticas,usuarios,chamados,categorias}`, `POST /api/admin/prestadores/:id/aprovar`, `POST /api/admin/chamados/:id/cancelar`, `PATCH /api/admin/categorias/:id` |

### Recuperação de senha

1. `esqueci-senha` recebe o e-mail, gera um código de 6 dígitos e o envia por e-mail.
2. `validar-codigo` confere o código (com limite de tentativas e validade) e devolve um `resetToken` temporário.
3. `redefinir-senha` grava a nova senha e derruba as outras sessões da conta.

## Produção

- `NODE_ENV=production`, `CORS_ORIGINS` com o(s) domínio(s) do frontend, `TRUST_PROXY` se houver proxy, `ADMIN_SENHA_HASH`, SMTP e `CODIGO_REDEFINICAO_SEGREDO` definidos.
- Use HTTPS válido (certificado real ou proxy com `USAR_HTTPS=0`).
- No Google Cloud Console, cadastre o endereço do **frontend** em "Origens JavaScript autorizadas".
