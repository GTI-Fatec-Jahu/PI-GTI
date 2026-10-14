# Aula 05 — Gestão de Reservas de Salas e Validação de Conflitos de Horário

## 1. A Regra de Ouro de um Sistema de Reservas: Prevenção de Choques de Horário

Quando dois usuários tentam reservar a mesma sala para a mesma data, como saber no MySQL se há sobreposição entre o intervalo existente (`hora_inicio`, `hora_fim`) e o novo intervalo solicitado (`novo_inicio`, `novo_fim`)?

### Lógica Matemática da Interseção de Intervalos

Dois intervalos de tempo `[A_inicio, A_fim]` e `[B_inicio, B_fim]` se sobrepõem **se, e somente se**:

$$\text{hora\_inicio\_existente} < \text{novo\_fim} \quad \text{E} \quad \text{hora\_fim\_existente} > \text{novo\_inicio}$$

Observe como essa condição cobre 100% dos cenários (sobreposição parcial no início, sobreposição parcial no fim, novo horário englobando o antigo ou novo horário totalmente dentro do antigo), mas **permite reservas adjacentes** — por exemplo, uma reserva das `08:00` às `10:00` e outra começando exatamente às `10:00` até `12:00` (`10:00 > 10:00` é falso, logo não há conflito!).

### Consulta SQL Anti-Conflito no MySQL

```sql
SELECT r.id, r.titulo, r.hora_inicio, r.hora_fim, u.nome AS solicitante
FROM reservas r
INNER JOIN usuarios u ON u.id = r.usuario_id
WHERE r.sala_id = %s
  AND r.data_reserva = %s
  AND r.status IN ('pendente', 'aprovada')
  AND r.hora_inicio < %s
  AND r.hora_fim > %s;
```

---

## 2. Implementando o Blueprint de Reservas (`app/blueprints/reservas.py`)

O módulo de reservas aplica as seguintes regras de permissão personalizada:
- Qualquer usuário com `reservas.visualizar` pode pesquisar e paginar a agenda.
- Usuários com `reservas.criar` podem solicitar reservas (se o usuário também tiver `reservas.aprovar`, sua própria reserva já nasce com status `aprovada`; caso contrário, nasce como `pendente`).
- Somente usuários com `reservas.aprovar` podem aprovar ou rejeitar reservas pendentes.
- O próprio autor da reserva pode cancelá-la; já cancelar reservas de terceiros exige a permissão personalizada `reservas.cancelar_qualquer`.

Crie o arquivo `app/blueprints/reservas.py`:

