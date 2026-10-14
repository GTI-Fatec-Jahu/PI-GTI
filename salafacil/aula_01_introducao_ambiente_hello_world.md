# Aula 01 — Introdução ao Flask, Ambiente Virtual e Hello World com Tailwind CSS

## 1. O que é o Flask e Como Ele Funciona

O **Flask** é um microframework web escrito em Python, criado por Armin Ronacher. O termo "micro" não significa que o Flask seja limitado ou voltado apenas para sistemas pequenos, mas sim que seu núcleo é enxuto, modular e não impõe decisões rígidas sobre qual banco de dados, biblioteca de formulários ou motor de autenticação você deve utilizar.

Ao desenvolver nosso **Sistema de Gestão de Reservas de Salas com Permissões Personalizadas**, essa flexibilidade permite conectar o Python diretamente ao **MySQL**, estruturar consultas SQL otimizadas para paginação e busca dinâmica, e estilizar toda a interface com **Tailwind CSS**.

### Os Pilares da Arquitetura do Flask

| Componente | Papel na Aplicação | Como Usaremos no Projeto |
| :--- | :--- | :--- |
| **WSGI (Werkzeug)** | Camada que gerencia requisições HTTP (`Request`), respostas (`Response`), roteamento de URLs e sessões seguras. | Capturar parâmetros de busca (`?q=`), página atual (`?page=`), dados de formulários (`POST`) e cookies de sessão criptografados. |
| **Jinja2** | Motor de templates que combina HTML com expressões Python (`{{ variavel }}` e `{% bloco %}`). | Renderizar layouts com Tailwind CSS, tabelas paginadas, componentes de busca e ocultar botões conforme as permissões do usuário. |
| **Application Factory & Blueprints** | Padrão arquitetural que divide o sistema em módulos independentes (autenticação, salas, reservas, usuários, relatórios). | Manter o código organizado e escalável conforme adicionamos novas telas e regras de negócio. |

---

## 2. Pré-requisitos do Ambiente

Antes de iniciar, certifique-se de ter instalado em sua máquina:

1. **Python 3.10 ou superior**: Verifique no terminal com `python --version` ou `python3 --version`.
2. **Servidor MySQL 8.0+**: Pode ser instalado localmente (MySQL Community Server, XAMPP, Laragon ou Docker) com acesso a um cliente como MySQL Workbench, DBeaver ou terminal `mysql`.
3. **Editor de Código**: Visual Studio Code (recomendado com as extensões *Python*, *Jinja* e *Tailwind CSS IntelliSense*).

---

## 3. Criação da Pasta do Projeto e Ambiente Virtual (`venv`)

Um **ambiente virtual (`venv`)** isola as bibliotecas Python do nosso projeto em uma pasta própria, evitando conflitos de versão entre diferentes projetos instalados no mesmo computador.

### Passo 3.1 — Criar a pasta do projeto

Abra o terminal (PowerShell/CMD no Windows, ou Bash/Zsh no Linux/macOS) e execute:

```bash
# 1. Cria a pasta principal do sistema
mkdir sistema_reservas_salas

# 2. Entra na pasta criada
cd sistema_reservas_salas
```

### Passo 3.2 — Criar o ambiente virtual

Execute o comando correspondente ao seu sistema operacional:

```bash
# No Windows:
python -m venv venv

# No Linux ou macOS:
python3 -m venv venv
```

Após executar o comando, uma pasta chamada `venv/` será criada dentro de `sistema_reservas_salas/`. Ela contém uma cópia isolada do interpretador Python e do gerenciador de pacotes `pip`.

### Passo 3.3 — Ativar o ambiente virtual

Sempre que for trabalhar no projeto ou instalar pacotes, o ambiente virtual deve estar ativo:

```bash
# No Windows (CMD):
venv\Scripts\activate.bat

# No Windows (PowerShell):
venv\Scripts\Activate.ps1

# No Linux ou macOS:
source venv/bin/activate
```

> **Dica para Windows PowerShell:** Caso receba um erro de política de execução de scripts ao rodar `Activate.ps1`, execute `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` no PowerShell e tente novamente.

Quando o ambiente estiver ativo, você verá o prefixo `(venv)` no início da linha do terminal.

---

## 4. Instalação do Flask e Dependências do Projeto

Com o `(venv)` ativo, atualize o `pip` e instale os pacotes que utilizaremos ao longo de todas as aulas:

```bash
# Atualiza o gerenciador de pacotes pip
pip install --upgrade pip

# Instala o Flask, o conector oficial do MySQL e o leitor de variáveis de ambiente
pip install Flask==3.0.3 mysql-connector-python==9.0.0 python-dotenv==1.0.1
```

### Por que escolhemos cada pacote?

