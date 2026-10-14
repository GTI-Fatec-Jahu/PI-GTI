# Aula 03 — Autenticação, Controle de Permissões Personalizadas e Layout Tailwind CSS

## 1. Como Funciona o Controle de Acesso Baseado em Permissões no Flask

Na Aula 02 criamos a estrutura no MySQL onde cada usuário possui um conjunto personalizado de códigos de permissão (como `salas.criar`, `reservas.aprovar`, `usuarios.permissoes`, `relatorios.exportar`).

Para que o sistema seja realmente seguro e intuitivo, precisamos aplicar essas permissões em **duas camadas complementares**:

1. **No Backend (Rotas Flask)**: Mesmo que um usuário tente digitar a URL `/usuarios/novo` ou enviar um `POST` diretamente, um *decorator* Python (`@permissao_obrigatoria('usuarios.gerenciar')`) interceptará a requisição e bloqueará o acesso com código HTTP `403 Forbidden` se ele não tiver a permissão no banco.
2. **No Frontend (Templates Jinja2 + Tailwind CSS)**: Uma função auxiliar `tem_permissao('salas.criar')` ficará disponível em todos os templates HTML para ocultar botões, links de menu e ações que o usuário logado não pode executar.

---

## 2. Utilitário de Segurança e Decorators (`app/utils/security.py`)

Crie o arquivo `app/utils/security.py`. Ele carregará os dados do usuário logado e a lista atualizada de suas permissões personalizadas diretamente do MySQL a cada requisição (`@app.before_request`), garantindo que, se um administrador revogar uma permissão de um usuário, o efeito seja **imediato** (sem precisar esperar o usuário fazer logout!).

```python
# app/utils/security.py
from functools import wraps
from flask import session, redirect, url_for, flash, g, abort, request, jsonify
from app.database import query_one, query_all


def carregar_usuario_logado():
    """
    Executa antes de cada requisição HTTP.
    Busca o usuário da sessão e carrega o conjunto (set) de códigos de permissão.
    """
    usuario_id = session.get("usuario_id")
    g.usuario = None
    g.permissoes = set()

    if usuario_id is not None:
        usuario = query_one(
            """
            SELECT id, nome, email, setor, cargo_etiqueta, ativo
            FROM usuarios
            WHERE id = %s AND ativo = 1
            """,
            (usuario_id,),
        )
        if usuario:
            g.usuario = usuario
            linhas = query_all(
                """
                SELECT p.codigo
                FROM permissoes p
                INNER JOIN usuario_permissoes up ON up.permissao_id = p.id
                WHERE up.usuario_id = %s
                """,
                (usuario_id,),
            )
            g.permissoes = {linha["codigo"] for linha in linhas}
        else:
            # Se o usuário foi desativado enquanto navegava, limpa a sessão
            session.clear()


def tem_permissao(codigo_permissao):
    """Retorna True se o usuário logado possuir a permissão informada."""
    if not getattr(g, "usuario", None):
        return False
    return codigo_permissao in getattr(g, "permissoes", set())


def login_obrigatorio(view_func):
    """Decorator que exige que o usuário esteja autenticado."""
    @wraps(view_func)
    def wrapper(*args, **kwargs):
        if g.usuario is None:
            if request.headers.get("X-Requested-With") == "XMLHttpRequest":
                return jsonify({"erro": "Sessão expirada. Faça login novamente."}), 401
            flash("Por favor, faça login para acessar esta página.", "aviso")
            return redirect(url_for("auth.login", proximo=request.path))
        return view_func(*args, **kwargs)
    return wrapper


def permissao_obrigatoria(*codigos_necessarios):
    """
    Decorator que exige que o usuário logado possua TODAS as permissões
    passadas como argumento. Exemplo:
    @permissao_obrigatoria('reservas.visualizar', 'reservas.aprovar')
    """
    def decorator(view_func):
        @wraps(view_func)
        @login_obrigatorio
        def wrapper(*args, **kwargs):
            faltantes = [c for c in codigos_necessarios if not tem_permissao(c)]
            if faltantes:
                if request.headers.get("X-Requested-With") == "XMLHttpRequest":
                    return jsonify({
                        "erro": "Você não possui permissão para realizar esta ação.",
                        "permissoes_faltantes": faltantes,
                    }), 403
                abort(403)
            return view_func(*args, **kwargs)
        return wrapper
    return decorator
```

---

## 3. Blueprint de Autenticação (`app/blueprints/auth.py`)

Agora vamos criar o Blueprint responsável pelo login, logout e exibição do perfil de permissões do próprio usuário. Crie o arquivo `app/blueprints/auth.py`:

```python
# app/blueprints/auth.py
from flask import Blueprint, render_template, request, redirect, url_for, flash, session, g
from werkzeug.security import check_password_hash
from app.database import query_one, query_all
from app.utils.security import login_obrigatorio

auth_bp = Blueprint("auth", __name__, url_prefix="/auth")


@auth_bp.route("/login", methods=["GET", "POST"])
def login():
    if g.usuario is not None:
        return redirect(url_for("reservas.listar"))

    if request.method == "POST":
        email = request.form.get("email", "").strip().lower()
        senha = request.form.get("senha", "")

        usuario = query_one(
            "SELECT * FROM usuarios WHERE email = %s",
            (email,),
        )

        if usuario is None or not check_password_hash(usuario["senha_hash"], senha):
            flash("E-mail ou senha inválidos. Verifique suas credenciais.", "erro")
        elif not usuario["ativo"]:
            flash("Sua conta está desativada. Contate o administrador.", "erro")
        else:
            session.clear()
            session["usuario_id"] = usuario["id"]
            flash(f"Bem-vindo(a), {usuario['nome']}!", "sucesso")
            proximo = request.args.get("proximo")
            return redirect(proximo or url_for("reservas.listar"))

    return render_template("auth/login.html")


@auth_bp.route("/logout")
def logout():
    session.clear()
    flash("Você saiu do sistema com segurança.", "info")
    return redirect(url_for("auth.login"))


@auth_bp.route("/meu-perfil")
@login_obrigatorio
def meu_perfil():
    minhas_permissoes = query_all(
        """
        SELECT p.codigo, p.nome, p.modulo, p.descricao, up.concedido_em
        FROM permissoes p
        INNER JOIN usuario_permissoes up ON up.permissao_id = p.id
        WHERE up.usuario_id = %s
        ORDER BY p.modulo, p.nome
        """,
        (g.usuario["id"],),
    )
    return render_template("auth/perfil.html", permissoes=minhas_permissoes)
```

---

## 4. Layout Principal Responsivo com Tailwind CSS (`app/templates/base.html`)

Nosso layout base utilizará **Tailwind CSS** para criar uma interface administrativa moderna com:
- Menu lateral (*Sidebar*) que exibe apenas os módulos para os quais o usuário logado possui permissão (`tem_permissao(...)`).
- Cabeçalho superior com identificação do usuário, etiqueta de cargo e botão rápido para inspecionar suas permissões ativas.
- Sistema de notificações (*Flash Messages*) com cores semânticas (verde para sucesso, vermelho para erro, âmbar para aviso).

Crie o arquivo `app/templates/base.html`:

```html
<!-- app/templates/base.html -->
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{% block titulo %}SalaFácil — Gestão de Reservas{% endblock %}</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @media print {
            .no-print { display: none !important; }
            body { background: white !important; }
        }
    </style>
</head>
<body class="bg-slate-100 text-slate-800 min-h-screen flex flex-col md:flex-row">

    {% if g.usuario %}
    <!-- Menu Lateral (Sidebar) -->
    <aside class="w-full md:w-64 bg-slate-900 text-slate-100 flex-shrink-0 flex flex-col justify-between no-print">
        <div>
            <div class="p-6 border-b border-slate-800 flex items-center gap-3">
                <span class="bg-indigo-600 text-white font-extrabold px-3 py-1.5 rounded-lg text-lg">SF</span>
                <div>
                    <h1 class="font-bold text-lg leading-none">SalaFácil</h1>
                    <span class="text-xs text-slate-400">Reservas & Espaços</span>
                </div>
            </div>

            <nav class="p-4 space-y-1">
                {% if tem_permissao('reservas.visualizar') %}
                <a href="{{ url_for('reservas.listar') }}"
                   class="flex items-center gap-3 px-4 py-2.5 rounded-lg text-sm font-medium transition
                          {% if request.blueprint == 'reservas' %}bg-indigo-600 text-white{% else %}text-slate-300 hover:bg-slate-800{% endif %}">
                    <span>📅</span> Agenda de Reservas
                </a>
                {% endif %}

                {% if tem_permissao('salas.visualizar') %}
                <a href="{{ url_for('salas.listar') }}"
                   class="flex items-center gap-3 px-4 py-2.5 rounded-lg text-sm font-medium transition
                          {% if request.blueprint == 'salas' %}bg-indigo-600 text-white{% else %}text-slate-300 hover:bg-slate-800{% endif %}">
                    <span>🏢</span> Catálogo de Salas
                </a>
                {% endif %}

                {% if tem_permissao('usuarios.visualizar') %}
                <a href="{{ url_for('usuarios.listar') }}"
                   class="flex items-center gap-3 px-4 py-2.5 rounded-lg text-sm font-medium transition
                          {% if request.blueprint == 'usuarios' %}bg-indigo-600 text-white{% else %}text-slate-300 hover:bg-slate-800{% endif %}">
                    <span>👥</span> Usuários & Permissões
                </a>
                {% endif %}

                {% if tem_permissao('relatorios.visualizar') %}
                <a href="{{ url_for('relatorios.painel') }}"
                   class="flex items-center gap-3 px-4 py-2.5 rounded-lg text-sm font-medium transition
                          {% if request.blueprint == 'relatorios' %}bg-indigo-600 text-white{% else %}text-slate-300 hover:bg-slate-800{% endif %}">
                    <span>📊</span> Relatórios Gerenciais
                </a>
                {% endif %}
            </nav>
        </div>

        <div class="p-4 border-t border-slate-800 bg-slate-950/50">
            <a href="{{ url_for('auth.meu_perfil') }}" class="block hover:bg-slate-800 p-2 rounded-lg transition">
                <p class="text-sm font-semibold text-white truncate">{{ g.usuario.nome }}</p>
                <p class="text-xs text-indigo-400 truncate">{{ g.usuario.cargo_etiqueta }} • {{ g.usuario.setor }}</p>
                <p class="text-[11px] text-slate-400 mt-1">{{ g.permissoes|length }} permissões ativas</p>
            </a>
            <a href="{{ url_for('auth.logout') }}"
               class="mt-3 w-full block text-center py-2 px-3 rounded-lg bg-rose-600/20 text-rose-300 hover:bg-rose-600 hover:text-white text-xs font-semibold transition">
                Sair do Sistema
            </a>
        </div>
    </aside>
    {% endif %}

    <!-- Área de Conteúdo Principal -->
    <div class="flex-1 flex flex-col min-w-0">
        <!-- Mensagens Flash -->
        {% with mensagens = get_flashed_messages(with_categories=true) %}
        {% if mensagens %}
        <div class="max-w-7xl w-full mx-auto px-6 pt-6 space-y-2 no-print">
            {% for categoria, texto in mensagens %}
            <div class="p-4 rounded-xl border text-sm font-medium flex items-center justify-between shadow-sm
                {% if categoria == 'sucesso' %}bg-emerald-50 border-emerald-200 text-emerald-800
                {% elif categoria == 'erro' %}bg-rose-50 border-rose-200 text-rose-800
                {% elif categoria == 'aviso' %}bg-amber-50 border-amber-200 text-amber-800
                {% else %}bg-indigo-50 border-indigo-200 text-indigo-800{% endif %}">
                <span>{{ texto }}</span>
                <button onclick="this.parentElement.remove()" class="text-xs font-bold opacity-60 hover:opacity-100">✕</button>
            </div>
            {% endfor %}
        </div>
        {% endif %}
        {% endwith %}

        <main class="flex-1 max-w-7xl w-full mx-auto px-6 py-8">
            {% block conteudo %}{% endblock %}
        </main>
    </div>

    <script src="{{ url_for('static', filename='js/tabela_dinamica.js') }}"></script>
    {% block scripts %}{% endblock %}
</body>
</html>
```

---

## 5. Templates de Login e Página de Erro 403 (`Acesso Negado`)

### Passo 5.1 — Tela de Login (`app/templates/auth/login.html`)

```html
<!-- app/templates/auth/login.html -->
{% extends "base.html" %}
{% block titulo %}Login — SalaFácil{% endblock %}

{% block conteudo %}
<div class="min-h-[80vh] flex items-center justify-center">
    <div class="bg-white w-full max-w-md rounded-2xl shadow-lg border border-slate-200 p-8">
        <div class="text-center mb-8">
            <span class="inline-block bg-indigo-600 text-white font-extrabold px-4 py-2 rounded-xl text-xl mb-3">SF</span>
            <h2 class="text-2xl font-bold text-slate-900">Acesso ao SalaFácil</h2>
            <p class="text-sm text-slate-500 mt-1">Entre com suas credenciais para gerenciar reservas de salas</p>
        </div>

        <form method="POST" action="{{ url_for('auth.login') }}" class="space-y-5">
            <div>
                <label class="block text-sm font-semibold text-slate-700 mb-1">E-mail Institucional</label>
                <input type="email" name="email" required placeholder="exemplo@salafacil.com"
                       class="w-full px-4 py-2.5 rounded-xl border border-slate-300 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none text-sm">
            </div>

            <div>
                <label class="block text-sm font-semibold text-slate-700 mb-1">Senha</label>
                <input type="password" name="senha" required placeholder="••••••••"
                       class="w-full px-4 py-2.5 rounded-xl border border-slate-300 focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 outline-none text-sm">
            </div>

            <button type="submit"
                    class="w-full py-3 px-4 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold rounded-xl shadow-sm transition text-sm">
                Entrar no Sistema
            </button>
        </form>

        <div class="mt-6 pt-5 border-t border-slate-100 text-xs text-slate-500 space-y-1">
            <p class="font-semibold text-slate-700">Contas de demonstração (Senha: <code>123456</code>):</p>
            <p>• <strong>Admin:</strong> admin@salafacil.com</p>
            <p>• <strong>Gestor:</strong> gestor@salafacil.com</p>
            <p>• <strong>Solicitante:</strong> beatriz@salafacil.com</p>
        </div>
    </div>
</div>
{% endblock %}
```