```python
# app/blueprints/reservas.py
from flask import Blueprint, render_template, request, redirect, url_for, flash, g, jsonify
from app.database import query_all, query_one, execute_write
from app.utils.security import permissao_obrigatoria, tem_permissao
from app.utils.pagination import obter_parametros_listagem, paginar_consulta

reservas_bp = Blueprint("reservas", __name__, url_prefix="/reservas")


def verificar_conflito_horario(sala_id, data_reserva, hora_inicio, hora_fim, ignorar_reserva_id=None):
    """Retorna a reserva conflitante caso haja sobreposição de horário na mesma sala e data."""
    sql = """
        SELECT r.id, r.titulo,
               TIME_FORMAT(r.hora_inicio, '%H:%i') AS inicio_fmt,
               TIME_FORMAT(r.hora_fim, '%H:%i') AS fim_fmt,
               u.nome AS solicitante
        FROM reservas r
        INNER JOIN usuarios u ON u.id = r.usuario_id
        WHERE r.sala_id = %s
          AND r.data_reserva = %s
          AND r.status IN ('pendente', 'aprovada')
          AND r.hora_inicio < %s
          AND r.hora_fim > %s
    """
    params = [sala_id, data_reserva, hora_fim, hora_inicio]
    if ignorar_reserva_id:
        sql += " AND r.id != %s"
        params.append(ignorar_reserva_id)

    return query_one(sql, tuple(params))


@reservas_bp.route("/")
@permissao_obrigatoria("reservas.visualizar")
def listar():
    busca, pagina, por_pagina = obter_parametros_listagem()
    status_filtro = request.args.get("status", "").strip()
    data_filtro = request.args.get("data", "").strip()

    sql = """
        SELECT r.id, r.titulo, r.finalidade, r.data_reserva,
               TIME_FORMAT(r.hora_inicio, '%H:%i') AS hora_inicio_fmt,
               TIME_FORMAT(r.hora_fim, '%H:%i') AS hora_fim_fmt,
               r.participantes_estimados, r.status, r.usuario_id,
               s.codigo AS sala_codigo, s.nome AS sala_nome, s.bloco AS sala_bloco,
               u.nome AS solicitante_nome, u.setor AS solicitante_setor,
               ap.nome AS aprovador_nome
        FROM reservas r
        INNER JOIN salas s ON s.id = r.sala_id
        INNER JOIN usuarios u ON u.id = r.usuario_id
        LEFT JOIN usuarios ap ON ap.id = r.aprovador_id
        WHERE 1=1
    """
    params = []

    if busca:
        sql += """
            AND (
                r.titulo LIKE %s
                OR r.finalidade LIKE %s
                OR s.codigo LIKE %s
                OR s.nome LIKE %s
                OR u.nome LIKE %s
                OR u.setor LIKE %s
            )
        """
        termo = f"%{busca}%"
        params.extend([termo, termo, termo, termo, termo, termo])

    if status_filtro in ("pendente", "aprovada", "rejeitada", "cancelada"):
        sql += " AND r.status = %s"
        params.append(status_filtro)

    if data_filtro:
        sql += " AND r.data_reserva = %s"
        params.append(data_filtro)

    paginacao = paginar_consulta(
        sql_base=sql,
        params=params,
        pagina=pagina,
        por_pagina=por_pagina,
        ordenacao="r.data_reserva DESC, r.hora_inicio ASC",
    )

    return render_template(
        "reservas/index.html",
        paginacao=paginacao,
        busca=busca,
        status_filtro=status_filtro,
        data_filtro=data_filtro,
    )


@reservas_bp.route("/nova", methods=["GET", "POST"])
@permissao_obrigatoria("reservas.criar")
def criar():
    salas_ativas = query_all("SELECT * FROM salas WHERE ativa = 1 ORDER BY codigo ASC")
    sala_pre_selecionada = request.args.get("sala_id", type=int)

    if request.method == "POST":
        sala_id = int(request.form.get("sala_id", 0))
        titulo = request.form.get("titulo", "").strip()
        finalidade = request.form.get("finalidade", "").strip()
        data_reserva = request.form.get("data_reserva", "").strip()
        hora_inicio = request.form.get("hora_inicio", "").strip()
        hora_fim = request.form.get("hora_fim", "").strip()
        participantes = int(request.form.get("participantes_estimados", 1))
        observacoes = request.form.get("observacoes", "").strip()

        # 1. Valida ordem cronológica dos horários
        if hora_inicio >= hora_fim:
            flash("O horário de término deve ser posterior ao horário de início.", "erro")
            return render_template("reservas/form.html", salas=salas_ativas, sala_selecionada=sala_id)

        # 2. Valida existência e capacidade da sala
        sala = query_one("SELECT * FROM salas WHERE id = %s AND ativa = 1", (sala_id,))
        if not sala:
            flash("A sala selecionada não está disponível.", "erro")
            return render_template("reservas/form.html", salas=salas_ativas, sala_selecionada=sala_id)

        if participantes > sala["capacidade"]:
            flash(
                f"A sala {sala['codigo']} comporta no máximo {sala['capacidade']} pessoas "
                f"(você informou {participantes}).",
                "erro",
            )
            return render_template("reservas/form.html", salas=salas_ativas, sala_selecionada=sala_id)

        # 3. Valida conflito de horário no banco MySQL
        conflito = verificar_conflito_horario(sala_id, data_reserva, hora_inicio, hora_fim)
        if conflito:
            flash(
                f"Conflito de horário! Esta sala já está reservada das {conflito['inicio_fmt']} "
                f"às {conflito['fim_fmt']} para '{conflito['titulo']}' ({conflito['solicitante']}).",
                "erro",
            )
            return render_template("reservas/form.html", salas=salas_ativas, sala_selecionada=sala_id)

        # 4. Define status inicial com base nas permissões personalizadas do usuário
        aprovacao_automatica = tem_permissao("reservas.aprovar")
        status_inicial = "aprovada" if aprovacao_automatica else "pendente"
        aprovador_id = g.usuario["id"] if aprovacao_automatica else None

        execute_write(
            """
            INSERT INTO reservas (
                sala_id, usuario_id, aprovador_id, titulo, finalidade,
                data_reserva, hora_inicio, hora_fim, participantes_estimados,
                status, observacoes
            )
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            """,
            (
                sala_id, g.usuario["id"], aprovador_id, titulo, finalidade,
                data_reserva, hora_inicio, hora_fim, participantes,
                status_inicial, observacoes,
            ),
        )

        msg = "Reserva cadastrada e aprovada automaticamente!" if aprovacao_automatica else "Solicitação enviada! Aguardando aprovação do gestor."
        flash(msg, "sucesso")
        return redirect(url_for("reservas.listar"))

    return render_template("reservas/form.html", salas=salas_ativas, sala_selecionada=sala_pre_selecionada)


@reservas_bp.route("/<int:reserva_id>/status", methods=["POST"])
@permissao_obrigatoria("reservas.visualizar")
def alterar_status(reserva_id):
    novo_status = request.form.get("novo_status", "")
    reserva = query_one("SELECT * FROM reservas WHERE id = %s", (reserva_id,))
    if not reserva:
        flash("Reserva não encontrada.", "erro")
        return redirect(url_for("reservas.listar"))

    # Aprovar ou rejeitar exige a permissão 'reservas.aprovar'
    if novo_status in ("aprovada", "rejeitada"):
        if not tem_permissao("reservas.aprovar"):
            flash("Você não possui a permissão 'reservas.aprovar'.", "erro")
            return redirect(url_for("reservas.listar"))

        execute_write(
            "UPDATE reservas SET status = %s, aprovador_id = %s WHERE id = %s",
            (novo_status, g.usuario["id"], reserva_id),
        )
        flash(f"Reserva #{reserva_id} marcada como {novo_status.upper()}.", "sucesso")

    # Cancelar permite ao próprio dono OU a quem tem 'reservas.cancelar_qualquer'
    elif novo_status == "cancelada":
        eh_dono = reserva["usuario_id"] == g.usuario["id"]
        if not (eh_dono or tem_permissao("reservas.cancelar_qualquer")):
            flash("Você só pode cancelar suas próprias reservas.", "erro")
            return redirect(url_for("reservas.listar"))

        execute_write(
            "UPDATE reservas SET status = 'cancelada' WHERE id = %s",
            (reserva_id,),
        )
        flash(f"Reserva #{reserva_id} cancelada com sucesso.", "aviso")

    return redirect(url_for("reservas.listar"))
```

