# Aula 04 — Motor de Paginação, Pesquisa Dinâmica e Módulos de Usuários e Salas

## 1. Arquitetura Universal de Paginação e Pesquisa Dinâmica

Um dos requisitos essenciais do nosso sistema é que **todas as tabelas possuam paginação e pesquisa dinâmica**. Em vez de duplicar cálculos de páginas e código JavaScript em cada tela, construiremos uma arquitetura reutilizável em 3 partes:

1. **No Backend (`app/utils/pagination.py`)**: Uma função `paginar_consulta()` que recebe a consulta SQL base, os parâmetros de filtro, a página atual (`page`) e o limite por página (`per_page`), executando automaticamente o `COUNT(*)` e o `SELECT ... LIMIT %s OFFSET %s` no MySQL.
2. **No Template (`app/templates/components/paginacao.html`)**: Uma macro Jinja2 estilizada com Tailwind CSS que renderiza os botões de página, seletor de itens por página e resumo ("Exibindo 1 a 8 de 42 registros").
3. **No Frontend (`app/static/js/tabela_dinamica.js`)**: Um controlador JavaScript com *Debounce* de 300ms que intercepta a digitação nos campos de busca (`.js-busca-dinamica`) e os cliques na paginação, buscando a tabela atualizada via `fetch()` e substituindo apenas o container `#container-tabela` sem recarregar a página inteira!

---

## 2. Construindo o Motor de Paginação no Python (`app/utils/pagination.py`)

Crie o arquivo `app/utils/pagination.py`:

```python
# app/utils/pagination.py
import math
from flask import request, current_app
from app.database import query_one, query_all


def obter_parametros_listagem():
    """
    Extrai da query string (?q=...&page=...&per_page=...) os parâmetros
    padronizados usados em todas as tabelas do sistema.
    """
    busca = request.args.get("q", "").strip()
    try:
        pagina = max(1, int(request.args.get("page", 1)))
    except ValueError:
        pagina = 1

    padrao_por_pagina = current_app.config.get("ITEMS_PER_PAGE", 8)
    try:
        por_pagina = int(request.args.get("per_page", padrao_por_pagina))
        if por_pagina not in (5, 8, 10, 20, 50):
            por_pagina = padrao_por_pagina
    except ValueError:
        por_pagina = padrao_por_pagina

    return busca, pagina, por_pagina


def paginar_consulta(sql_base, params=None, pagina=1, por_pagina=8, ordenacao=""):
    """
    Executa a contagem total e a busca paginada no MySQL a partir de uma query SQL base.
    Retorna um dicionário com os itens e todos os metadados de paginação.
    """
    parametros = list(params or [])

    # 1. Conta o total de registros que atendem ao filtro da pesquisa dinâmica
    sql_contagem = f"SELECT COUNT(*) AS total FROM ({sql_base}) AS subconsulta_total"
    resultado_total = query_one(sql_contagem, tuple(parametros))
    total_registros = resultado_total["total"] if resultado_total else 0

    total_paginas = max(1, math.ceil(total_registros / por_pagina))
    if pagina > total_paginas:
        pagina = total_paginas

    offset = (pagina - 1) * por_pagina

    # 2. Busca apenas a fatia (página) solicitada
    clausula_ordem = f" ORDER BY {ordenacao} " if ordenacao else " "
    sql_paginado = f"{sql_base}{clausula_ordem}LIMIT %s OFFSET %s"
    itens = query_all(sql_paginado, tuple(parametros + [por_pagina, offset]))

    inicio_item = offset + 1 if total_registros > 0 else 0
    fim_item = min(offset + por_pagina, total_registros)

    # 3. Calcula janela de páginas visíveis nos botões (ex: [1, 2, 3, 4, 5])
    inicio_janela = max(1, pagina - 2)
    fim_janela = min(total_paginas, pagina + 2)
    paginas_visiveis = list(range(inicio_janela, fim_janela + 1))

    return {
        "itens": itens,
        "pagina": pagina,
        "por_pagina": por_pagina,
        "total_registros": total_registros,
        "total_paginas": total_paginas,
        "tem_anterior": pagina > 1,
        "tem_proxima": pagina < total_paginas,
        "pagina_anterior": pagina - 1,
        "proxima_pagina": pagina + 1,
        "inicio_item": inicio_item,
        "fim_item": fim_item,
        "paginas_visiveis": paginas_visiveis,
    }
```

