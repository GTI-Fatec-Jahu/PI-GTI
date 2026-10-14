# Curso Prático de Flask com MySQL e Tailwind CSS — Sistema SalaFácil

Bem-vindo(a) ao curso prático de desenvolvimento web com **Python (Flask)**, **MySQL** e **Tailwind CSS**. Ao longo de **6 aulas progressivas**, você construirá do zero o **SalaFácil**, um sistema completo de **Gestão de Reservas de Salas** com controle de acesso baseado em **permissões personalizadas por usuário**, **paginação server-side**, **pesquisa dinâmica em todas as tabelas** e **relatórios gerenciais exportáveis**.

---

## 1. Apresentação do Curso e Objetivos de Aprendizagem

Este material foi estruturado para guiar estudantes e desenvolvedores desde os primeiros passos no ecossistema Flask até a entrega de uma aplicação corporativa modular. Ao concluir a série de aulas, você será capaz de:

- Compreender o funcionamento interno do **Flask** (arquitetura WSGI, Werkzeug, Jinja2, Application Factory e Blueprints).
- Configurar ambientes isolados em Python (`venv`), variáveis de ambiente seguras (`.env`) e dependências (`requirements.txt`).
- Modelar bancos de dados relacionais no **MySQL** com chaves estrangeiras, integridade referencial e índices compostos otimizados para buscas textuais e intervalos de horário.
- Implementar **Permissões Personalizadas (Granulares)** por usuário no banco de dados, protegendo rotas no backend via *decorators* e adaptando a interface dinamicamente no Jinja2.
- Construir um motor universal de **Paginação Server-Side** (`LIMIT`/`OFFSET`) aliado à **Pesquisa Dinâmica Assíncrona** (`fetch` + *debounce*) aplicado a 100% das tabelas do sistema.
- Validar regras críticas de negócio no MySQL, como a **detecção matemática de conflito de horários** em agendamentos de salas.
- Desenvolver um painel de **Relatórios Gerenciais** estilizado com **Tailwind CSS**, contendo indicadores (KPIs), agregações SQL (`TIMESTAMPDIFF`), layout próprio para impressão e exportação em **CSV (UTF-8 com BOM para Excel)**.

---

## 2. Tecnologias Utilizadas

| Camada | Tecnologia / Biblioteca | Versão Recomendada | Papel no Projeto |
| :--- | :--- | :--- | :--- |
| **Linguagem Base** | Python | 3.10+ | Lógica de negócio, rotas, decorators de permissão e paginação. |
| **Framework Web** | Flask (`Werkzeug` + `Jinja2`) | 3.0.3 | Roteamento modular com Blueprints, sessões seguras e renderização de templates. |
| **Banco de Dados** | MySQL + `mysql-connector-python` | 8.0+ / 9.0.0 | Persistência relacional, consultas parametrizadas anti-SQL Injection e agregações. |
| **Estilização UI** | Tailwind CSS | 3.x (Play CDN) | Layout responsivo, sidebar administrativa, badges de status, formulários e cards de KPIs. |
| **Interatividade** | JavaScript Vanilla (`Fetch API` + `DOMParser`) | ES6+ | Pesquisa dinâmica com *debounce* de 300ms e troca de páginas sem recarregar a tela. |
| **Configuração** | `python-dotenv` | 1.0.1 | Leitura segura de variáveis de ambiente a partir do arquivo `.env`. |

---

## 3. Sumário Geral dos Arquivos das Aulas

Abaixo está o guia completo dos **6 arquivos Markdown (`.md`)** que compõem o curso, detalhando os tópicos e arquivos de código construídos em cada etapa:

| Aula | Arquivo Markdown | Resumo do Conteúdo | Principais Arquivos Criados no Projeto |
| :--- | :--- | :--- | :--- |
| **Aula 01** | [aula\_01\_introducao\_ambiente\_hello\_world.md](https://www.google.com/search?q=.%2Faula_01_introducao_ambiente_hello_world.md) | Fundamentos do Flask, criação da pasta, `venv`, instalação e primeiro *Hello World* com Tailwind CSS. | `hello.py`, `.env`, `requirements.txt`, `run.py`, `app/config.py`, `app/__init__.py`, `boas_vindas.html` |
| **Aula 02** | [aula\_02\_banco\_dados\_mysql\_permissoes.md](https://www.google.com/search?q=.%2Faula_02_banco_dados_mysql_permissoes.md) | Modelagem do banco MySQL para reservas de salas e arquitetura de permissões personalizadas por usuário. | `sql/schema.sql`, `app/database.py`, `seed_db.py` |
| **Aula 03** | [aula\_03\_autenticacao\_permissoes\_tailwind.md](https://www.google.com/search?q=.%2Faula_03_autenticacao_permissoes_tailwind.md) | Login com hash de senha, decorators `@permissao_obrigatoria`, helper `tem_permissao` e layout base em Tailwind CSS. | `app/utils/security.py`, `app/blueprints/auth.py`, `base.html`, `auth/login.html`, `auth/403.html` |
| **Aula 04** | [aula\_04\_usuarios\_salas\_paginacao\_busca\_dinamica.md](https://www.google.com/search?q=.%2Faula_04_usuarios_salas_paginacao_busca_dinamica.md) | Motor reutilizável de paginação e busca dinâmica + CRUD de Usuários (matriz de permissões) e Catálogo de Salas. | `app/utils/pagination.py`, `components/paginacao.html`, `static/js/tabela_dinamica.js`, `blueprints/usuarios.py`, `blueprints/salas.py` |
| **Aula 05** | [aula\_05\_gestao\_reservas\_validacao\_conflitos.md](https://www.google.com/search?q=.%2Faula_05_gestao_reservas_validacao_conflitos.md) | Agendamento de salas, validação SQL anti-conflito de horários, aprovação/cancelamento e agenda paginada. | `app/blueprints/reservas.py`, `templates/reservas/index.html`, `templates/reservas/form.html` |
| **Aula 06** | [aula\_06\_relatorios\_gerenciais\_exportacao.md](https://www.google.com/search?q=.%2Faula_06_relatorios_gerenciais_exportacao.md) | Indicadores gerenciais (KPIs), relatórios consolidado e analítico com paginação e busca dinâmica, impressão e CSV. | `app/blueprints/relatorios.py`, `templates/relatorios/painel.html` |

---

## 4. Detalhamento Programático Aula por Aula

### Aula 01 — Introdução ao Flask, Ambiente Virtual e Hello World com Tailwind CSS
**Arquivo:** `aula_01_introducao_ambiente_hello_world.md`
1. **O que é o Flask e Como Ele Funciona**: Pilares WSGI (Werkzeug), Jinja2 e padrão Application Factory com Blueprints.
2. **Pré-requisitos do Ambiente**: Python 3.10+, MySQL 8.0+ e editor de código.
3. **Criação da Pasta do Projeto e Ambiente Virtual (`venv`)**: Comandos passo a passo para Windows (CMD e PowerShell), Linux e macOS.
4. **Instalação do Flask e Dependências**: Instalação de `Flask`, `mysql-connector-python` e `python-dotenv`, além da geração do `requirements.txt`.
5. **Primeiro Aplicativo ("Hello World")**: Criação e execução de `hello.py` retornando HTML e JSON.
6. **Estruturando o Projeto Profissional com Tailwind CSS**: Árvore completa de diretórios, `.env`, `app/config.py`, `app/__init__.py` e primeiro template estilizado.
7. **Exercícios de Fixação da Aula 01**.

### Aula 02 — Banco de Dados MySQL e Modelagem de Permissões Personalizadas
**Arquivo:** `aula_02_banco_dados_mysql_permissoes.md`
1. **Arquitetura de Dados do Sistema de Reservas**: Diferença entre cargos estáticos e permissões granulares personalizadas por usuário.
2. **Visão Geral das Tabelas (`gestao_reservas_salas`)**: Relacionamentos entre `usuarios`, `permissoes`, `usuario_permissoes`, `salas` e `reservas`.
3. **Script SQL Completo (`sql/schema.sql`)**: Criação das 5 tabelas com índices compostos para busca dinâmica e detecção de conflitos.
4. **Catálogo de Permissões Personalizadas**: Detalhamento das 13 chaves de permissão divididas nos módulos *Salas*, *Reservas*, *Usuários* e *Relatórios*.
5. **Conexão Segura entre Flask e MySQL (`app/database.py`)**: Gerenciamento de conexão no objeto `g` do Flask com `teardown_appcontext` e queries parametrizadas.
6. **Script de Inicialização e Carga de Dados (`seed_db.py`)**: População automática de permissões, 3 perfis de usuários de teste e 10 salas físicas.
7. **Consulta SQL das Permissões de um Usuário e Exercícios de Fixação**.

### Aula 03 — Autenticação, Controle de Permissões Personalizadas e Layout Tailwind CSS
**Arquivo:** `aula_03_autenticacao_permissoes_tailwind.md`
1. **Controle de Acesso em Duas Camadas**: Proteção de rotas no Python (Backend) e ocultação condicional de elementos no Jinja2 (Frontend).
2. **Utilitário de Segurança e Decorators (`app/utils/security.py`)**: Implementação de `carregar_usuario_logado()`, `tem_permissao()`, `@login_obrigatorio` e `@permissao_obrigatoria(*codigos)`.
3. **Blueprint de Autenticação (`app/blueprints/auth.py`)**: Rotas `/auth/login`, `/auth/logout` e `/auth/meu-perfil` com `check_password_hash`.
4. **Layout Principal Responsivo (`app/templates/base.html`)**: Menu lateral dinâmico em Tailwind CSS, notificações *Flash* e estilos de impressão (`@media print`).
5. **Templates de Login e Erro 403 (`Acesso Negado`)**: Telas `auth/login.html` e `auth/403.html`.
6. **Integração no `app/__init__.py` e Exercícios de Fixação**.

### Aula 04 — Motor de Paginação, Pesquisa Dinâmica e Módulos de Usuários e Salas
**Arquivo:** `aula_04_usuarios_salas_paginacao_busca_dinamica.md`
1. **Arquitetura Universal de Paginação e Pesquisa Dinâmica**: Como padronizar todas as listagens do sistema em 3 camadas.
2. **Motor de Paginação no Python (`app/utils/pagination.py`)**: Funções `obter_parametros_listagem()` e `paginar_consulta()` com cálculo automático de `COUNT(*)`, `LIMIT`, `OFFSET` e janela de páginas.
3. **Componente Visual de Paginação (`app/templates/components/paginacao.html`)**: Macro Jinja2 reutilizável com Tailwind CSS.
4. **Script de Pesquisa Dinâmica Assíncrona (`app/static/js/tabela_dinamica.js`)**: Busca em tempo real com *debounce* de 300ms, `fetch()` e atualização parcial do DOM via `DOMParser`.
5. **Módulo de Usuários e Permissões Personalizadas (`app/blueprints/usuarios.py`)**: Listagem paginada de usuários com busca dinâmica e tela de checkboxes agrupados por módulo para atribuir permissões individuais.
6. **Módulo de Catálogo de Salas (`app/blueprints/salas.py`)**: Listagem paginada com filtro instantâneo por código, nome, bloco, equipamentos e capacidade mínima, além de cadastro e edição.

### Aula 05 — Gestão de Reservas de Salas e Validação de Conflitos de Horário
**Arquivo:** `aula_05_gestao_reservas_validacao_conflitos.md`
1. **Prevenção de Choques de Horário**: Demonstração matemática e consulta SQL (`hora_inicio < novo_fim AND hora_fim > novo_inicio`) para impedir agendamentos sobrepostos na mesma sala e data.
2. **Blueprint de Reservas (`app/blueprints/reservas.py`)**: Validação de capacidade máxima da sala, definição de status inicial (`aprovada` ou `pendente`) conforme a permissão `reservas.aprovar` e controle de cancelamento (`reservas.cancelar_qualquer` vs. próprio autor).
3. **Template da Agenda de Reservas (`app/templates/reservas/index.html`)**: Tabela paginada com busca dinâmica simultânea por título do evento, finalidade, código/nome da sala, solicitante ou setor, além de filtros por data e status.
4. **Formulário de Solicitação de Reserva (`app/templates/reservas/form.html`) e Exercícios de Fixação**.

### Aula 06 — Relatórios Gerenciais com Tailwind CSS, Paginação, Busca Dinâmica e Exportação CSV
**Arquivo:** `aula_06_relatorios_gerenciais_exportacao.md`
1. **Estrutura do Módulo de Relatórios**: Arquitetura de múltiplas tabelas paginadas independentes na mesma página.
2. **Consultas SQL Analíticas com `TIMESTAMPDIFF`**: Cálculo preciso de horas ocupadas por sala e indicadores de aprovação/cancelamento.
3. **Blueprint de Relatórios (`app/blueprints/relatorios.py`)**: Rota do painel gerencial (`/relatorios/`) e rota de exportação (`/relatorios/exportar-csv`) protegida pela permissão `relatorios.exportar`.
4. **Template do Painel de Relatórios (`app/templates/relatorios/painel.html`)**:
   - Cards de KPIs em Tailwind CSS (Total de Solicitações, Reservas Aprovadas, Pendentes e Horas Alocadas).
   - **Tabela 1**: Desempenho e Ocupação Consolidada por Sala (com pesquisa dinâmica e paginação próprias).
   - **Tabela 2**: Relatório Analítico de Reservas no Período (com pesquisa dinâmica, filtro de datas/status e paginação próprios).
5. **Checklist Final de Revisão do Curso**.

---

## 5. Mapa de Tabelas com Paginação e Pesquisa Dinâmica

Todas as **5 tabelas de dados** construídas ao longo do curso contam com paginação server-side no MySQL e pesquisa dinâmica instantânea:

| Tabela / Tela | Módulo | Campos Pesquisados Dinamicamente (`LIKE`) | Filtros Adicionais |
| :--- | :--- | :--- | :--- |
| **Lista de Usuários** | `usuarios.listar` | Nome, e-mail, setor e etiqueta de cargo | Status da conta (Ativo/Inativo) e itens por página |
| **Catálogo de Salas** | `salas.listar` | Código da sala, nome, bloco e recursos/equipamentos | Capacidade mínima de lugares e itens por página |
| **Agenda de Reservas** | `reservas.listar` | Título do evento, finalidade, código/nome da sala, solicitante e setor | Data específica, status da reserva e itens por página |
| **Ocupação Consolidada por Sala** | `relatorios.painel` | Código da sala, nome da sala e bloco | Intervalo de datas (`data_inicio` e `data_fim`) |
| **Relatório Analítico de Reservas** | `relatorios.painel` | Evento, código da sala, nome da sala, solicitante e setor | Intervalo de datas, status da reserva e exportação CSV |

---

## 6. Guia Rápido de Execução do Projeto Completo

Depois de acompanhar as aulas e criar os arquivos na pasta `sistema_reservas_salas/`, você pode executar o sistema completo com os 4 comandos abaixo:

```bash
# 1. Ative o ambiente virtual
source venv/bin/activate  # Linux/macOS
# ou: venv\Scripts\activate (Windows)

# 2. Instale as dependências
pip install -r requirements.txt

# 3. Crie o banco de dados, as tabelas, as 13 permissões e os dados iniciais
python seed_db.py

# 4. Inicie o servidor Flask
python run.py
```

Acesse `http://127.0.0.1:5000` no navegador e teste os diferentes perfis de permissão criados pelo `seed_db.py` (senha padrão `123456` para todos):

- **Administrador Geral (`admin@salafacil.com`)**: Possui todas as 13 permissões personalizadas ativas.
- **Gestor de Espaços (`gestor@salafacil.com`)**: Gerencia salas, aprova/cancela reservas e exporta relatórios, mas não gerencia usuários nem permissões.
- **Solicitante (`beatriz@salafacil.com`)**: Visualiza salas e agenda de reservas, cria solicitações e cancela apenas os próprios agendamentos.