---

## 3. Template da Agenda de Reservas com Busca Dinâmica (`app/templates/reservas/index.html`)

```html
<!-- app/templates/reservas/index.html -->
{% extends "base.html" %}
{% from "components/paginacao.html" import renderizar_paginacao %}
{% block titulo %}Agenda de Reservas — SalaFácil{% endblock %}

{% block conteudo %}
<div class="flex flex-col sm:flex-row sm:items-center justify-between gap-4 mb-6">
    <div>
        <h2 class="text-2xl font-bold text-slate-900">Agenda de Reservas de Salas</h2>
        <p class="text-sm text-slate-500">Consulte horários, filtre por status e gerencie aprovações em tempo real</p>
    </div>
    {% if tem_permissao('reservas.criar') %}
    <a href="{{ url_for('reservas.criar') }}"
       class="px-4 py-2.5 bg-indigo-600 hover:bg-indigo-700 text-white text-sm font-semibold rounded-xl shadow-sm transition">
        + Nova Reserva
    </a>
    {% endif %}
</div>

<!-- Barra de Pesquisa Dinâmica e Filtros -->
<form method="GET" action="{{ url_for('reservas.listar') }}"
      data-target="#container-tabela-reservas"
      class="js-form-filtro bg-white p-4 rounded-2xl shadow-sm border border-slate-200 mb-6 flex flex-col lg:flex-row gap-4">
    <div class="flex-1">
        <input type="text" name="q" value="{{ busca }}"
               placeholder="Pesquisar por evento, finalidade, código da sala, nome da sala ou solicitante..."
               class="js-busca-dinamica w-full px-4 py-2 rounded-xl border border-slate-300 focus:ring-2 focus:ring-indigo-500 outline-none text-sm">
    </div>
    <input type="date" name="data" value="{{ data_filtro }}"
           class="js-filtro-imediato px-3 py-2 rounded-xl border border-slate-300 text-sm bg-white">
    <select name="status" class="js-filtro-imediato px-3 py-2 rounded-xl border border-slate-300 text-sm bg-white">
        <option value="">Todos os Status</option>
        <option value="pendente" {% if status_filtro == 'pendente' %}selected{% endif %}>Pendentes</option>
        <option value="aprovada" {% if status_filtro == 'aprovada' %}selected{% endif %}>Aprovadas</option>
        <option value="rejeitada" {% if status_filtro == 'rejeitada' %}selected{% endif %}>Rejeitadas</option>
        <option value="cancelada" {% if status_filtro == 'cancelada' %}selected{% endif %}>Canceladas</option>
    </select>
    <select name="per_page" class="js-filtro-imediato px-3 py-2 rounded-xl border border-slate-300 text-sm bg-white">
        {% for qtd in [5, 8, 10, 20] %}
        <option value="{{ qtd }}" {% if paginacao.por_pagina == qtd %}selected{% endif %}>{{ qtd }} por página</option>
        {% endfor %}
    </select>
</form>

<!-- Tabela Paginada de Reservas -->
<div id="container-tabela-reservas" class="bg-white rounded-2xl shadow-sm border border-slate-200 overflow-hidden">
    <div class="overflow-x-auto">
        <table class="w-full text-left border-collapse">
            <thead>
                <tr class="bg-slate-50 border-b border-slate-200 text-xs font-bold text-slate-600 uppercase">
                    <th class="py-3.5 px-6">Data e Horário</th>
                    <th class="py-3.5 px-6">Sala</th>
                    <th class="py-3.5 px-6">Evento / Finalidade</th>
                    <th class="py-3.5 px-6">Solicitante</th>
                    <th class="py-3.5 px-6 text-center">Status</th>
                    <th class="py-3.5 px-6 text-right">Ações</th>
                </tr>
            </thead>
            <tbody class="divide-y divide-slate-200 text-sm">
                {% for r in paginacao.itens %}
                <tr class="hover:bg-slate-50/80 transition">
                    <td class="py-4 px-6 whitespace-nowrap">
                        <p class="font-bold text-slate-900">{{ r.data_reserva.strftime('%d/%m/%Y') }}</p>
                        <p class="text-xs font-mono text-indigo-600">{{ r.hora_inicio_fmt }} às {{ r.hora_fim_fmt }}</p>
                    </td>
                    <td class="py-4 px-6">
                        <span class="font-mono text-xs font-bold bg-slate-100 text-slate-800 px-2 py-0.5 rounded">{{ r.sala_codigo }}</span>
                        <p class="font-medium text-slate-900 mt-0.5">{{ r.sala_nome }}</p>
                    </td>
                    <td class="py-4 px-6">
                        <p class="font-semibold text-slate-900">{{ r.titulo }}</p>
                        <p class="text-xs text-slate-500">{{ r.finalidade }} • {{ r.participantes_estimados }} pessoas</p>
                    </td>
                    <td class="py-4 px-6">
                        <p class="font-medium text-slate-800">{{ r.solicitante_nome }}</p>
                        <p class="text-xs text-slate-500">{{ r.solicitante_setor }}</p>
                    </td>
                    <td class="py-4 px-6 text-center">
                        {% if r.status == 'aprovada' %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-emerald-100 text-emerald-800">Aprovada</span>
                        {% elif r.status == 'pendente' %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-amber-100 text-amber-800">Pendente</span>
                        {% elif r.status == 'rejeitada' %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-rose-100 text-rose-800">Rejeitada</span>
                        {% else %}
                        <span class="px-2.5 py-1 rounded-full text-xs font-semibold bg-slate-200 text-slate-700">Cancelada</span>
                        {% endif %}
                    </td>
                    <td class="py-4 px-6 text-right whitespace-nowrap space-x-1">
                        {% if r.status == 'pendente' and tem_permissao('reservas.aprovar') %}
                        <form method="POST" action="{{ url_for('reservas.alterar_status', reserva_id=r.id) }}" class="inline">
                            <input type="hidden" name="novo_status" value="aprovada">
                            <button type="submit" class="px-2.5 py-1 rounded-lg bg-emerald-600 hover:bg-emerald-700 text-white text-xs font-semibold">
                                ✓ Aprovar
                            </button>
                        </form>
                        <form method="POST" action="{{ url_for('reservas.alterar_status', reserva_id=r.id) }}" class="inline">
                            <input type="hidden" name="novo_status" value="rejeitada">
                            <button type="submit" class="px-2.5 py-1 rounded-lg bg-rose-600 hover:bg-rose-700 text-white text-xs font-semibold">
                                ✕ Recusar
                            </button>
                        </form>
                        {% endif %}

                        {% if r.status in ('pendente', 'aprovada') and (r.usuario_id == g.usuario.id or tem_permissao('reservas.cancelar_qualquer')) %}
                        <form method="POST" action="{{ url_for('reservas.alterar_status', reserva_id=r.id) }}" class="inline">
                            <input type="hidden" name="novo_status" value="cancelada">
                            <button type="submit" class="px-2.5 py-1 rounded-lg bg-slate-100 hover:bg-slate-200 text-slate-700 text-xs font-semibold">
                                Cancelar
                            </button>
                        </form>
                        {% endif %}
                    </td>
                </tr>
                {% else %}
                <tr>
                    <td colspan="6" class="py-10 text-center text-slate-500">
                        Nenhuma reserva encontrada para os filtros selecionados.
                    </td>
                </tr>
                {% endfor %}
            </tbody>
        </table>
    </div>

    {{ renderizar_paginacao(paginacao, '#container-tabela-reservas') }}
</div>
{% endblock %}
```