---

## 3. Componente de Paginação em Tailwind CSS (`app/templates/components/paginacao.html`)

Crie a macro Jinja2 em `app/templates/components/paginacao.html`:

```html
<!-- app/templates/components/paginacao.html -->
{% macro renderizar_paginacao(paginacao, alvo_container='#container-tabela') %}
<div class="px-6 py-4 bg-slate-50 border-t border-slate-200 flex flex-col sm:flex-row items-center justify-between gap-4 text-sm no-print">
    <div class="text-slate-600">
        {% if paginacao.total_registros > 0 %}
            Exibindo <span class="font-semibold text-slate-900">{{ paginacao.inicio_item }}</span>
            a <span class="font-semibold text-slate-900">{{ paginacao.fim_item }}</span>
            de <span class="font-semibold text-indigo-600">{{ paginacao.total_registros }}</span> registros
        {% else %}
            Nenhum registro encontrado para os filtros aplicados
        {% endif %}
    </div>

    {% if paginacao.total_paginas > 1 %}
    <div class="flex items-center gap-1.5">
        <button type="button"
                data-page="{{ paginacao.pagina_anterior }}"
                data-target="{{ alvo_container }}"
                {% if not paginacao.tem_anterior %}disabled{% endif %}
                class="js-btn-pagina px-3 py-1.5 rounded-lg border text-xs font-semibold transition
                       {% if paginacao.tem_anterior %}bg-white border-slate-300 text-slate-700 hover:bg-indigo-50 hover:border-indigo-300
                       {% else %}bg-slate-100 border-slate-200 text-slate-400 cursor-not-allowed{% endif %}">
            ← Anterior
        </button>

        {% for num in paginacao.paginas_visiveis %}
        <button type="button"
                data-page="{{ num }}"
                data-target="{{ alvo_container }}"
                class="js-btn-pagina w-8 h-8 rounded-lg border text-xs font-bold transition
                       {% if num == paginacao.pagina %}bg-indigo-600 border-indigo-600 text-white shadow-sm
                       {% else %}bg-white border-slate-300 text-slate-700 hover:bg-indigo-50{% endif %}">
            {{ num }}
        </button>
        {% endfor %}

        <button type="button"
                data-page="{{ paginacao.proxima_pagina }}"
                data-target="{{ alvo_container }}"
                {% if not paginacao.tem_proxima %}disabled{% endif %}
                class="js-btn-pagina px-3 py-1.5 rounded-lg border text-xs font-semibold transition
                       {% if paginacao.tem_proxima %}bg-white border-slate-300 text-slate-700 hover:bg-indigo-50 hover:border-indigo-300
                       {% else %}bg-slate-100 border-slate-200 text-slate-400 cursor-not-allowed{% endif %}">
            Próxima →
        </button>
    </div>
    {% endif %}
</div>
{% endmacro %}
```

---

## 4. Script de Pesquisa Dinâmica e Paginação Assíncrona (`app/static/js/tabela_dinamica.js`)

Este script transforma qualquer formulário com a classe `.js-form-filtro` em um buscador dinâmico instantâneo. Conforme o usuário digita no campo de pesquisa ou altera um seletor, o script aguarda 300ms (*debounce*), busca o HTML atualizado no Flask e substitui apenas o conteúdo da tabela correspondente (inclusive quando há múltiplas tabelas na mesma página, como na tela de relatórios!).

Crie o arquivo `app/static/js/tabela_dinamica.js`:

```javascript
// app/static/js/tabela_dinamica.js
document.addEventListener("DOMContentLoaded", () => {
    let temporizadorDebounce = null;

    function carregarTabelaDinamica(form, pagina = 1) {
        const seletorAlvo = form.dataset.target || "#container-tabela";
        const container = document.querySelector(seletorAlvo);
        if (!container) return;

        const url = new URL(form.action || window.location.href, window.location.origin);
        const dadosForm = new FormData(form);

        dadosForm.forEach((valor, chave) => {
            if (valor !== "") {
                url.searchParams.set(chave, valor);
            } else {
                url.searchParams.delete(chave);
            }
        });
        url.searchParams.set("page", pagina);

        // Identifica qual tabela está sendo atualizada (útil em páginas com várias tabelas)
        if (form.dataset.tabelaId) {
            url.searchParams.set("tabela_id", form.dataset.tabelaId);
        }

        container.classList.add("opacity-60", " transition-opacity");

        fetch(url.toString(), {
            headers: { "X-Requested-With": "XMLHttpRequest" }
        })
        .then((resposta) => resposta.text())
        .then((html) => {
            const docTemp = new DOMParser().parseFromString(html, "text/html");
            const novoConteudo = docTemp.querySelector(seletorAlvo);
            if (novoConteudo) {
                container.innerHTML = novoConteudo.innerHTML;
            } else {
                container.innerHTML = html;
            }
            window.history.replaceState({}, "", url.toString());
        })
        .finally(() => {
            container.classList.remove("opacity-60");
        });
    }

    // 1. Escuta digitação nos campos de busca dinâmica com Debounce de 300ms
    document.body.addEventListener("input", (evento) => {
        if (evento.target.matches(".js-busca-dinamica")) {
            const form = evento.target.closest(".js-form-filtro");
            clearTimeout(temporizadorDebounce);
            temporizadorDebounce = setTimeout(() => {
                carregarTabelaDinamica(form, 1);
            }, 300);
        }
    });

    // 2. Escuta mudança em selects/filtros de data ou itens por página
    document.body.addEventListener("change", (evento) => {
        if (evento.target.matches(".js-filtro-imediato")) {
            const form = evento.target.closest(".js-form-filtro");
            carregarTabelaDinamica(form, 1);
        }
    });

    // 3. Escuta cliques nos botões de paginação
    document.body.addEventListener("click", (evento) => {
        const botaoPagina = evento.target.closest(".js-btn-pagina");
        if (botaoPagina && !botaoPagina.disabled) {
            evento.preventDefault();
            const seletorAlvo = botaoPagina.dataset.target || "#container-tabela";
            const form = document.querySelector(`.js-form-filtro[data-target="${seletorAlvo}"]`)
                      || document.querySelector(".js-form-filtro");
            if (form) {
                carregarTabelaDinamica(form, botaoPagina.dataset.page);
            }
        }
    });
});
```

---

## 5. Módulo de Usuários e Permissões Personalizadas (`app/blueprints/usuarios.py`)

Neste módulo, o administrador (ou qualquer usuário que tenha recebido as permissões `usuarios.visualizar`, `usuarios.gerenciar` e `usuarios.permissoes`) pode:
- Pesquisar usuários dinamicamente por **nome**, **e-mail**, **setor** ou **etiqueta de cargo**.
- Cadastrar e editar usuários.
- Abrir a matriz visual de **Permissões Personalizadas** e marcar/desmarcar permissões granulares para um usuário específico.

Crie o arquivo `app/blueprints/usuarios.py`:

```python
# app/blueprints/usuarios.py
from flask import Blueprint, render_template, request, redirect, url_for, flash
from werkzeug.security import generate_password_hash
from app.database import query_all, query_one, execute_write, get_db
from app.utils.security import permissao_obrigatoria
from app.utils.pagination import obter_parametros_listagem, paginar_consulta

usuarios_bp = Blueprint("usuarios", __name__, url_prefix="/usuarios")


@usuarios_bp.route("/")
@permissao_obrigatoria("usuarios.visualizar")
def listar():
    busca, pagina, por_pagina = obter_parametros_listagem()
    status_filtro = request.args.get("ativo", "")

    sql = """
        SELECT u.id, u.nome, u.email, u.setor, u.cargo_etiqueta, u.ativo, u.criado_em,
               COUNT(up.permissao_id) AS total_permissoes
        FROM usuarios u
        LEFT JOIN usuario_permissoes up ON up.usuario_id = u.id
        WHERE 1=1
    """
    params = []

    if busca:
        sql += " AND (u.nome LIKE %s OR u.email LIKE %s OR u.setor LIKE %s OR u.cargo_etiqueta LIKE %s)"
        termo = f"%{busca}%"
        params.extend([termo, termo, termo, termo])

    if status_filtro in ("0", "1"):
        sql += " AND u.ativo = %s"
        params.append(int(status_filtro))

    sql += " GROUP BY u.id"

    paginacao = paginar_consulta(
        sql_base=sql,
        params=params,
        pagina=pagina,
        por_pagina=por_pagina,
        ordenacao="u.nome ASC",
    )

    return render_template(
        "usuarios/index.html",
        paginacao=paginacao,
        busca=busca,
        status_filtro=status_filtro,
    )


@usuarios_bp.route("/novo", methods=["GET", "POST"])
@permissao_obrigatoria("usuarios.gerenciar")
def criar():
    todas_permissoes = query_all("SELECT * FROM permissoes ORDER BY modulo, nome")

    if request.method == "POST":
        nome = request.form.get("nome", "").strip()
        email = request.form.get("email", "").strip().lower()
        senha = request.form.get("senha", "")
        setor = request.form.get("setor", "Geral").strip()
        cargo_etiqueta = request.form.get("cargo_etiqueta", "Solicitante").strip()
        permissoes_selecionadas = request.form.getlist("permissoes_ids")

        existente = query_one("SELECT id FROM usuarios WHERE email = %s", (email,))
        if existente:
            flash("Já existe um usuário cadastrado com este e-mail.", "erro")
            return render_template("usuarios/form.html", permissoes=todas_permissoes, usuario=None)

        novo_id = execute_write(
            """
            INSERT INTO usuarios (nome, email, senha_hash, setor, cargo_etiqueta, ativo)
            VALUES (%s, %s, %s, %s, %s, 1)
            """,
            (nome, email, generate_password_hash(senha), setor, cargo_etiqueta),
        )

        # Grava as permissões personalizadas escolhidas nos checkboxes
        conn = get_db()
        cursor = conn.cursor()
        for pid in permissoes_selecionadas:
            cursor.execute(
                "INSERT IGNORE INTO usuario_permissoes (usuario_id, permissao_id) VALUES (%s, %s)",
                (novo_id, int(pid)),
            )
        conn.commit()
        cursor.close()

        flash(f"Usuário '{nome}' criado com {len(permissoes_selecionadas)} permissões personalizadas!", "sucesso")
        return redirect(url_for("usuarios.listar"))

    return render_template("usuarios/form.html", permissoes=todas_permissoes, usuario=None)


@usuarios_bp.route("/<int:usuario_id>/permissoes", methods=["GET", "POST"])
@permissao_obrigatoria("usuarios.permissoes")
def gerenciar_permissoes(usuario_id):
    usuario = query_one("SELECT * FROM usuarios WHERE id = %s", (usuario_id,))
    if not usuario:
        flash("Usuário não encontrado.", "erro")
        return redirect(url_for("usuarios.listar"))

    todas_permissoes = query_all("SELECT * FROM permissoes ORDER BY modulo, nome")
    atuais = query_all(
        "SELECT permissao_id FROM usuario_permissoes WHERE usuario_id = %s",
        (usuario_id,),
    )
    ids_atuais = {linha["permissao_id"] for linha in atuais}

    if request.method == "POST":
        novos_ids = [int(pid) for pid in request.form.getlist("permissoes_ids")]
        cargo_etiqueta = request.form.get("cargo_etiqueta", usuario["cargo_etiqueta"]).strip()
        ativo = 1 if request.form.get("ativo") == "1" else 0

        conn = get_db()
        cursor = conn.cursor()
        try:
            cursor.execute(
                "UPDATE usuarios SET cargo_etiqueta = %s, ativo = %s WHERE id = %s",
                (cargo_etiqueta, ativo, usuario_id),
            )
            cursor.execute("DELETE FROM usuario_permissoes WHERE usuario_id = %s", (usuario_id,))
            for pid in novos_ids:
                cursor.execute(
                    "INSERT INTO usuario_permissoes (usuario_id, permissao_id) VALUES (%s, %s)",
                    (usuario_id, pid),
                )
            conn.commit()
        finally:
            cursor.close()

        flash(f"Permissões personalizadas de {usuario['nome']} atualizadas com sucesso!", "sucesso")
        return redirect(url_for("usuarios.listar"))

    # Agrupa permissões por módulo para exibir em cards organizados no Tailwind CSS
    permissoes_por_modulo = {}
    for p in todas_permissoes:
        permissoes_por_modulo.setdefault(p["modulo"], []).append(p)

    return render_template(
        "usuarios/permissoes.html",
        usuario=usuario,
        permissoes_por_modulo=permissoes_por_modulo,
        ids_atuais=ids_atuais,
    )
```

### Template da Tabela de Usuários com Busca Dinâmica (`app/templates/usuarios/index.html`)

