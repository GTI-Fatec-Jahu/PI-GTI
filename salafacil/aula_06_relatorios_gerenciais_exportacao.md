# Aula 02 — Banco de Dados MySQL e Modelagem de Permissões Personalizadas

## 1. Arquitetura de Dados do Sistema de Reservas de Salas

Nesta aula vamos projetar e implementar o banco de dados **MySQL** do nosso sistema. Um requisito central do projeto é que **os usuários possuam permissões personalizadas**.

### Por que Permissões Personalizadas (Granulares) em vez de Apenas "Cargos Fixos"?

Em muitos sistemas básicos, o controle de acesso verifica apenas se `usuario.cargo == 'admin'`. Porém, no mundo real:

* Um **recepcionista** pode precisar cadastrar reservas e aprovar reservas de salas de reunião, mas **não** pode excluir salas nem gerenciar usuários.

* Um **coordenador** pode precisar visualizar e exportar relatórios gerenciais e aprovar reservas, mas **não** pode alterar permissões de outros usuários.

* Um **estagiário administrativo** pode receber temporariamente a permissão específica `salas.editar` sem se tornar administrador do sistema.

Para resolver isso de forma elegante no MySQL, implementaremos uma modelagem híbrida:

1. Cada usuário possui um **perfil base informativo** (`cargo`: ex. *Administrador*, *Gestor de Espaços*, *Solicitante*), que serve para pré-selecionar um conjunto padrão de permissões no momento do cadastro.

2. As permissões efetivas são gravadas de maneira granular na tabela associativa `usuario_permissoes`, vinculando diretamente `usuario_id` a cada `permissao_id` (ex.: `salas.criar`, `reservas.aprovar`, `relatorios.exportar`). Assim, o administrador pode marcar ou desmarcar qualquer permissão individualmente para qualquer usuário!

## 2. Visão Geral das Tabelas do Banco (`gestao_reservas_salas`)

| Tabela | Responsabilidade | Relacionamentos Principais | 
| ----- | ----- | ----- | 
| `usuarios` | Armazena dados cadastrais, hash de senha e status ativo/inativo. | `1:N` com `usuario_permissoes` e `1:N` com `reservas`. | 
| `permissoes` | Catálogo de ações protegidas do sistema, organizadas por `modulo` e `codigo` único. | `N:M` com `usuarios` através de `usuario_permissoes`. | 
| `usuario_permissoes` | Tabela pivô que define exatamente quais permissões personalizadas cada usuário possui. | Chaves estrangeiras para `usuarios(id)` e `permissoes(id)`. | 
| `salas` | Cadastro das salas físicas (código, bloco, capacidade, recursos multimídia e status). | `1:N` com `reservas`. | 
| `reservas` | Agendamentos com data, hora inicial, hora final, finalidade, status e responsável pela aprovação. | Chaves estrangeiras para `salas(id)` e `usuarios(id)`. | 

## 3. Script SQL Completo de Criação do Esquema (`sql/schema.sql`)

Crie o arquivo `sql/schema.sql` com a definição das tabelas e dos **índices compostos**, projetados especificamente para acelerar nossas consultas de **pesquisa dinâmica** (`LIKE`), **paginação** (`LIMIT/OFFSET`) e **verificação de conflito de horários**:

