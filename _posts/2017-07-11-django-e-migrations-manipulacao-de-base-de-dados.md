---
layout: post
title:  "Django e Migrations: Manipulação de Base de Dados"
date:   2017-07-11 23:14:11 +0000
categories: django
image:  /preview.png
---

Criar, alterar ou remover as estruturas ou dados de um banco de dados manualmente sempre foi uma atividade de alto risco e demorada. Mesmo com a realização de backups, uma alteração incorreta em uma base de dados (de produção!) pode trazer muita dor de cabeça para a equipe de desenvolvimento e infraestrutura.

Percebendo essa dificuldade, a partir do Django 1.7 foi inserida a migração de forma nativa. Antes disso existiam soluções de terceiros, como o [South](http://south.readthedocs.io/en/latest/), base da atual implementação nativa.

## South

South é um projeto que iniciou em 2007 e que em breve deverá ser descontinuado. Como dito na [própria documentação do projeto](http://south.readthedocs.io/en/latest/releasenotes/1.0.html), o último grande lançamento foi a versão 1.0. A partir daí somente versões com atualizações de segurança foram/serão lançadas.

Atualmente o autor da biblioteca concentra suas atenções na migração nativa do Django, que utiliza como base o South.

## Migração Nativa (Migrations)

No Django, existem vários comandos que podem ser utilizados para interagir com a base de dados. Abaixo citamos os principais utilizados:

* `migrate`: possibilita aplicar ou cancelar migrations. Também possibilita listar o status das operações.
* `makemigrations`: responsável pela criação de novos migrations baseados nas alterações dos modelos.
* `sqlmigrate`: exibi as SQLs de um migrations.
* `showmigrations`: apresenta todos os migrations associados ao projeto.

Em resumo, podemos dizer que o `makemigrations` é o responsável por criar versões das alterações realizadas nos modelos das aplicações, enquanto que o `migrate` é o responsável pela aplicação de tais mudanças na base de dados.

### Parâmetros Especiais

Alguns dos comandos acima aceitam parâmetros que podem modificar a ação a ser executada. Abaixo citaremos alguns deles.

#### makemigrations

Alguns dos parâmetros aceitos no `makemigrations` são:

* `--name <nome_da_alteracao>`: através desse parâmetro é possível nomear a alteração realizada.
* `--empty <nome_do_app>`: indicar que um arquivo de migração vazio deve ser criado (isso geralmente é (utilizado para realizar alterações nos dados – *data migrations*. Poderemos ver isso em outro post!). Com esse parâmetro, o Django irá colocar o arquivo de migração vazio no lugar certo, sugerir um nome e adicionar dependências.
* `--merge`: esse parâmetro é utilizado para resolver conflitos simples de *migrations*. Em casos de conflitos mais complexos o Django provavelmente não realizará o merge e outro método deverá ser utilizado.

#### migrate

Já no `migrate`, alguns dos parâmetros aceitos são:

* `--fake`: nesse caso o Django somente irá marcar o *migrations* como executado, entretanto não realizará alterações na base dados.
* `--fake-initial`: em resumo, esse parâmetro irá fazer com que o Django verifique se as tabelas existem e, em caso afirmativo, irá falso-aplicar migrações iniciais. Ou seja, o Django irá marcar os *migrations* de criação da tabela como já executado. Do mesmo modo com as colunas das tabelas.
* `--list`: irá mostrar uma lista de todas as *migrations* conhecidas e quais foram aplicadas.

Bom pessoal, espero ter ajudado. Em outras postagens pretendo abordar outros assuntos relacionados ao `Migrations` (*data migrations*, *rollback*, outros métodos de merge, etc).

Abraços e até a próxima!

## Links Interessantes

* [Documentação Oficial (Django 1.9)](https://docs.djangoproject.com/pt-br/1.9/topics/migrations/)
* [Documentação do South](http://south.readthedocs.io/en/latest/)
