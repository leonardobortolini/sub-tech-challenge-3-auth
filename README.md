# TRABALHO SUBSTITUTIVO DE TECH CHALLENGE - FASE 3 AUTH

Serviço de autenticação e autorização da solução, implementado utilizando Keycloak e PostgreSQL.

O projeto é mantido de forma independente da API de veículos, garantindo a separação entre os dados de autenticação dos usuários e os dados transacionais da aplicação.

## Projeto relacionado

A API responsável pelo cadastro, gerenciamento e venda de veículos está em um repositório separado:

- [sub-tech-challenge-3-api](https://github.com/leonardobortolini/sub-tech-challenge-3-api)

Para executar a solução completa localmente, recomenda-se iniciar este projeto (Auth) antes da API.

---

## Arquitetura

```text
┌─────────────────────────────────┐
│          Projeto Auth           │
│                                 │
│       Keycloak + PostgreSQL     │
│                                 │
│  Usuários / Roles / JWT         │
└────────────────┬────────────────┘
                 │
                 │ JWT
                 ▼
┌─────────────────────────────────┐
│           Projeto API           │
│                                 │
│       FastAPI + PostgreSQL      │
│                                 │
│  Veículos / Vendas / Pagamentos │
└─────────────────────────────────┘
```

O projeto Auth possui seu próprio banco PostgreSQL e não compartilha dados de usuários com o banco utilizado pela API.

O Keycloak é responsável por:

- Autenticação dos usuários
- Gerenciamento dos usuários
- Gerenciamento das roles
- Emissão dos tokens JWT

A API utiliza os tokens emitidos pelo Keycloak para autenticar e autorizar as requisições.

---

## Tecnologias

- Keycloak 26.3.5
- PostgreSQL 15
- Docker
- Docker Compose
- Kubernetes
- Minikube
- GitHub Actions

---

## Configuração

O realm utilizado pela solução é:

```text
revenda
```

A configuração do realm está versionada no projeto em:

```text
keycloak/revenda-realm.json
```

O projeto utiliza as seguintes roles:

- `admin`
- `cliente`

Os usuários utilizados durante os testes são criados manualmente através da interface administrativa do Keycloak.

---

# Executando localmente

## Pré-requisitos

Para executar o projeto localmente utilizando Docker:

- Docker
- Docker Compose

Para executar utilizando Kubernetes:

- Docker
- kubectl
- Minikube

---

## 1. Iniciar o Auth com Docker Compose

Na raiz do projeto, execute:

```bash
docker compose up -d
```

Verifique os containers:

```bash
docker compose ps
```

O ambiente irá iniciar:

- Keycloak
- PostgreSQL utilizado pelo Keycloak
- Rede Docker utilizada pela solução

O Keycloak ficará disponível em:

```text
http://localhost:8080
```

---

## 2. Verificar o realm

Após iniciar o ambiente, valide se o realm `revenda` está disponível:

```bash
curl -s   http://localhost:8080/realms/revenda/.well-known/openid-configuration
```

A resposta deve conter as informações do OpenID Connect, incluindo o
`issuer` e o endpoint de emissão de tokens.

Exemplo:

```json
{
  "issuer": "http://localhost:8080/realms/revenda",
  "token_endpoint": "http://localhost:8080/realms/revenda/protocol/openid-connect/token"
}
```

---

# Usuários e Roles

O projeto utiliza as seguintes roles:

| Role | Finalidade |
|---|---|
| `admin` | Operações administrativas relacionadas aos veículos |
| `cliente` | Operações de compra de veículos |

Os usuários devem ser criados manualmente através da interface administrativa
do Keycloak.

Acesse:

```text
http://localhost:8080
```

Selecione o realm:

```text
revenda
```

Em seguida, crie os usuários e atribua as roles necessárias.

> A criação dos usuários e a atribuição das roles não são realizadas automaticamente pela aplicação.

---

# Obtenção de token

A emissão dos tokens é realizada pelo Keycloak.

O endpoint utilizado é:

```text
POST /realms/revenda/protocol/openid-connect/token
```

Exemplo:

```bash
curl -X POST   http://localhost:8080/realms/revenda/protocol/openid-connect/token   -H "Content-Type: application/x-www-form-urlencoded"   -d "client_id=revenda-api"   -d "username=USUARIO"   -d "password=SENHA"   -d "grant_type=password"
```

A resposta conterá um `access_token`:

```json
{
  "access_token": "eyJ...",
  "expires_in": 1800,
  "token_type": "Bearer"
}
```

O token deve ser utilizado pela API através do header:

```text
Authorization: Bearer <TOKEN>
```

Para o fluxo completo de utilização do token e execução da API, consulte o
README do projeto [sub-tech-challenge-3-api](https://github.com/leonardobortolini/sub-tech-challenge-3-api).

---

# Executando com Kubernetes

O projeto também possui manifests Kubernetes na pasta:

```text
k8s/
```

## 1. Inicializar o Minikube

```bash
minikube start
```

## 2. Aplicar os manifests

```bash
kubectl apply -f k8s/
```

Os manifests criam os recursos necessários para o serviço de autenticação,
incluindo:

- Namespace
- ConfigMap
- Secret
- PostgreSQL
- Service do PostgreSQL
- Keycloak
- Service do Keycloak

Verifique os pods:

```bash
kubectl get pods -n revenda
```

Verifique os serviços:

```bash
kubectl get svc -n revenda
```

O Keycloak será disponibilizado através do serviço Kubernetes configurado nos
manifests.

---

# CI — Integração Contínua

O projeto utiliza GitHub Actions para validação automática das alterações.

O CI realiza validações dos arquivos do projeto, incluindo:

- Validação dos manifests Kubernetes com `yamllint`
- Validação do JSON do realm
- Validação da configuração do Docker Compose

As alterações são realizadas utilizando Pull Requests.

O pipeline é executado para Pull Requests e alterações nas branches:

- `dev`
- `main`

---

# CD — Deploy Contínuo

O projeto possui deploy automatizado utilizando GitHub Actions e Kubernetes.

O CD é executado após alterações nas branches:

- `dev`
- `main`

O pipeline:

1. Inicia um ambiente Minikube
2. Aplica os manifests Kubernetes
3. Aguarda o PostgreSQL
4. Aguarda o deployment do Keycloak
5. Verifica os pods
6. Valida a disponibilidade do realm através do endpoint OpenID Connect

---

# CI/CD e Pull Requests

Todas as alterações do projeto devem passar pelo fluxo de desenvolvimento utilizando Pull Requests.

Fluxo adotado:

```text
Feature Branch
      │
      ▼
Pull Request
      │
      ▼
CI
      │
      ├── YAML
      ├── JSON
      └── Docker Compose
      │
      ▼
Merge
      │
      ▼
dev / main
      │
      ▼
CD
      │
      ▼
Kubernetes
      │
      ▼
Keycloak
```

Essa abordagem permite validar as alterações automaticamente antes da integração e realizar o deploy de forma automatizada.

---

# Variáveis de ambiente

Utilize o arquivo:

```text
.env.example
```

como referência para criar o arquivo `.env`.

As variáveis utilizadas pelo ambiente incluem as configurações do PostgreSQL
e as credenciais administrativas iniciais do Keycloak.

---

# Observações

- Este projeto é responsável exclusivamente pela autenticação e autorização.
- O serviço deve ser iniciado antes da API durante a execução local completa.
- O Keycloak utiliza o realm `revenda`.
- Os usuários e roles utilizados nos testes são criados manualmente pela
  interface administrativa do Keycloak.
- A API não compartilha o banco de dados utilizado pelo Auth.
- O Keycloak emite os JWTs utilizados pela API para autenticação e autorização.
- Para executar o fluxo completo da solução, consulte também o README do
  [sub-tech-challenge-3-api](https://github.com/leonardobortolini/sub-tech-challenge-3-api).
