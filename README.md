# Aula de Revisão: Construindo com HTML5 e CSS3 Passo a Passo

Olá, alunos! Bem-vindos à nossa aula de revisão. O objetivo deste material não é apenas "jogar" o código para vocês, mas sim construir o conhecimento tijolo por tijolo. 

Vamos criar uma página web do zero. Nossa missão de hoje tem duas partes principais:
1. Criar um **menu superior de navegação** que fique perfeitamente centralizado.
2. Criar um **formulário de cadastro de filmes** que explore praticamente todos os tipos de campos de entrada (`inputs`) que o HTML oferece.

---

## Etapa 1: O Esqueleto do HTML

Todo documento HTML precisa de uma base. Pense nisso como a fundação de uma casa.

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cadastro de Filmes</title>
    <style>
        /* O nosso CSS vai entrar aqui mais tarde! */
    </style>
</head>
<body>
    <!-- O conteúdo visível da nossa página vai aqui -->
</body>
</html>
```

### O que cada linha faz?
* `<!DOCTYPE html>`: Avisa ao navegador que estamos usando a versão mais recente do HTML (HTML5).
* `<html lang="pt-BR">`: A tag principal que abraça todo o site. O `lang` diz ao navegador e aos leitores de tela que o idioma é o Português do Brasil.
* `<head>`: A "cabeça" do site. Aqui ficam as configurações que o usuário *não vê* diretamente na página.
* `<meta charset="UTF-8">`: Permite que o navegador entenda acentuações e caracteres especiais (como ç, ã, é).
* `<meta name="viewport"...>`: Essencial para a responsividade.
* `<title>`: O nome que aparece na aba do navegador.
* `<style>`: Como estamos em uma aula de revisão contínua, vamos colocar nosso CSS aqui dentro. Em projetos reais, usaríamos um arquivo `.css` separado.
* `<body>`: O "corpo". Tudo que o usuário realmente *vê* na tela vai aqui.

---

## Etapa 2: Estruturando o Menu Superior (HTML)

Vamos criar o nosso menu de navegação dentro da tag `<body>`.

```html
<body>
    <header>
        <nav>
            <ul class="menu-principal">
                <li><a href="#">Início</a></li>
                <li><a href="#">Catálogo</a></li>
                <li><a href="#">Cadastrar Filme</a></li>
                <li><a href="#">Contato</a></li>
            </ul>
        </nav>
    </header>
    
    <!-- O formulário virá aqui depois -->
</body>
```

### Explicando as tags:
* `<header>`: Representa o cabeçalho da página (tag semântica).
* `<nav>`: Indica uma seção de navegação. 
* `<ul>`: *Unordered List* (Lista não ordenada).
* `<li>`: *List Item* (Item da lista).
* `<a>`: *Anchor* (Âncora). É o link propriamente dito. O atributo `href="#"` indica um link sem destino no momento.

---

## Etapa 3: Estilizando o Menu (CSS)

Adicione este código dentro da tag `<style>` (no `<head>`):

```css
/* 1. Resetando o padrão do navegador */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: Arial, sans-serif;
    background-color: #f4f4f9;
    color: #333;
}

/* 2. Estilizando o Cabeçalho e o Menu */
header {
    background-color: #2c3e50;
    padding: 15px 0;
}

.menu-principal {
    list-style-type: none;
    display: flex; /* Ativa o Flexbox */
    justify-content: center; /* Centraliza horizontalmente */
    gap: 30px; /* Espaço entre os itens */
}

.menu-principal a {
    text-decoration: none;
    color: white;
    font-weight: bold;
    font-size: 16px;
    padding: 10px;
    transition: color 0.3s;
}

.menu-principal a:hover {
    color: #e74c3c;
}
```

---

## Etapa 4: Iniciando o Formulário de Cadastro (HTML)

Logo abaixo da tag `</header>`, adicione o corpo principal:

```html
    <main>
        <section class="container-form">
            <h2>Cadastro de Novo Filme</h2>
            <p>Preencha os dados abaixo com o máximo de detalhes.</p>

            <form action="#" method="POST">
                <!-- Nossos campos de input entrarão aqui -->
            </form>
        </section>
    </main>
