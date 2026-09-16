# Arquitetura-n8n

# Documentação da API PZaaS - Cadastro e Identidade

Este documento apresenta a documentação oficial do microsserviço de Cadastro e Identidade (Serviço 08). Ele é o encarregado de gerenciar usuários, controlar saldos (carteiras), realizar autenticações, validar tokens e comunicar-se com o banco de dados dentro da arquitetura distribuída da pizzaria.

---

## 1. Visão Geral

O *Serviço 08* atua fornecendo a identidade, os perfis de acesso e a gestão financeira (carteira) dos clientes no ecossistema PZaaS. Construído utilizando o *n8n* (no servidor compartilhado da turma) com integração ao *PostgreSQL (Neon Tech)*, suas funções incluem autenticar credenciais, registrar novas contas, administrar os saldos e retornar as informações de cadastro baseadas em tokens UUID gerados no próprio banco.

## 2. Tecnologias Utilizadas

•⁠ ⁠*n8n*: Responsável por orquestrar e executar a lógica de negócios e os endpoints.
•⁠ ⁠*PostgreSQL (Neon Tech)*: Banco de dados relacional usado para armazenar os dados de perfil e o controle financeiro (através da tabela `usuarios_pzaas`, que contém a coluna `saldo`).
•⁠ ⁠*Postman*: Ferramenta de apoio para realizar as chamadas e testes das rotas.

---

## 3. Arquitetura e Endpoints

O serviço disponibiliza quatro rotas principais, que funcionam de maneira isolada (sob o caminho `cadastro-v2`) no ambiente de produção.

### `GET /cadastro-v2/health` (Healthcheck)

•⁠ ⁠*Propósito*: Checar se o microsserviço está operante.
•⁠ ⁠*Retorno de Sucesso (`HTTP 200 OK`)*: Devolve o payload `{"status": "tudo certo"}`.

### `POST /cadastro-v2/cadastro` (Registro de Usuário)

•⁠ ⁠*Propósito*: Cadastrar um novo usuário e emitir seu token de acesso. A carteira do cliente é criada automaticamente no banco de dados com o saldo inicial de `0.00`.
•⁠ ⁠*Corpo da Requisição (JSON)*:

```json
{
"nome": "Joao Del",
"email": "joao@del.tech",
"senha": "123"
}

```

•⁠ ⁠*Retorno de Sucesso (`HTTP 201 Created`)*:

```json
{
"mensagem": "Usuario criado com sucesso",
"token": "uuid-gerado-pelo-banco"
}

```

### `POST /cadastro-v2/login` (Autenticação)

•⁠ ⁠*Propósito*: Checar as credenciais informadas e retornar o token de acesso do usuário.
•⁠ ⁠*Corpo da Requisição (JSON)*:

```json
{
"email": "joao@del.tech",
"senha": "123"
}

```

•⁠ ⁠*Retorno de Sucesso (`HTTP 200 OK`)*: Entrega o token de acesso no formato `{"token": "uuid-do-banco"}`.
•⁠ ⁠*Retorno de Erro (`HTTP 401 Unauthorized`)*: Devolve `{"erro": "Credenciais invalidas"}` caso as informações não constem no banco.

### `POST /cadastro-v2/perfil` (Verificação de Perfil e Saldo)

•⁠ ⁠*Propósito*: A partir de um token válido, devolver as informações detalhadas do usuário, incluindo o saldo de sua carteira, que é controlado exclusivamente por este domínio.
•⁠ ⁠*Corpo da Requisição (JSON)*:

```json
{
"token": "uuid-do-banco"
}

```

•⁠ ⁠*Retorno de Sucesso (`HTTP 200 OK`)*:

```json
{
"nome": "Joao Del",
"perfil": "cliente",
"email": "joao@del.tech",
"saldo": 150.00
}

```

•⁠ ⁠*Retorno de Erro (`HTTP 401 Unauthorized`)*: Exibe `{"erro": "Token invalido ou nao encontrado"}` se o token fornecido não pertencer a nenhum usuário.

---

## 4. Segurança e Regras Globais

Todas as rotas exigem autenticação do tipo `headerAuth` no n8n. No ambiente de produção, as requisições devem seguir estas especificações de cabeçalho:

•⁠ ⁠`x-api-key`: Chave global de autorização (valor: `turma2026`).
•⁠ ⁠`x-pedido-id`: ID de rastreamento (opcional).

Como os fluxos de cadastro e login podem ocorrer antes de um pedido ser formalizado, a ausência do header `x-pedido-id` não bloqueia a requisição nem causa erros. Caso seja enviado, ele é automaticamente repassado para a API de logs via Header.

---

## 5. Observabilidade e Registro de Logs

Seguindo as novas diretrizes da arquitetura distribuída, o microsserviço conta com *observabilidade assíncrona e tolerante a falhas* conectada à Logger API:

•⁠ ⁠O número *8* identifica este serviço no ecossistema global.
•⁠ ⁠As operações bem-sucedidas (`CRIAR_USUARIO`, `LOGIN` e `CONSULTAR_PERFIL`) enviam logs estruturados em background para a rota `POST /v1/log` do serviço da turma.
•⁠ ⁠O identificador de rastreio (`x-pedido-id`) é transmitido de forma nativa para a Logger API usando os cabeçalhos HTTP (substituindo o envio tradicional via body).
•⁠ ⁠*Fallback (Tolerância a Falhas)*: O disparo de logs utiliza a diretiva `onError: continueRegularOutput`. Isso assegura explicitamente que, caso o Logger caia (como em erros 404 ou 503), a falha seja contornada pelo sistema, permitindo que a jornada do cliente e a operação de identidade sejam concluídas sem interrupções.

---

## 6. Testes Práticos (Guia Postman)

Para validar o funcionamento, utilize a URL de produção do servidor da turma. Certifique-se de configurar a aba *Headers* nas suas requisições:

•⁠ ⁠`Content-Type`: `application/json` (para chamadas do tipo POST)
•⁠ ⁠`x-api-key`: `turma2026` (chave requerida pela proteção `headerAuth`)
•⁠ ⁠`x-pedido-id`: `12345` (opcional; recomenda-se testar com e sem este campo)

*Links de Acesso:*

1.⁠ ⁠*Checar Saúde*: `GET` [Abrir endpoint de healthcheck](https://www.google.com/search?q=https://pzaas.online/webhook/cadastro-v2/health)
2.⁠ ⁠*Registrar Usuário*: `POST` [Abrir endpoint de cadastro](https://www.google.com/search?q=https://pzaas.online/webhook/cadastro-v2/cadastro)
3.⁠ ⁠*Efetuar Login*: `POST` [Abrir endpoint de login](https://www.google.com/search?q=https://pzaas.online/webhook/cadastro-v2/login)
4.⁠ ⁠*Ver Perfil*: `POST` [Abrir endpoint de perfil](https://www.google.com/search?q=https://pzaas.online/webhook/cadastro-v2/perfil)

As rotas de cadastro, login e perfil interagem diretamente com a tabela `usuarios_pzaas`. O processo de criação de conta define o perfil como `cliente` por padrão e inicia a carteira zerada (via regra `DEFAULT` configurada no banco). O token nasce da função `gen_random_uuid()`. Já o login realiza a validação de `email` e `senha` fazendo um match direto no banco de dados (o workflow atual não aplica mecanismos adicionais de proteção/criptografia na senha).