```html
<!-- app/templates/usuarios/index.html -->
{% extends "base.html" %}
{% from "components/paginacao.html" import renderizar_paginacao %}
{% block titulo %}Usuários e Permissões — SalaFácil{% endblock %}

{% block conteudo %}
<div class="flex flex-col sm:flex-row sm:items-center justify-between gap-4 mb-6">
    <div>
        <h2 class="text-2xl font-bold text-slate-900">Usuários & Permissões Personalizadas</h2>
        <p class="text-sm text-slate-500">Gerencie contas e configure permissões granulares por usuário</p>
    </div>
    {% if tem_permissao('usuarios.gerenciar') %}
    <a href="{{ url_for('usuarios.criar') }}"
       class="px-4 py-2.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold rounded-xl shadow-sm transition">
        + Novo Usuário
    </a>
    {% endif %}
</div>

<!-- Barra de Pesquisa Dinâmica e Filtros -->
<form method="GET" action="{{ url_for('usuarios.listar') }}"
      data-target="#container-tabela-usuarios"
      class="js-form-filtro bg-white p-4 rounded-2xl shadow-sm border border-slate-200 mb-6 flex flex-col md:flex-row gap-4">
    <div class="flex-1">
        <input type="text" name="q" value="{{ busca }}"
               placeholder="Pesquisar dinamicamente por nome, e-mail, setor ou cargo..."
               class="js-busca-dinamica w-full px-4 py-2 rounded-xl border border-slate-300 focus:ring-2 focus:ring-indigo-500 outline-none text-sm">
    </div>
    <select name="ativo" class="js-filtro-imediato px-3 py-2 rounded-xl border border-slate-300 text-sm bg-white">
        <option value="">Todos os Status</option>
        <option value="1" {% if status_filtro == '1' %}selected{% endif %}>Somente Ativos</option>
        <option value="0" {% if status_filtro == '0' %}selected{% endif %}>Somente Inativos</option>
    </select>
    <select name="per_page" class="js-filtro-imediato px-3 py-2 rounded-xl border border-slate-300 text-sm bg-white">
        {% for qtd in [5, 8, 10, 20] %}
        <option value="{{ qtd }}" {% if paginacao.por_pagina == qtd %}selected{% endif %}>{{ qtd }} por página</option>
        {% endfor %}
    </select>
</form>

<!-- Container da Tabela Paginada -->
<div id="container-tabela-usuarios" class="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
    <div class="overflow-x-auto">
        <table class="w-full text-left border-collapse">
            <thead>
                <tr class="bg-slate-50 border-b border-slate-200 text-xs font-bold text-slate-600 uppercase">
                    <th class="py-3.5 px-6">Nome / E-mail</th>
                    <th class="py-3.5 px-6">Setor</th>
                    <th class="py-3.5 px-6">Perfil Base</th>
                    <th class="py-3.5 px-6 text-center">Permissões Ativas</th>
                    <th class="py-3.5 px-6 text-center">Status</th>
                    <th class="py-3.5 px-6 text-right">Ações</th>
                </tr>
            </thead>
            <tbody class="divide-y divide-slate-200 text-sm">
                {% for u in paginacao.itens %}
                <tr class="hover:bg-slate-50/80 transition">
                    <td class="py-4 px-6">
                        <p class="font-semibold text-slate-900">{{ u.nome }}</p>
                        <p class="text-xs text-slate-500">{{ u.email }}</p>
                    </td>
                    <td class="py-4 px-6 text-slate-700">{{ u.setor }}</td>
                    <td class="py-4 px-6">
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-indigo-50 text-indigo-700">
                            {{ u.cargo_etiqueta }}
                        </span>
                    </td>
                    <td class="py-4 px-6 text-center">
                        <span class="px-2.5 py-1 rounded-lg bg-slate-100 text-slate-800 font-bold text-xs">
                            {{ u.total_permissoes }} chaves
                        </span>
                    </td>
                    <td class="py-4 px-6 text-center">
                        {% if u.ativo %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-emerald-100 text-emerald-800">Ativo</span>
                        {% else %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-rose-100 text-rose-800">Inativo</span>
                        {% endif %}
                    </td>
                    <td class="py-4 px-6 text-right">
                        {% if tem_permissao('usuarios.permissoes') %}
                        <a href="{{ url_for('usuarios.gerenciar_permissoes', usuario_id=u.id) }}"
                           class="inline-flex items-center gap-1 px-3 py-1.5 rounded-lg bg-indigo-50 hover:bg-indigo-100 text-indigo-700 text-xs font-semibold transition">
                            🔐 Permissões
                        </a>
                        {% endif %}
                    </td>
                </tr>
                {% else %}
                <tr>
                    <td colspan="6" class="py-10 text-center text-slate-500">
                        Nenhum usuário encontrado para "{{ busca }}".
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>

    {{ renderizar_paginacao(paginacao, '#container-tabela-usuarios') }}
</div>
{% endblock %}
```