### Passo 5.2 — Tela de Erro 403 (`app/templates/auth/403.html`)

```html
<!-- app/templates/auth/403.html -->
{% extends "base.html" %}
{% block titulo %}Acesso Negado (403) — SalaFácil{% endblock %}

{% block conteudo %}
<div class="bg-white rounded-2xl border border-rose-200 p-10 text-center max-w-lg mx-auto mt-12 shadow-sm">
    <div class="w-16 h-16 bg-rose-100 text-rose-600 rounded-full flex items-center justify-center text-3xl mx-auto mb-4">
        🔒
    </div>
    <h2 class="text-2xl font-bold text-slate-900">Permissão Insuficiente</h2>
    <p class="text-slate-600 text-sm mt-2">
        Seu perfil atual (<strong>{{ g.usuario.cargo_etiqueta }}</strong>) não possui a permissão personalizada necessária para executar esta operação.
    </p>
    <div class="mt-6 flex justify-center gap-3">
        <a href="{{ url_for('auth.meu_perfil') }}" class="px-4 py-2 rounded-xl bg-slate-100 hover:bg-slate-200 text-slate-700 text-sm font-medium">
            Ver Minhas Permissões
        </a>
        <a href="{{ url_for('reservas.listar') }}" class="px-4 py-2 rounded-xl bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-medium">
            Voltar para a Agenda
        </a>
    </div>
</div>
{% endblock %}
```

---

## 6. Integrando Segurança e Context Processor no `app/__init__.py`

Atualize o arquivo `app/__init__.py` para conectar o banco de dados, carregar o usuário antes de cada requisição, injetar `tem_permissao` no Jinja2 e registrar os Blueprints:

```python
# app/__init__.py
from flask import Flask, render_template, redirect, url_for
from app.config import Config
from app.database import init_db_app
from app.utils.security import carregar_usuario_logado, tem_permissao


def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)

    # Inicializa o gerenciamento de conexão MySQL
    init_db_app(app)

    # Carrega o usuário logado e suas permissões personalizadas antes de cada request
    app.before_request(carregar_usuario_logado)

    # Torna a função tem_permissao('codigo') acessível em qualquer template HTML
    @app.context_processor
    def injetar_permissoes():
        return {"tem_permissao": tem_permissao}

    # Tratador personalizado para erro 403 (Acesso Negado)
    @app.errorhandler(403)
    def acesso_negado(erro):
        return render_template("auth/403.html"), 403

    # Registro dos Blueprints do sistema
    from app.blueprints.auth import auth_bp
    from app.blueprints.usuarios import usuarios_bp
    from app.blueprints.salas import salas_bp
    from app.blueprints.reservas import reservas_bp
    from app.blueprints.relatorios import relatorios_bp

    app.register_blueprint(auth_bp)
    app.register_blueprint(usuarios_bp)
    app.register_blueprint(salas_bp)
    app.register_blueprint(reservas_bp)
    app.register_blueprint(relatorios_bp)

    @app.route("/")
    def index():
        return redirect(url_for("reservas.listar"))

    return app
```

---

## 7. Exercícios de Fixação da Aula 03

1. Crie o template `app/templates/auth/perfil.html` estendendo `base.html` para listar em uma tabela as permissões retornadas pela rota `/auth/meu-perfil`.
2. Faça login com `beatriz@salafacil.com` e observe que os itens **Usuários & Permissões** e **Relatórios Gerenciais** desaparecem automaticamente do menu lateral.
3. Na **Aula 04**, construiremos o motor universal de **Paginação e Pesquisa Dinâmica** e implementaremos os módulos completos de **Usuários (com matriz de permissões)** e **Salas**.
