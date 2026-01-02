# Recallify

Recallify é uma plataforma de estudos e leitura com **Spaced Repetition (SRS)**, feita para treinar revisão de **PostgreSQL** na prática e, ao mesmo tempo, aplicar conceitos de **algoritmos** de forma simples e útil. A ideia central é você cadastrar conteúdos (livros, artigos), organizar capítulos/seções e criar **cards** de pergunta/resposta. A partir das revisões, o sistema calcula automaticamente **quando você deve revisar cada card** para maximizar retenção e reduzir tempo perdido.

O projeto foi desenhado como **monorepo com microserviços isolados**, cada um com seu próprio Dockerfile e linguagem, mas todos compartilhando o mesmo PostgreSQL com **schemas separados** e permissões por serviço. Assim você tem a experiência real de “um banco compartilhado” com isolamento e ownership claros.

## Objetivo do projeto

O objetivo é construir um sistema simples, evolutivo e bem dividido que te faça praticar:

PostgreSQL de verdade: chaves estrangeiras, constraints, índices, joins, transações, migrations, schemas, roles, performance com `EXPLAIN ANALYZE`, e desenho de queries para fila de revisão.

Algoritmos de forma aplicada: um algoritmo de Spaced Repetition (baseado no SM-2 simplificado), priorização por data (fila por vencimento) e métricas/estatísticas para acompanhar desempenho e dificuldade.

## Arquitetura (microserviços)

A arquitetura é composta por três serviços principais, cada um com responsabilidade única e banco separado por schema.

O **Catalog Service (Java)** gerencia o catálogo de estudo: conteúdos (livros/artigos) e suas seções/capítulos. Ele é o “inventário” do que existe para estudar.

O **SRS Service (Go)** é o coração do projeto: gerencia os cards, registra revisões e executa o algoritmo de agendamento do Spaced Repetition para decidir o próximo dia de revisão.

O **Metrics Service** consolida métricas e relatórios (streak, cards vencidos por dia, taxa de acerto, tempo médio). Ele pode ser implementado como agregador consumindo dados do SRS e do Catalog, ou calculando diretamente a partir de views/queries, dependendo do nível de isolamento que você quer no começo.

Os serviços rodam no mesmo ambiente Docker e se comunicam via HTTP. Todos usam o mesmo Postgres, mas cada serviço escreve apenas no seu próprio schema.

## Banco de dados e isolamento por schema

O PostgreSQL usa schemas para separar ownership e evitar acoplamento acidental.

O schema `catalog` pertence ao Catalog Service e contém as tabelas de conteúdos e seções.

O schema `srs` pertence ao SRS Service e contém as tabelas de cards, revisões e os campos de agendamento.

O schema `metrics` (opcional) pode conter tabelas materializadas, views ou snapshots de dados agregados.

A recomendação é usar roles específicas, por exemplo `catalog_user` com permissão total apenas em `catalog`, e `srs_user` com permissão total apenas em `srs`. Isso simula bem um ambiente de microserviços sem exigir bancos separados.

## Modelo de dados (MVP)

No **Catalog (schema `catalog`)** você tem `items` e `sections`.

`items` representa o conteúdo principal (livro, artigo, curso). `sections` representa capítulos/seções do item com ordenação.

No **SRS (schema `srs`)** você tem `cards` e `reviews`.

`cards` armazena frente/verso do card e os campos necessários para agendamento (próxima revisão, intervalo e fator de facilidade). `reviews` armazena cada tentativa de revisão com nota e tempo, criando um histórico.

Essa separação permite que o Catalog evolua como um sistema de organização, enquanto o SRS evolui como um motor de repetição espaçada.

## Algoritmo de Spaced Repetition (SM-2 simplificado)

O Recallify usa uma variação simples inspirada no SM-2, com nota de revisão `grade` de 0 a 5. A cada revisão o sistema atualiza o card com base no desempenho, ajustando o intervalo e o próximo dia de revisão.

Cada card mantém os campos: `interval_days`, `ease_factor`, `reps`, `lapses` e `next_review_at`.