### Template de Configuração de Permissões Personalizadas (`app/templates/usuarios/permissoes.html`)

```html
<!-- app/templates/usuarios/permissoes.html -->
{% extends "base.html" %}
{% block titulo %}Permissões de {{ usuario.nome }} — SalaFácil{% endblock %}

{% block conteudo %}
<div class="mb-6 flex items-center justify-between">
    <div>
        <h2 class="text-2xl font-bold text-slate-900">Permissões Personalizadas: {{ usuario.nome }}</h2>
        <p class="text-sm text-slate-500">{{ usuario.email }} • Setor: {{ usuario.setor }}</p>
    </div>
    <a href="{{ url_for('usuarios.listar') }}" class="text-sm text-slate-600 hover:text-slate-900 font-medium">
        ← Voltar para Usuários
    </a>
</div>

<form method="POST" class="space-y-6">
    <div class="bg-white p-6 rounded-2xl shadow-sm border border-slate-200 grid grid-cols-1 md:grid-cols-2 gap-6">
        <div>
            <label class="block text-sm font-semibold text-slate-700 mb-1">Etiqueta de Cargo / Função</label>
            <input type="text" name="cargo_etiqueta" value="{{ usuario.cargo_etiqueta }}"
                   class="w-full px-4 py-2 rounded-xl border border-slate-300 text-sm">
        </div>
        <div>
            <label class="block text-sm font-semibold text-slate-700 mb-1">Status da Conta</label>
            <select name="ativo" class="w-full px-4 py-2 rounded-xl border border-slate-300 text-sm bg-white">
                <option value="1" {% if usuario.ativo %}selected{% endif %}>Conta Ativa (Pode fazer login)</option>
                <option value="0" {% if not usuario.ativo %}selected{% endif %}>Conta Bloqueada / Inativa</option>
            </select>
        </div>
    </div>

    <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
        {% for modulo, lista_permissoes in permissoes_por_modulo.items() %}
        <div class="bg-white p-6 rounded-2xl shadow-sm border border-slate-200">
            <h3 class="font-bold text-base text-indigo-700 border-b border-slate-100 pb-3 mb-4">
                Módulo: {{ modulo }}
            </h3>
            <div class="space-y-3">
                {% for perm in lista_permissoes %}
                <label class="flex items-start gap-3 p-2.5 rounded-xl hover:bg-slate-50 cursor-pointer border border-transparent hover:border-slate-200 transition">
                    <input type="checkbox" name="permissoes_ids" value="{{ perm.id }}"
                           {% if perm.id in ids_atuais %}checked{% endif %}
                           class="mt-1 h-4 w-4 rounded border-slate-300 text-indigo-600 focus:ring-indigo-500">
                    <div>
                        <p class="text-sm font-semibold text-slate-900">
                            {{ perm.nome }}
                            <code class="text-[11px] bg-slate-100 text-slate-600 px-1.5 py-0.5 rounded ml-1">{{ perm.codigo }}</code>
                        </p>
                        <p class="text-xs text-slate-500">{{ perm.descricao }}</p>
                    </div>
                </label>
                {% endfor %}
            </div>
        </div>
        {% endfor %}
    </div>

    <div class="flex justify-end gap-3">
        <button type="submit"
                class="px-6 py-2.5 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold rounded-xl shadow-sm text-sm transition">
            Salvar Permissões Personalizadas
        </button>
    </div>
</form>
{% endblock %}
```

---

## 6. Módulo de Catálogo de Salas (`app/blueprints/salas.py`)

O módulo de Salas também utiliza nosso motor de paginação e pesquisa dinâmica, permitindo filtrar salas instantaneamente por **código (`LAB-101`)**, **nome**, **bloco** ou **recursos (`Projetor`, `Ar-condicionado`)**, além de filtrar por capacidade mínima.

Crie o arquivo `app/blueprints/salas.py`:

```python
# app/blueprints/salas.py
from flask import Blueprint, render_template, request, redirect, url_for, flash
from app.database import query_one, execute_write
from app.utils.security import permissao_obrigatoria
from app.utils.pagination import obter_parametros_listagem, paginar_consulta

salas_bp = Blueprint("salas", __name__, url_prefix="/salas")


@salas_bp.route("/")
@permissao_obrigatoria("salas.visualizar")
def listar():
    busca, pagina, por_pagina = obter_parametros_listagem()
    capacidade_min = request.args.get("capacidade_min", "").strip()

    sql = "SELECT * FROM salas WHERE 1=1"
    params = []

    if busca:
        sql += " AND (codigo LIKE %s OR nome LIKE %s OR bloco LIKE %s OR recursos LIKE %s)"
        termo = f"%{busca}%"
        params.extend([termo, termo, termo, termo])

    if capacidade_min.isdigit():
        sql += " AND capacidade >= %s"
        params.append(int(capacidade_min))

    paginacao = paginar_consulta(
        sql_base=sql,
        params=params,
        pagina=pagina,
        por_pagina=por_pagina,
        ordenacao="codigo ASC",
    )

    return render_template(
        "salas/index.html",
        paginacao=paginacao,
        busca=busca,
        capacidade_min=capacidade_min,
    )


@salas_bp.route("/nova", methods=["GET", "POST"])
@permissao_obrigatoria("salas.criar")
def criar():
    if request.method == "POST":
        codigo = request.form.get("codigo", "").strip().upper()
        nome = request.form.get("nome", "").strip()
        bloco = request.form.get("bloco", "").strip()
        capacidade = int(request.form.get("capacidade", 10))
        recursos = request.form.get("recursos", "").strip()

        execute_write(
            """
            INSERT INTO salas (codigo, nome, bloco, capacidade, recursos, ativa)
            VALUES (%s, %s, %s, %s, %s, 1)
            """,
            (codigo, nome, bloco, capacidade, recursos),
        )
        flash(f"Sala {codigo} ({nome}) cadastrada com sucesso!", "sucesso")
        return redirect(url_for("salas.listar"))

    return render_template("salas/form.html", sala=None)


@salas_bp.route("/<int:sala_id>/editar", methods=["GET", "POST"])
@permissao_obrigatoria("salas.editar")
def editar(sala_id):
    sala = query_one("SELECT * FROM salas WHERE id = %s", (sala_id,))
    if not sala:
        flash("Sala não encontrada.", "erro")
        return redirect(url_for("salas.listar"))

    if request.method == "POST":
        nome = request.form.get("nome", "").strip()
        bloco = request.form.get("bloco", "").strip()
        capacidade = int(request.form.get("capacidade", 10))
        recursos = request.form.get("recursos", "").strip()
        ativa = 1 if request.form.get("ativa") == "1" else 0

        execute_write(
            """
            UPDATE salas
            SET nome = %s, bloco = %s, capacidade = %s, recursos = %s, ativa = %s
            WHERE id = %s
            """,
            (nome, bloco, capacidade, recursos, ativa, sala_id),
        )
        flash(f"Sala {sala['codigo']} atualizada com sucesso!", "sucesso")
        return redirect(url_for("salas.listar"))

    return render_template("salas/form.html", sala=sala)
```

### Template da Tabela de Salas (`app/templates/salas/index.html`)