```
-- sql/schema.sql
CREATE DATABASE IF NOT EXISTS gestao_reservas_salas
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE gestao_reservas_salas;

-- 1. Tabela de Usuários
CREATE TABLE IF NOT EXISTS usuarios (
    id INT AUTO_INCREMENT PRIMARY KEY,
    nome VARCHAR(120) NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    senha_hash VARCHAR(255) NOT NULL,
    setor VARCHAR(80) NOT NULL DEFAULT 'Geral',
    cargo_etiqueta VARCHAR(60) NOT NULL DEFAULT 'Solicitante',
    ativo TINYINT(1) NOT NULL DEFAULT 1,
    criado_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    atualizado_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_usuarios_busca (nome, email, setor, cargo_etiqueta)
) ENGINE=InnoDB;

-- 2. Catálogo de Permissões do Sistema
CREATE TABLE IF NOT EXISTS permissoes (
    id INT AUTO_INCREMENT PRIMARY KEY,
    codigo VARCHAR(60) NOT NULL UNIQUE,
    nome VARCHAR(100) NOT NULL,
    modulo VARCHAR(50) NOT NULL,
    descricao VARCHAR(255) NOT NULL,
    INDEX idx_permissoes_modulo (modulo, codigo)
) ENGINE=InnoDB;

-- 3. Tabela Associativa de Permissões Personalizadas por Usuário
CREATE TABLE IF NOT EXISTS usuario_permissoes (
    usuario_id INT NOT NULL,
    permissao_id INT NOT NULL,
    concedido_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (usuario_id, permissao_id),
    CONSTRAINT fk_up_usuario
        FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
        ON DELETE CASCADE,
    CONSTRAINT fk_up_permissao
        FOREIGN KEY (permissao_id) REFERENCES permissoes(id)
        ON DELETE CASCADE
) ENGINE=InnoDB;

-- 4. Tabela de Salas Disponíveis para Reserva
CREATE TABLE IF NOT EXISTS salas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    codigo VARCHAR(25) NOT NULL UNIQUE,
    nome VARCHAR(120) NOT NULL,
    bloco VARCHAR(60) NOT NULL,
    capacidade INT NOT NULL DEFAULT 10,
    recursos VARCHAR(255) NOT NULL DEFAULT '',
    ativa TINYINT(1) NOT NULL DEFAULT 1,
    criado_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_salas_busca (nome, codigo, bloco, recursos)
) ENGINE=InnoDB;

-- 5. Tabela de Reservas de Salas
CREATE TABLE IF NOT EXISTS reservas (
    id INT AUTO_INCREMENT PRIMARY KEY,
    sala_id INT NOT NULL,
    usuario_id INT NOT NULL,
    aprovador_id INT NULL,
    titulo VARCHAR(150) NOT NULL,
    finalidade VARCHAR(255) NOT NULL,
    data_reserva DATE NOT NULL,
    hora_inicio TIME NOT NULL,
    hora_fim TIME NOT NULL,
    participantes_estimados INT NOT NULL DEFAULT 1,
    status ENUM('pendente', 'aprovada', 'rejeitada', 'cancelada') NOT NULL DEFAULT 'pendente',
    observacoes TEXT NULL,
    criado_em DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_reserva_sala
        FOREIGN KEY (sala_id) REFERENCES salas(id)
        ON DELETE RESTRICT,
    CONSTRAINT fk_reserva_usuario
        FOREIGN KEY (usuario_id) REFERENCES usuarios(id)
        ON DELETE RESTRICT,
    CONSTRAINT fk_reserva_aprovador
        FOREIGN KEY (aprovador_id) REFERENCES usuarios(id)
        ON DELETE SET NULL,
    INDEX idx_reservas_conflito (sala_id, data_reserva, status, hora_inicio, hora_fim),
    INDEX idx_reservas_listagem (data_reserva, status, titulo)
) ENGINE=InnoDB;

```

## 4. Catálogo de Permissões Personalizadas do Sistema

Nossa tabela `permissoes` contará com **12 permissões granulares** divididas em 4 módulos. Veja como cada uma atua na prática:

| Código da Permissão | Módulo | Nome Amigável | O que Permite Fazer no Sistema | 
| ----- | ----- | ----- | ----- | 
| `salas.visualizar` | Salas | Visualizar Salas | Consultar a tabela paginada de salas e seus recursos. | 
| `salas.criar` | Salas | Cadastrar Salas | Adicionar novas salas físicas ao catálogo. | 
| `salas.editar` | Salas | Editar Salas | Alterar capacidade, recursos e status ativo/inativo de uma sala. | 
| `salas.excluir` | Salas | Excluir Salas | Remover salas que não possuam histórico de reservas. | 
| `reservas.visualizar` | Reservas | Visualizar Reservas | Consultar a agenda geral de reservas de salas. | 
| `reservas.criar` | Reservas | Solicitar Reserva | Criar novos agendamentos de salas verificando conflitos. | 
| `reservas.aprovar` | Reservas | Aprovar/Rejeitar Reservas | Aprovar ou rejeitar solicitações pendentes de qualquer usuário. | 
| `reservas.cancelar_qualquer` | Reservas | Cancelar Qualquer Reserva | Cancelar reservas de terceiros (sem essa permissão, o usuário só cancela as próprias). | 
| `usuarios.visualizar` | Usuários | Visualizar Usuários | Listar usuários cadastrados e consultar seus acessos. | 
| `usuarios.gerenciar` | Usuários | Criar/Editar Usuários | Cadastrar novos usuários, alterar dados e ativar/desativar contas. | 
| `usuarios.permissoes` | Usuários | Configurar Permissões | Marcar/desmarcar permissões personalizadas individuais de cada usuário. | 
| `relatorios.visualizar` | Relatórios | Acessar Relatórios | Visualizar indicadores, relatórios de ocupação e auditoria. | 
| `relatorios.exportar` | Relatórios | Exportar Relatórios | Baixar relatórios filtrados em CSV ou acionar modo de impressão oficial. | 