| Pacote | Finalidade no Sistema |
| :--- | :--- |
| `Flask` | Framework web principal (inclui `Werkzeug` para hash de senhas e `Jinja2` para templates). |
| `mysql-connector-python` | Driver oficial da Oracle/MySQL para conectar o Python ao banco de dados MySQL com suporte a pool de conexões e cursores em formato de dicionário. |
| `python-dotenv` | Carrega credenciais sensíveis (senha do banco, `SECRET_KEY`) a partir de um arquivo `.env`, evitando expor senhas no código-fonte. |

### Congelando as dependências (`requirements.txt`)

Para que qualquer desenvolvedor consiga recriar o mesmo ambiente com `pip install -r requirements.txt`, salve a lista de pacotes instalados:

```bash
pip freeze > requirements.txt
```

---

## 5. Primeiro Aplicativo: O Clássico "Hello World"

Vamos criar nosso primeiro arquivo Python para compreender o ciclo de vida mínimo de uma aplicação Flask. Crie o arquivo `hello.py` na raiz da pasta `sistema_reservas_salas/`:

```python
# hello.py
from flask import Flask

# Instancia a aplicação Flask.
# __name__ informa ao Flask onde localizar arquivos estáticos e templates.
app = Flask(__name__)


# O decorator @app.route associa a URL "/" à função Python logo abaixo.
@app.route("/")
def ola_mundo():
    return "<h1>Olá, Mundo! Nosso servidor Flask está no ar!</h1>"


@app.route("/sobre")
def sobre():
    return {
        "sistema": "Gestão de Reservas de Salas",
        "tecnologias": ["Python", "Flask", "MySQL", "Tailwind CSS"],
        "status": "online",
    }


if __name__ == "__main__":
    # debug=True ativa o recarregamento automático ao salvar arquivos
    # e exibe o depurador interativo no navegador em caso de erros.
    app.run(host="127.0.0.1", port=5000, debug=True)
```

### Executando o Hello World

No terminal (com o `venv` ativo), execute:

```bash
python hello.py
```

O terminal exibirá uma saída semelhante a esta:

```text
 * Serving Flask app 'hello'
 * Debug mode: on
 * Running on http://127.0.0.1:5000
Press CTRL+C to quit
 * Restarting with stat
 * Debugger is active!
```

Abra o navegador e acesse:
- `http://127.0.0.1:5000/` para ver a mensagem em HTML.
- `http://127.0.0.1:5000/sobre` para observar que, ao retornar um dicionário Python, o Flask o converte automaticamente para uma resposta **JSON** (`application/json`) — recurso essencial que usaremos nas próximas aulas para a pesquisa dinâmica nas tabelas!

---

## 6. Estruturando o Projeto Profissional com Tailwind CSS

Agora que entendemos o funcionamento básico, vamos organizar a estrutura definitiva de diretórios do nosso **Sistema de Gestão de Reservas de Salas** para suportar múltiplos módulos, banco de dados MySQL e templates com Tailwind CSS.

### Estrutura de Pastas do Projeto

Crie a seguinte árvore de pastas e arquivos dentro de `sistema_reservas_salas/`:

```text
sistema_reservas_salas/
├── venv/
├── .env
├── requirements.txt
├── run.py
├── sql/
│   ├── schema.sql
│   └── seed.sql
└── app/
    ├── __init__.py
    ├── config.py
    ├── database.py
    ├── utils/
    │   ├── __init__.py
    │   ├── pagination.py
    │   └── security.py
    ├── blueprints/
    │   ├── __init__.py
    │   ├── auth.py
    │   ├── usuarios.py
    │   ├── salas.py
    │   ├── reservas.py
    │   └── relatorios.py
    ├── static/
    │   └── js/
    │       └── tabela_dinamica.js
    └── templates/
        ├── base.html
        ├── components/
        │   └── paginacao.html
        ├── auth/
        ├── usuarios/
        ├── salas/
        ├── reservas/
        └── relatorios/
```

Você pode criar todas as pastas de uma só vez pelo terminal:

```bash
# Linux / macOS / Git Bash / PowerShell:
mkdir -p sql app/utils app/blueprints app/static/js app/templates/components app/templates/auth app/templates/usuarios app/templates/salas app/templates/reservas app/templates/relatorios
```

### Passo 6.1 — Criando o arquivo `.env`

Na raiz do projeto (`sistema_reservas_salas/.env`), crie o arquivo com as variáveis de configuração:

```ini
FLASK_ENV=development
SECRET_KEY=chave_super_secreta_desenvolvimento_reservas_2026
DB_HOST=127.0.0.1
DB_PORT=3306
DB_USER=root
DB_PASSWORD=sua_senha_mysql
DB_NAME=gestao_reservas_salas
```

### Passo 6.2 — Criando o arquivo de Configuração (`app/config.py`)