---

## 4. Formulário de Solicitação de Reserva (`app/templates/reservas/form.html`)

```html
<!-- app/templates/reservas/form.html -->
{% extends "base.html" %}
{% block titulo %}Nova Reserva de Sala — SalaFácil{% endblock %}

{% block conteudo %}
<div class="max-w-3xl mx-auto">
    <div class="mb-6 flex items-center justify-between">
        <div>
            <h2 class="text-2xl font-bold text-slate-900">Solicitar Reserva de Sala</h2>
            <p class="text-sm text-slate-500">O sistema verificará automaticamente a capacidade e conflitos de horário</p>
        </div>
        <a href="{{ url_for('reservas.listar') }}" class="text-sm text-slate-600 hover:text-slate-900 font-medium">
            ← Voltar para a Agenda
        </a>
    </div>

    <form method="POST" class="bg-white p-8 rounded-2xl shadow-sm border border-slate-200 space-y-5">
        <div class="grid grid-cols-1 md:grid-cols-2 gap-5">
            <div class="md:col-span-2">
                <label class="block text-sm font-semibold text-slate-700 mb-1">Sala Desejada</label>
                <select name="sala_id" required class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm bg-white">
                    <option value="">Selecione uma sala disponível...</option>
                    {% for s in salas %}
                    <option value="{{ s.id }}" {% if sala_selecionada == s.id %}selected{% endif %}>
                        {{ s.codigo }} — {{ s.nome }} (Até {{ s.capacidade }} lugares • {{ s.bloco }})
                    </option>
                    {% endfor %}
                </select>
            </div>

            <div class="md:col-span-2">
                <label class="block text-sm font-semibold text-slate-700 mb-1">Título do Evento / Aula / Reunião</label>
                <input type="text" name="titulo" required placeholder="Ex: Defesa de TCC / Reunião de Planejamento"
                       class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm">
            </div>

            <div>
                <label class="block text-sm font-semibold text-slate-700 mb-1">Finalidade</label>
                <input type="text" name="finalidade" required placeholder="Ex: Aula Prática, Workshop, Entrevista"
                       class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm">
            </div>

            <div>
                <label class="block text-sm font-semibold text-slate-700 mb-1">Nº Estimado de Participantes</label>
                <input type="number" name="participantes_estimados" min="1" value="10" required
                       class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm">
            </div>

            <div>
                <label class="block text-sm font-semibold text-slate-700 mb-1">Data da Reserva</label>
                <input type="date" name="data_reserva" required
                       class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm">
            </div>

            <div class="grid grid-cols-2 gap-3">
                <div>
                    <label class="block text-sm font-semibold text-slate-700 mb-1">Início</label>
                    <input type="time" name="hora_inicio" required
                           class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm">
                </div>
                <div>
                    <label class="block text-sm font-semibold text-slate-700 mb-1">Término</label>
                    <input type="time" name="hora_fim" required
                           class="w-full px-4 py-2.5 rounded-xl border border-slate-300 text-sm">
                </div>
            </div>
        </div>

        <div class="pt-4 flex justify-end gap-3">
            <button type="submit"
                    class="px-6 py-3 bg-indigo-600 hover:bg-indigo-700 text-white font-semibold rounded-xl shadow-sm text-sm transition">
                Confirmar Agendamento
            </button>
        </div>
    </form>
</div>
{% endblock %}
```

---

## 5. Exercícios de Fixação da Aula 05

1. Cadastre uma reserva no `LAB-101` para amanhã das `14:00` às `16:00`. Em seguida, tente cadastrar outra reserva no mesmo laboratório e data das `15:00` às `17:00` e valide a mensagem de bloqueio por conflito de horário.
2. Na **Aula 06**, concluiremos nosso curso construindo o módulo completo de **Relatórios Gerenciais, Indicadores e Exportação CSV** com tabelas analíticas e consolidadas dotadas de pesquisa dinâmica e paginação.