```html
<!-- app/templates/salas/index.html -->
{% extends "base.html" %}
{% from "components/paginacao.html" import renderizar_paginacao %}
{% block titulo %}Catálogo de Salas — SalaFácil{% endblock %}

{% block conteudo %}
<div class="flex flex-col sm:flex-row sm:items-center justify-between gap-4 mb-6">
    <div>
        <h2 class="text-2xl font-bold text-slate-900">Catálogo de Salas e Laboratórios</h2>
        <p class="text-sm text-slate-500">Pesquise salas por código, bloco, capacidade ou equipamentos disponíveis</p>
    </div>
    {% if tem_permissao('salas.criar') %}
    <a href="{{ url_for('salas.criar') }}"
       class="px-4 py-2.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold rounded-xl shadow-sm transition">
        + Cadastrar Sala
    </a>
    {% endif %}
</div>

<form method="GET" action="{{ url_for('salas.listar') }}"
      data-target="#container-tabela-salas"
      class="js-form-filtro bg-white p-4 rounded-2xl shadow-sm border border-slate-200 mb-6 flex flex-col md:flex-row gap-4">
    <div class="flex-1">
        <input type="text" name="q" value="{{ busca }}"
               placeholder="Pesquisar sala por código (ex: LAB-101), nome, bloco ou recurso (ex: Projetor)..."
               class="js-busca-dinamica w-full px-4 py-2 rounded-xl border border-slate-300 focus:ring-2 focus:ring-indigo-500 outline-none text-sm">
    </div>
    <div class="w-full md:w-48">
        <input type="number" name="capacidade_min" value="{{ capacidade_min }}" min="1"
               placeholder="Capacidade mín."
               class="js-busca-dinamica w-full px-4 py-2 rounded-xl border border-slate-300 text-sm">
    </div>
    <select name="per_page" class="js-filtro-imediato px-3 py-2 rounded-xl border border-slate-300 text-sm bg-white">
        {% for qtd in [5, 8, 10, 20] %}
        <option value="{{ qtd }}" {% if paginacao.por_pagina == qtd %}selected{% endif %}>{{ qtd }} por página</option>
        {% endfor %}
    </select>
</form>

<div id="container-tabela-salas" class="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
    <div class="overflow-x-auto">
        <table class="w-full text-left border-collapse">
            <thead>
                <tr class="bg-slate-50 border-b border-slate-200 text-xs font-bold text-slate-600 uppercase">
                    <th class="py-3.5 px-6">Código</th>
                    <th class="py-3.5 px-6">Sala / Localização</th>
                    <th class="py-3.5 px-6 text-center">Capacidade</th>
                    <th class="py-3.5 px-6">Recursos e Equipamentos</th>
                    <th class="py-3.5 px-6 text-center">Status</th>
                    <th class="py-3.5 px-6 text-right">Ações</th>
                </tr>
            </thead>
            <tbody class="divide-y divide-slate-200 text-sm">
                {% for s in paginacao.itens %}
                <tr class="hover:bg-slate-50/80 transition">
                    <td class="py-4 px-6 font-mono font-bold text-indigo-700">{{ s.codigo }}</td>
                    <td class="py-4 px-6">
                        <p class="font-semibold text-slate-900">{{ s.nome }}</p>
                        <p class="text-xs text-slate-500">{{ s.bloco }}</p>
                    </td>
                    <td class="py-4 px-6 text-center font-semibold text-slate-800">{{ s.capacidade }} lugares</td>
                    <td class="py-4 px-6 text-slate-600 text-xs">{{ s.recursos }}</td>
                    <td class="py-4 px-6 text-center">
                        {% if s.ativa %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-emerald-100 text-emerald-800">Disponível</span>
                        {% else %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-amber-100 text-amber-800">Manutenção</span>
                        {% endif %}
                    </td>
                    <td class="py-4 px-6 text-right space-x-2">
                        {% if tem_permissao('reservas.criar') and s.ativa %}
                        <a href="{{ url_for('reservas.criar', sala_id=s.id) }}"
                           class="px-3 py-1.5 rounded-lg bg-indigo-600 hover:bg-indigo-700 text-white text-xs font-semibold transition">
                            Reservar
                        </a>
                        {% endif %}
                        {% if tem_permissao('salas.editar') %}
                        <a href="{{ url_for('salas.editar', sala_id=s.id) }}"
                           class="px-3 py-1.5 rounded-lg bg-slate-100 hover:bg-slate-200 text-slate-700 text-xs font-semibold transition">
                            Editar
                        </a>
                        {% endif %}
                    </td>
                </tr>
                {% else %}
                <tr>
                    <td colspan="6" class="py-10 text-center text-slate-500">
                        Nenhuma sala encontrada com os critérios informados.
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>

    {{ renderizar_paginacao(paginacao, '#container-tabela-salas') }}
</div>
{% endblock %}
```

---

## 7. Exercícios de Fixação da Aula 04

1. Acesse `/usuarios`, clique em **Permissões** na usuária `Beatriz Professora` e conceda a ela a permissão `salas.editar`. Em outra janela anônima logada com a Beatriz, verifique que o botão **Editar** passa a aparecer imediatamente na tabela de salas!
2. Na **Aula 05**, implementaremos o coração operacional do sistema: a **Gestão de Reservas de Salas** com algoritmo SQL anti-conflito de horários, fluxo de aprovação e busca dinâmica paginada.
