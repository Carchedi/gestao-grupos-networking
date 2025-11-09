# 📐 ARQUITETURA.md - Plataforma de Gestão de Networking

Este documento detalha a arquitetura proposta para a Plataforma de Gestão de Networking, cobrindo a **Stack Técnica**, **Modelagem de Dados** e a implementação do **Módulo Obrigatório (Fluxo de Admissão)** e do **Módulo Opcional B (Dashboard de Performance)**.

## 1. Visão Geral da Solução e Stack Técnica

### 1.1 Stack Técnica Escolhida e Justificativa

| Camada             | Tecnologia                              | Justificativa                                                                                                                                                     |
| :----------------- | :-------------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frontend**       | **Next.js + React + TypeScript**        | Escolha obrigatória. O Next.js facilita o desenvolvimento _fullstack_ e oferece otimizações de performance (SSR/SSG) e tipagem segura.                            |
| **Backend**        | **Next.js API Routes (Node.js)**        | Permite uma arquitetura coesa e eficiente, onde a lógica de _backend_ e _frontend_ convive no mesmo projeto.                                                      |
| **Banco de Dados** | **PostgreSQL**                          | Escolhido por sua **robustez**, excelente suporte a **consultas complexas de agregação** (essenciais para o Dashboard) e confiabilidade transacional.             |
| **Acesso ao DB**   | **`pg` (node-postgres) com SQL Direto** | Demonstra o domínio da comunicação direta e **segura** com o banco de dados, utilizando _prepared statements_ nativos (`$1`, `$2`) para evitar **SQL Injection**. |
| **Autenticação**   | **JWT (JSON Web Tokens)**               | Padrão _stateless_ (sem estado) para autenticação de APIs, ideal para proteger as rotas de Admin e Membros.                                                       |

### 1.2 Estrutura e Organização

O projeto segue uma estrutura modular:

- **`pages/api/*`**: Contém todos os _endpoints_ do _backend_. Rotas administrativas estarão em `pages/api/admin/*`.
- **`lib/db.ts`**: Módulo centralizado que gerencia o **Pool de Conexões** do `pg`.
- **`lib/auth.ts`**: Funções para _middleware_ de autenticação e manipulação de JWTs.
- **`lib/utils.ts`**: Funções auxiliares (ex: geração de `UUID` para tokens).

---

## 2. Modelagem de Dados (SQL Direto)

As tabelas serão criadas diretamente no PostgreSQL.

### 2.1 Estrutura das Tabelas Principais (Esquema Simplificado)

| Tabela             | Descrição                                                                | Campos Chave e Tipos (SQL)                                                                                                                                                       |
| :----------------- | :----------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`applications`** | Registra a intenção de participação e o _status_ de admissão.            | `id` (SERIAL PRIMARY KEY), `email` (VARCHAR UNIQUE), `status` (VARCHAR), `invitation_token` (UUID UNIQUE).                                                                       |
| **`members`**      | Membros ativos. É populada após a aprovação e cadastro final.            | `id` (SERIAL PK), `application_id` (INT REFERENCES applications), `email` (VARCHAR UNIQUE), `is_admin` (BOOLEAN), `status` (VARCHAR, Active/Inactive), `created_at` (TIMESTAMP). |
| **`thanks`**       | Registra os "Obrigados" dados entre membros. Essencial para o Dashboard. | `id` (SERIAL PK), `giver_id` (INT REFERENCES members), `receiver_id` (INT REFERENCES members), `description` (TEXT), `created_at` (TIMESTAMP).                                   |
| **`referrals`**    | Registra as indicações de negócios feitas por membros.                   | `id` (SERIAL PK), `referrer_id` (INT REFERENCES members), `referred_email` (VARCHAR), `status` (VARCHAR), `created_at` (TIMESTAMP).                                              |

---

## 3. Módulo Obrigatório: Fluxo de Admissão de Membros

### 3.1 Fluxo e Endpoints de Backend

| Etapa                 | Endpoint                              | Método | Lógica de Backend (SQL)                                                                                                                         |
| :-------------------- | :------------------------------------ | :----- | :---------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Intenção**       | `/api/applications`                   | `POST` | Executa **`INSERT INTO applications`** com `status = 'PENDING'`.                                                                                |
| **2. Admin Listar**   | `/api/admin/applications`             | `GET`  | **`SELECT`** de aplicações. Requer **Autenticação JWT/Admin**.                                                                                  |
| **3. Admin Aprovar**  | `/api/admin/applications/[id]/status` | `POST` | Gera **`UUID`** como token de convite. **`UPDATE applications SET status='APPROVED', invitation_token=...`**.                                   |
| **4. Cadastro Final** | `/api/register/[token]`               | `POST` | Valida token (data e existência). Se válido, **`INSERT INTO members`** (movendo o registro para a tabela de membros ativos) e invalida o token. |

---

## 4. Módulo Opcional B: Dashboard de Performance

### 4.1 Estratégia de Agregação de Dados

O Dashboard será alimentado por um _endpoint_ único, otimizado para a **agregação e _date-truncation_** do PostgreSQL.

- **Endpoint:** `GET /api/admin/dashboard` (Protegido por JWT de Admin)

### 4.2 Indicadores e SQL Estratégico

| Indicador                       | Consulta Estratégica                                                                     |
| :------------------------------ | :--------------------------------------------------------------------------------------- |
| **Total de Membros Ativos**     | `SELECT COUNT(id) FROM members WHERE status = 'Active';`                                 |
| **Total de "Obrigados" no Mês** | `SELECT COUNT(id) FROM thanks WHERE created_at >= date_trunc('month', CURRENT_DATE);`    |
| **Total de Indicações no Mês**  | `SELECT COUNT(id) FROM referrals WHERE created_at >= date_trunc('month', CURRENT_DATE);` |

---

## 5. Boas Práticas e Testes

### 5.1 Estratégia de Testes (30% da Avaliação)

- **Testes Unitários:** Focados em `lib/utils.ts` e lógica de validação de dados.
- **Testes de Integração (Prioridade):** O foco é validar as **API Routes e a comunicação SQL**. Será utilizado um ambiente de teste para garantir que:
  1.  O _status_ HTTP retornado está correto.
  2.  O **SQL `SELECT`** após cada `POST/UPDATE` confirma a persistência e manipulação correta dos dados no banco.

### 5.2 Boas Práticas Gerais

- **Segurança:** Uso de _prepared statements_ com _placeholders_ em **todas** as interações com o DB.
- **Versionamento:** Histórico de _commits_ claro e atômico.