```

---

## Etapa 5: Explorando Todos os Tipos de Inputs

Adicione os campos dentro da tag `<form>`.

### 5.1 Textos e Senhas
```html
<div class="grupo-input">
    <label for="titulo">Título do Filme:</label>
    <input type="text" id="titulo" name="titulo" placeholder="Ex: Matrix" required>
</div>

<div class="grupo-input">
    <label for="senha-admin">Senha do Administrador:</label>
    <input type="password" id="senha-admin" name="senha-admin" required>
</div>
```

### 5.2 Números, Datas e Horas
```html
<div class="grupo-input linha-dupla">
    <div>
        <label for="ano">Ano de Lançamento:</label>
        <input type="number" id="ano" name="ano" min="1888" max="2030">
    </div>
    <div>
        <label for="data-estreia">Data de Estreia:</label>
        <input type="date" id="data-estreia" name="data-estreia">
    </div>
</div>
```

### 5.3 Contatos e Web
```html
<div class="grupo-input">
    <label for="email-diretor">Email do Diretor:</label>
    <input type="email" id="email-diretor" name="email-diretor">
</div>

<div class="grupo-input">
    <label for="trailer">Link do Trailer:</label>
    <input type="url" id="trailer" name="trailer">
</div>
```

### 5.4 Interativos (Radio, Checkbox e Range)
```html
<div class="grupo-input">
    <p class="label-falso">Gêneros:</p>
    <div class="opcoes-inline">
        <input type="checkbox" id="gen-acao" name="generos" value="Acao">
        <label for="gen-acao">Ação</label>

        <input type="checkbox" id="gen-drama" name="generos" value="Drama">
        <label for="gen-drama">Drama</label>
    </div>
</div>

<div class="grupo-input">
    <label for="nota">Nota de Expectativa (0 a 10):</label>
    <input type="range" id="nota" name="nota" min="0" max="10" step="1">
</div>
```

### 5.5 Arquivos, Cores e Botões
```html
<div class="grupo-input linha-dupla">
    <div>
        <label for="poster">Pôster:</label>
        <input type="file" id="poster" name="poster" accept="image/*">
    </div>
    <div>
        <label for="cor-tema">Cor Tema:</label>
        <input type="color" id="cor-tema" name="cor-tema" value="#ff0000">
    </div>
</div>

<div class="grupo-botoes">
    <input type="reset" value="Limpar Tudo" class="btn-limpar">
    <input type="submit" value="Cadastrar Filme" class="btn-enviar">
</div>
```

---

## Etapa 6: Estilizando o Formulário (CSS)

Adicione estas regras ao `<style>`:

```css
.container-form {
    max-width: 600px;
    margin: 40px auto;
    background-color: white;
    padding: 30px;
    border-radius: 8px;
    box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
}

.grupo-input { margin-bottom: 20px; }

.grupo-input label, .label-falso {
    display: block;
    margin-bottom: 8px;
    font-weight: bold;
}

input[type="text"], input[type="password"], input[type="number"],
input[type="date"], input[type="email"], input[type="url"] {
    width: 100%;
    padding: 12px;
    border: 1px solid #ccc;
    border-radius: 5px;
}

.linha-dupla { display: flex; gap: 15px; }
.linha-dupla div { flex: 1; }

.grupo-botoes {
    display: flex;
    justify-content: space-between;
    margin-top: 30px;
}

input[type="submit"], input[type="reset"] {
    padding: 12px 25px;
    border: none;
    border-radius: 5px;
    font-weight: bold;
    cursor: pointer;
    color: white;
}

.btn-enviar { background-color: #27ae60; }
.btn-limpar { background-color: #e74c3c; }
```

## Conclusão da Etapa
Ao juntar as peças, você terá um layout centralizado e um formulário perfeitamente formatado. Treine alterando as cores e margens para fixar o aprendizado!