```python
# app/config.py
import os
from dotenv import load_dotenv

load_dotenv()


class Config:
    SECRET_KEY = os.getenv("SECRET_KEY", "chave-padrao-insegura")
    DB_HOST = os.getenv("DB_HOST", "127.0.0.1")
    DB_PORT = int(os.getenv("DB_PORT", 3306))
    DB_USER = os.getenv("DB_USER", "root")
    DB_PASSWORD = os.getenv("DB_PASSWORD", "")
    DB_NAME = os.getenv("DB_NAME", "gestao_reservas_salas")
    ITEMS_PER_PAGE = 8
```

### Passo 6.3 — Criando a Application Factory (`app/__init__.py`)

```python
# app/__init__.py
from flask import Flask, render_template
from app.config import Config


def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)

    # Rota inicial provisória de boas-vindas com Tailwind CSS
    @app.route("/")
    def index():
        resumo_inicial = [
            {"modulo": "Usuários & Permissões", "status": "Pronto para Aula 02/03", "icone": "🔐"},
            {"modulo": "Catálogo de Salas", "status": "Pronto para Aula 04", "icone": "🏢"},
            {"modulo": "Agenda de Reservas", "status": "Pronto para Aula 05", "icone": "📅"},
            {"modulo": "Relatórios Gerenciais", "status": "Pronto para Aula 06", "icone": "📊"},
        ]
        return render_template("boas_vindas.html", modulos=resumo_inicial)

    return app
```

### Passo 6.4 — Primeiro Template com Tailwind CSS (`app/templates/boas_vindas.html`)

Utilizaremos o **Tailwind CSS Play CDN** configurado com paleta customizada para termos todas as classes utilitárias disponíveis imediatamente, sem exigir Node.js em ambiente de aula:

```html
<!-- app/templates/boas_vindas.html -->
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>SalaFácil — Gestão de Reservas de Salas</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-slate-50 text-slate-800 min-h-screen flex flex-col justify-between">
    <header class="bg-indigo-700 text-white shadow-md">
        <div class="max-w-6xl mx-auto px-6 py-4 flex items-center justify-between">
            <div class="flex items-center gap-3">
                <span class="bg-white text-indigo-700 font-bold px-3 py-1 rounded-lg text-lg">SF</span>
                <h1 class="text-xl font-bold tracking-tight">SalaFácil — Flask + MySQL + Tailwind CSS</h1>
            </div>
            <span class="text-xs bg-indigo-900 px-3 py-1 rounded-full font-medium">Aula 01 Concluída</span>
        </div>
    </header>

    <main class="max-w-6xl mx-auto px-6 py-10 flex-1 w-full">
        <div class="bg-white rounded-2xl shadow-sm border border-slate-200 p-8 mb-8">
            <h2 class="text-2xl font-bold text-slate-900 mb-2">Ambiente Flask Configurado com Sucesso!</h2>
            <p class="text-slate-600">
                Sua aplicação já utiliza o padrão <strong>Application Factory</strong>, variáveis de ambiente com <code>.env</code> e estilização responsiva com <strong>Tailwind CSS</strong>.
            </p>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
            {% for item in modulos %}
            <div class="bg-white p-6 rounded-xl border border-slate-200 shadow-sm hover:shadow-md transition">
                <div class="text-3xl mb-3">{{ item.icone }}</div>
                <h3 class="font-semibold text-lg text-slate-900">{{ item.modulo }}</h3>
                <p class="text-sm text-indigo-600 mt-1 font-medium">{{ item.status }}</p>
            </div>
            {% endfor %}
        </div>
    </main>

    <footer class="bg-white border-t border-slate-200 py-4 text-center text-sm text-slate-500">
        Curso Prático de Flask com MySQL — Sistema de Gestão de Reservas de Salas
    </footer>
</body>
</html>
```

### Passo 6.5 — Ponto de Entrada (`run.py`)

Na raiz do projeto (`sistema_reservas_salas/run.py`), crie o arquivo que inicia o servidor:

```python
# run.py
from app import create_app

app = create_app()

if __name__ == "__main__":
    app.run(host="127.0.0.1", port=5000, debug=True)
```

Execute `python run.py` e acesse `http://127.0.0.1:5000`. Você verá os cards renderizados pelo Jinja2 e estilizados com Tailwind CSS.

---

## 7. Exercícios de Fixação da Aula 01

1. Adicione no arquivo `.env` uma variável chamada `NOME_INSTITUICAO="Centro Universitário Exemplo"` e exponha-a na classe `Config` em `app/config.py`.
2. Crie uma nova rota `/status` em `app/__init__.py` que retorne um JSON contendo o nome da instituição lido de `app.config["NOME_INSTITUICAO"]`.
3. Na próxima aula (**Aula 02**), modelaremos o banco de dados **MySQL** completo com controle de **permissões personalizadas por usuário** e conectaremos o Flask ao MySQL.