## 5. Conexão Segura entre Flask e MySQL (`app/database.py`)

Em vez de abrir e fechar conexões manualmente dentro de cada rota (o que pode vazar conexões caso ocorra uma exceção), utilizamos o objeto de contexto `g` do Flask e o gancho `teardown_appcontext`. Assim, uma única conexão é reutilizada durante a requisição HTTP e encerrada automaticamente ao final.

Crie o arquivo `app/database.py`:

```
# app/database.py
import mysql.connector
from mysql.connector import Error
from flask import current_app, g


def get_db():
    """
    Retorna a conexão MySQL ativa para a requisição atual.
    Se ainda não existir no objeto 'g' do Flask, cria uma nova conexão.
    """
    if "db_conn" not in g or not g.db_conn.is_connected():
        g.db_conn = mysql.connector.connect(
            host=current_app.config["DB_HOST"],
            port=current_app.config["DB_PORT"],
            user=current_app.config["DB_USER"],
            password=current_app.config["DB_PASSWORD"],
            database=current_app.config["DB_NAME"],
            charset="utf8mb4",
            autocommit=False,
        )
    return g.db_conn


def query_all(sql, params=None):
    """Executa um SELECT e retorna uma lista de dicionários."""
    conn = get_db()
    cursor = conn.cursor(dictionary=True)
    try:
        cursor.execute(sql, params or ())
        return cursor.fetchall()
    finally:
        cursor.close()


def query_one(sql, params=None):
    """Executa um SELECT e retorna apenas um dicionário (ou None)."""
    conn = get_db()
    cursor = conn.cursor(dictionary=True)
    try:
        cursor.execute(sql, params or ())
        return cursor.fetchone()
    finally:
        cursor.close()


def execute_write(sql, params=None, commit=True):
    """
    Executa INSERT, UPDATE ou DELETE.
    Retorna o lastrowid (útil em INSERTs) ou o número de linhas afetadas.
    """
    conn = get_db()
    cursor = conn.cursor()
    try:
        cursor.execute(sql, params or ())
        if commit:
            conn.commit()
        return cursor.lastrowid or cursor.rowcount
    except Error:
        conn.rollback()
        raise
    finally:
        cursor.close()


def close_db(exception=None):
    """Fecha a conexão MySQL ao final do ciclo de vida da requisição HTTP."""
    db_conn = g.pop("db_conn", None)
    if db_conn is not None and db_conn.is_connected():
        db_conn.close()


def init_db_app(app):
    """Registra o fechamento automático da conexão no Flask."""
    app.teardown_appcontext(close_db)

```

> **Por que sempre usar `params` (`%s`) em vez de `f-strings` no SQL?** Passar os parâmetros separadamente como tupla (`cursor.execute(sql, params)`) instrui o driver do MySQL a escapar todos os valores com segurança, protegendo 100% das nossas buscas dinâmicas e formulários contra ataques de **SQL Injection**.

## 6. Script de Inicialização e Carga de Dados (`seed_db.py`)

Para facilitar a criação das tabelas, o cadastro das 13 permissões granulares e a geração de usuários, salas e reservas de teste com senhas já criptografadas pelo `werkzeug.security`, crie o script `seed_db.py` na raiz do projeto:

```
# seed_db.py
import os
import mysql.connector
from dotenv import load_dotenv
from werkzeug.security import generate_password_hash

load_dotenv()


def executar_seed():
    # 1. Conecta ao servidor MySQL (sem especificar database inicialmente para criá-lo)
    conn = mysql.connector.connect(
        host=os.getenv("DB_HOST", "127.0.0.1"),
        port=int(os.getenv("DB_PORT", 3306)),
        user=os.getenv("DB_USER", "root"),
        password=os.getenv("DB_PASSWORD", ""),
    )
    cursor = conn.cursor()

    # 2. Lê e executa o arquivo sql/schema.sql
    with open("sql/schema.sql", "r", encoding="utf-8") as f:
        comandos_sql = f.read().split(";")
        for comando in comandos_sql:
            if comando.strip():
                cursor.execute(comando)
    conn.commit()

    # 3. Insere o catálogo de Permissões Personalizadas
    permissoes = [
        ("salas.visualizar", "Visualizar Salas", "Salas", "Permite listar e pesquisar salas"),
        ("salas.criar", "Cadastrar Salas", "Salas", "Permite criar novas salas no sistema"),
        ("salas.editar", "Editar Salas", "Salas", "Permite alterar dados e status de salas"),
        ("salas.excluir", "Excluir Salas", "Salas", "Permite remover salas sem vínculo"),
        ("reservas.visualizar", "Visualizar Reservas", "Reservas", "Permite consultar a agenda de reservas"),
        ("reservas.criar", "Solicitar Reserva", "Reservas", "Permite agendar salas"),
        ("reservas.aprovar", "Aprovar/Rejeitar Reservas", "Reservas", "Permite aprovar ou recusar pedidos"),
        ("reservas.cancelar_qualquer", "Cancelar Qualquer Reserva", "Reservas", "Permite cancelar reservas de terceiros"),
        ("usuarios.visualizar", "Visualizar Usuários", "Usuários", "Permite consultar a lista de usuários"),
        ("usuarios.gerenciar", "Criar/Editar Usuários", "Usuários", "Permite cadastrar e editar usuários"),
        ("usuarios.permissoes", "Configurar Permissões", "Usuários", "Permite alterar permissões granulares"),
        ("relatorios.visualizar", "Acessar Relatórios", "Relatórios", "Permite acessar painéis e relatórios"),
        ("relatorios.exportar", "Exportar Relatórios", "Relatórios", "Permite exportar CSV e imprimir relatórios"),
    ]

    cursor.executemany(
        """
        INSERT IGNORE INTO permissoes (codigo, nome, modulo, descricao)
        VALUES (%s, %s, %s, %s)
        """,
        permissoes,
    )

    # 4. Cria 3 usuários com diferentes conjuntos de permissões personalizadas
    senha_padrao = generate_password_hash("123456")
    usuarios = [
        ("Ana Administradora", "admin@salafacil.com", senha_padrao, "TI & Governança", "Administrador Geral"),
        ("Carlos Gestor", "gestor@salafacil.com", senha_padrao, "Infraestrutura", "Gestor de Espaços"),
        ("Beatriz Professora", "beatriz@salafacil.com", senha_padrao, "Docência", "Solicitante"),
    ]
    cursor.executemany(
        """
        INSERT IGNORE INTO usuarios (nome, email, senha_hash, setor, cargo_etiqueta)
        VALUES (%s, %s, %s, %s, %s)
        """,
        usuarios,
    )
    conn.commit()

    # 5. Atribui permissões personalizadas
    # Ana (Admin) recebe todas as permissões
    cursor.execute(
        """
        INSERT IGNORE INTO usuario_permissoes (usuario_id, permissao_id)
        SELECT u.id, p.id FROM usuarios u CROSS JOIN permissoes p
        WHERE u.email = 'admin@salafacil.com'
        """
    )

    # Carlos (Gestor) recebe permissões de salas, aprovação de reservas e relatórios
    codigos_gestor = (
        "salas.visualizar", "salas.criar", "salas.editar",
        "reservas.visualizar", "reservas.criar", "reservas.aprovar", "reservas.cancelar_qualquer",
        "relatorios.visualizar", "relatorios.exportar",
    )
    formato_in = ",".join(["%s"] * len(codigos_gestor))
    cursor.execute(
        f"""
        INSERT IGNORE INTO usuario_permissoes (usuario_id, permissao_id)
        SELECT u.id, p.id FROM usuarios u
        JOIN permissoes p ON p.codigo IN ({formato_in})
        WHERE u.email = 'gestor@salafacil.com'
        """,
        codigos_gestor,
    )

    # Beatriz (Solicitante) recebe permissões de visualização e criação de reserva
    codigos_solicitante = ("salas.visualizar", "reservas.visualizar", "reservas.criar")
    formato_sol = ",".join(["%s"] * len(codigos_solicitante))
    cursor.execute(
        f"""
        INSERT IGNORE INTO usuario_permissoes (usuario_id, permissao_id)
        SELECT u.id, p.id FROM usuarios u
        JOIN permissoes p ON p.codigo IN ({formato_sol})
        WHERE u.email = 'beatriz@salafacil.com'
        """,
        codigos_solicitante,
    )

    # 6. Insere Salas de Exemplo (para testar busca dinâmica e paginação)
    salas = [
        ("LAB-101", "Laboratório de Informática I", "Bloco A - Térreo", 40, "40 PCs, Projetor Laser, Ar-condicionado"),
        ("LAB-102", "Laboratório de Redes e Servidores", "Bloco A - 1º Andar", 25, "Bancada Cisco, Rack, Projetor, Ar-condicionado"),
        ("AUD-201", "Auditório Principal Tiradentes", "Bloco Central", 180, "Sistema de Som, 2 Telões, Palco, Videoconferência"),
        ("REU-301", "Sala de Reuniões Executiva", "Bloco B - 3º Andar", 14, "TV 75 pol, Câmera 360, Quadro de Vidro, Café"),
        ("REU-302", "Sala de Brainstorming Criativo", "Bloco B - 3º Andar", 10, "Paredes Lousa, Puffs, TV 55 pol, Ar-condicionado"),
        ("SAL-401", "Sala de Aula Invertida 01", "Bloco C - 1º Andar", 50, "Mesas Redondas, 2 Projetores, Microfone Sem Fio"),
        ("SAL-402", "Sala de Aula Multimídia 02", "Bloco C - 1º Andar", 45, "Lousa Digital Interativa, Som Ambiente"),
        ("EST-501", "Estúdio de Gravação e Podcast", "Bloco D - Subsolo", 6, "Isolamento Acústico, 4 Microfones XLR, Iluminação LED"),
        ("LAB-103", "Laboratório de Robótica e IoT", "Bloco A - 2º Andar", 30, "Impressoras 3D, Kits Arduino, Bancadas de Solda"),
        ("MIN-202", "Mini Auditório Anexo", "Bloco Central", 75, "Projetor Full HD, Púlpito Digital, Ar-condicionado"),
    ]
    cursor.executemany(
        """
        INSERT IGNORE INTO salas (codigo, nome, bloco, capacidade, recursos)
        VALUES (%s, %s, %s, %s, %s)
        """,
        salas,
    )

    conn.commit()
    cursor.close()
    conn.close()
    print("Banco de dados criado e populado com sucesso!")
    print("Logins de teste (Senha para todos: 123456):")
    print(" - admin@salafacil.com (Todas as permissões)")
    print(" - gestor@salafacil.com (Salas, Aprovação de Reservas e Relatórios)")
    print(" - beatriz@salafacil.com (Visualização e Solicitação de Reservas)")


if __name__ == "__main__":
    executar_seed()

```

Execute no terminal:

```
python seed_db.py

```

## 7. Consulta SQL das Permissões de um Usuário

Para verificar no MySQL como nossa modelagem recupera instantaneamente a lista de permissões personalizadas de qualquer usuário logado, veja a consulta que utilizaremos na próxima aula:

```
SELECT p.codigo, p.nome, p.modulo
FROM permissoes p
INNER JOIN usuario_permissoes up ON up.permissao_id = p.id
WHERE up.usuario_id = %s
ORDER BY p.modulo, p.nome;

```

## 8. Exercícios de Fixação da Aula 02

1. Execute `python seed_db.py` e conecte-se ao banco `gestao_reservas_salas` para inspecionar os registros gerados nas tabelas `permissoes` e `usuario_permissoes`.

2. Crie uma nova permissão no catálogo chamada `salas.manutencao` no módulo `Salas` e associe-a apenas ao usuário `gestor@salafacil.com`.

3. Na **Aula 03**, construiremos a camada de autenticação, o verificador de permissões personalizadas (no Python e no Jinja2) e o layout completo da aplicação com **Tailwind CSS**.