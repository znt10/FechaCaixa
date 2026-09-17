<div align="center">

# FechaCaixa

**O fechamento de caixa das lojas de uma empresa — lançado no celular, conferido no painel.**

A loja fecha o turno pelo celular, sem login. A gerência vê quanto cada loja
fez, o que falta lançar, para onde foi o dinheiro, e corrige o que veio errado.

[![Backend](https://img.shields.io/badge/backend-Django%20%2B%20DRF-092E20?style=flat-square&logo=django)](https://github.com/znt10/FechaCaixa-back)
[![Frontend](https://img.shields.io/badge/frontend-Next.js%2016-000000?style=flat-square&logo=nextdotjs)](https://github.com/znt10/FechaCaixa-front)
[![PWA](https://img.shields.io/badge/formul%C3%A1rio-PWA-5A0FC8?style=flat-square&logo=pwa)](#o-formulário-da-loja)
[![NF-e](https://img.shields.io/badge/notas-NF--e%20XML-0B7285?style=flat-square)](#notas-fiscais)
[![License](https://img.shields.io/badge/licen%C3%A7a-MIT-blue?style=flat-square)](#licença)

[Backend](https://github.com/znt10/FechaCaixa-back) &middot;
[Frontend](https://github.com/znt10/FechaCaixa-front) &middot;
[Quick start](#quick-start)

</div>

---

## Em resumo

Loja pequena fecha o caixa no caderno: quanto entrou em dinheiro, em cartão e
em Pix, quem retirou dinheiro da gaveta, o que foi gasto, o que os funcionários
consumiram. O caderno vai para a gerência no fim da semana, e aí já não dá para
perguntar a ninguém por que a conta não bateu.

| | |
|---|---|
| 📱 **Formulário sem login** | o celular da loja entra uma vez com o código da empresa e vira o caixa daquele balcão |
| 🕐 **Turno a turno** | manhã e tarde nos dias comuns; domingo e feriado são um turno só, com feriados nacionais calculados |
| 🧾 **Saídas separadas** | retirada, despesa, consumo dos funcionários e desperdício de salgados — cada um no seu lugar |
| 📊 **Painel da gerência** | por loja, por período, com gráficos, planilha para exportar e o que ainda falta lançar |
| 📄 **Notas fiscais** | importa o XML da NF-e, e a mesma nota não entra duas vezes |
| 🏢 **Várias empresas** | cada uma com as suas lojas, funcionários e código de acesso — isoladas na API, não só na tela |

**Stack:** Django + DRF + Celery + MySQL no backend, Next.js 16 + TypeScript no
frontend, JWT em cookie HTTP-only, tudo em Docker.

---

## O formulário da loja

Quem fecha o caixa é quem estava no balcão, no fim do expediente, com pressa.
Por isso o formulário **não tem login**.

A empresa tem um **código de acesso** (algo como `PRIM-4821`). O aparelho digita
esse código uma vez e recebe um cookie HTTP-only que vale 180 dias. Daí em
diante, abrir `/<empresa>` já mostra o formulário das lojas daquela empresa. A
empresa sai sempre desse cookie — nunca de um parâmetro na URL.

Três decisões que acompanham:

- **O código aceita do jeito que a pessoa digitar.** `prim 4821`, `PRIM4821` e
  `PRIM-4821` são a mesma pessoa acertando o código. O teclado do celular
  capitaliza sozinho, e ninguém digita o hífen.
- **Tentativa de código tem teto por IP**, e continua tendo para quem está
  logado. O throttle padrão do DRF para anônimo devolve `None` para requisição
  autenticada, o que deixaria qualquer conta tentar códigos sem limite.
- **Instala como app (PWA).** E avisa quando sai versão nova, sem recarregar a
  tela por cima de um lançamento pela metade.

Antes de enviar, o formulário mostra uma **revisão** com o total do turno.
É o último momento em que quem lançou ainda está com o dinheiro na mão.

---

## O painel da gerência

| Tela | O que responde |
|---|---|
| **Fechamentos** | o que foi lançado; conferir, corrigir, cancelar ou lançar um turno retroativo |
| **Por loja** | quanto cada loja fez no período e quais turnos ainda não chegaram |
| **Saídas** | retiradas (e quem retirou), despesas, consumo por funcionário, desperdício por salgado |
| **Gráficos** | a evolução do período |
| **Notas fiscais** | as NF-e importadas, por loja e por plano de contas |
| **Catálogo** | os salgados da empresa, para o desperdício ser contado por item |
| **Empresa** | código de acesso, lojas (com CNPJ), funcionários, quem pode retirar dinheiro, aparelhos conectados |

Três perfis, e a fronteira mora na API:

| Grupo | O que alcança |
|---|---|
| `Admin` | tudo, mais o Django Admin |
| `Gerente` | a própria empresa — e o painel |
| `Funcionario` | o painel: vê, confere e corrige. Não administra a empresa |

O isolamento por empresa está no `get_queryset` de cada ViewSet: quem chamar a
API direto continua vendo só a própria conta. E cada rota que o funcionário
**não** pode alcançar tem um teste próprio — afrouxar uma delas quebra a suíte
em vez de virar notícia.

---

## Notas fiscais

Um módulo ligado por empresa. A gerente sobe o XML da NF-e, e o backend:

- lê o XML com proteção contra expansão de entidades (há um teste com uma
  "bomba" de entidades);
- recusa nota de saída, nota cujo destinatário não é uma loja da empresa, e
  nota que já foi lançada;
- classifica a despesa num plano de contas (grupo → elemento → fornecedor).

A regra central: **nenhuma despesa entra sem prova, e a mesma prova não entra
duas vezes.** Ela vive num serviço que não conhece HTTP, para que a leitura
automática de uma caixa de e-mail, no futuro, obedeça exatamente à mesma regra.

---

## Arquitetura

```text
     celular da loja                         gerência
    exemplo.com/primavera                exemplo.com/fechamentos
     (cookie do aparelho)                  (login, JWT em cookie)
             │                                     │
             └──────────────────┬──────────────────┘
                                ▼
              ┌───────────────────────────────────┐
              │   Frontend — Next.js 16             │
              │   proxy.ts: rota pública, painel    │
              │   por papel, endereço de empresa    │
              │   /backend/* → rewrite para a API   │
              └─────────────────┬─────────────────┘
                                │  mesmo domínio: os cookies HTTP-only
                                │  do Django viram first-party
                                ▼
              ┌───────────────────────────────────┐
              │   Backend — Django + DRF            │
              │   empresa sempre do cookie ou do    │
              │   usuário, nunca da URL             │
              └───────┬──────────────────┬────────┘
                      ▼                  ▼
               ┌────────────┐      ┌────────────┐
               │   MySQL     │      │   Redis     │
               └────────────┘      └──────┬─────┘
                                          ▼
                                   ┌────────────┐
                                   │   Celery    │
                                   │ worker+beat │
                                   └────────────┘
```

O front **nunca chama a API direto**. Toda chamada vai para `/backend/...` no
próprio domínio, e o servidor do Next reescreve para o Django. É isso que faz o
navegador tratar os cookies do backend como first-party — e é por isso que o
front não tem nenhuma variável `NEXT_PUBLIC_` apontando para a API.

Toda resposta sob `/api/` sai com `Cache-Control: no-store`. Sem isso, a lista
de lojas ficava congelada no celular da loja mesmo depois de recarregar: o
pedido nem saía do aparelho.

---

## Como está organizado

Este repositório é o guarda-chuva. O código vive em dois repositórios
independentes, ligados aqui como submódulos:

```text
FechaCaixa/
├── backend/    → znt10/FechaCaixa-back    Django + DRF + Celery + MySQL
└── frontend/   → znt10/FechaCaixa-front   Next.js 16 + React 19 + TypeScript
```

Cada um tem o próprio README, com variáveis de ambiente, comandos e as decisões
internas daquele lado.

O backend sobe e testa **sozinho** — o front depende dele, não o contrário.

---

## Quick start

### Pré-requisitos

- **Docker** e **Docker Compose**
- **Git**

### Clonar

Com os submódulos — sem a flag, `backend/` e `frontend/` vêm vazias:

```bash
git clone --recurse-submodules https://github.com/znt10/FechaCaixa.git
cd FechaCaixa
```

Se já clonou sem a flag:

```bash
git submodule update --init --recursive
```

### Backend primeiro

```bash
cd backend
cp .env.example .env      # o arquivo documenta cada variável
docker compose up -d --build
```

Sobe MySQL, Redis, a API em `localhost:8000`, o worker e o beat. Ao subir, o
entrypoint aplica as migrations, cria os grupos e o usuário admin a partir do
`.env`.

| Onde | URL |
|---|---|
| API | http://localhost:8000 |
| Swagger | http://localhost:8000/api/schema/swagger/ |
| Admin Django | http://localhost:8000/admin/ |

### Frontend

O front entra na rede do compose do backend, então ele precisa estar de pé:

```bash
cd ../frontend/frontend
cp .env.example .env.local
docker compose up
```

- `http://localhost:3000/login` — o painel
- `http://localhost:3000/fechamento` — a entrada do formulário, pelo código da empresa

Crie a empresa, as lojas e o código de acesso pelo Django Admin ou pela tela
**Empresa** do painel.

### Testar

```bash
cd backend           && docker compose exec api python manage.py test app --noinput
cd frontend/frontend && npm test
```

---

## Tecnologias

**Backend** — Python 3.12, Django 6, Django REST Framework, SimpleJWT (cookie
HTTP-only), drf-spectacular, Celery + Redis, MySQL, openpyxl (planilha), Docker Compose

**Frontend** — Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS 4,
HeroUI, TanStack Query, Zustand, Vitest

---

## Histórico

O FechaCaixa nasceu do [Unistock](https://github.com/znt10/Unistock), um
sistema de estoque e pedidos. Estoque, PDV, pedidos e notificações foram
removidos quando o foco virou o caixa — alguns comentários no código ainda
citam o Unistock, sempre para explicar por que algo é do jeito que é.

---

## Licença

MIT — veja o arquivo `LICENSE` em cada repositório
([backend](https://github.com/znt10/FechaCaixa-back/blob/main/LICENSE),
[frontend](https://github.com/znt10/FechaCaixa-front/blob/main/LICENSE)).

---

<p align="center">
  Feito por <a href="https://github.com/znt10">znt10</a>
</p>