Quando `grade < 3`, considera-se que o card foi “esquecido” e ele volta para revisões curtas. O sistema zera `reps`, incrementa `lapses`, reduz o `ease_factor` e define `interval_days` para 1 (ou outro valor pequeno), recalculando `next_review_at`.

Quando `grade >= 3`, considera-se que houve retenção suficiente. O sistema incrementa `reps` e calcula um intervalo crescente. Nos dois primeiros acertos, usa-se intervalos fixos (por exemplo 1 e 3 dias). A partir do terceiro acerto, o intervalo cresce multiplicando o intervalo anterior pelo `ease_factor`. O `ease_factor` também é ajustado pela qualidade da resposta, com limite mínimo típico de 1.3.

Um exemplo de regras, em linguagem de produto, fica assim: se você foi bem, o Recallify “empurra” esse card mais para frente; se você foi mal, ele “puxa” para revisar mais cedo, reduzindo o fator de facilidade daquele card.

O efeito prático é uma fila diária de estudos que prioriza o que está vencido ou prestes a vencer, reduzindo revisões desnecessárias e melhorando retenção com menos esforço.

## Fila de revisão (o que revisar agora)

O “Revisar agora” do SRS é uma consulta que busca cards ativos com `next_review_at` menor ou igual ao momento atual e retorna ordenado por vencimento. Isso cria uma fila natural de prioridade por data, que é um padrão simples e muito poderoso para sistemas de agendamento.

Uma query típica é: buscar cards ativos vencidos, ordenar por `next_review_at` e limitar.

O índice mais importante do sistema é em `srs.cards(next_review_at)` para tornar essa fila barata mesmo com muitos cards.

## Endpoints (visão de alto nível)

O **Catalog Service (Java)** expõe endpoints para criar/listar itens e seções. Exemplos: criar item, listar itens, criar seção para um item, listar seções de um item.

O **SRS Service (Go)** expõe endpoints para criar/listar cards, buscar fila de revisão e registrar uma revisão. Exemplos: criar card em uma seção, buscar cards vencidos, registrar revisão com `grade` e `seconds_spent`.

O **Metrics Service** expõe endpoints de dashboard, como cards vencidos por dia, taxa de acerto, tempo médio por revisão e streak.

O design recomendado é o SRS referenciar o Catalog por `section_id` e, quando precisar de detalhes (título do livro/capítulo), buscar no Catalog via HTTP. Em um MVP, você pode manter isso simples e só armazenar `section_id` no card e buscar detalhes sob demanda.

## Como rodar o projeto (Docker)

O jeito mais simples é subir tudo pelo `docker-compose.yaml` na raiz do repositório.

Pré-requisitos: Docker e Docker Compose instalados.

```bash
docker compose up --build
```

Isso sobe o PostgreSQL e o Catalog Service (Java) na porta `8080`. O banco fica acessível em `localhost:5432` com as credenciais abaixo:

- Host: `localhost`
- Porta: `5432`
- Usuário: `recallify_user`
- Senha: `recallify_password`
- Database: `recallify_db`

Para derrubar os serviços:

```bash
docker compose down
```

Para resetar o banco removendo os dados do volume:

```bash
docker compose down -v
```

## Estrutura do repositório

- `docker-compose.yaml`: orquestra o PostgreSQL e os serviços.
- `recallify-catalog-service/`: serviço Java para catálogo (Dockerfile na raiz do serviço).
- `recallify-srs-service/`: serviço Go do SRS (em construção).
- `recallify-metrics-service/`: serviço de métricas/analytics (em construção).

## Próximos passos sugeridos

- Implementar o Catalog Service (rotas CRUD de itens/seções, migrations e seed inicial).
- Implementar o SRS Service em Go com as rotas de cards/reviews e o algoritmo descrito.
- Criar o Metrics Service com consultas agregadas e/ou views materializadas.
- Adicionar migrations organizadas por serviço usando schemas separados e roles dedicadas.
- Cobrir endpoints com testes de integração contra um Postgres de teste (pode usar o compose).
