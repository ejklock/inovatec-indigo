---
description: "Use when working on the tutor-indigo Open edX theme plugin: editing plugin.py, SCSS files, Mako templates, Jinja2 config variables, or any file under tutorindigo/. Covers plugin hooks, SCSS architecture, template conventions, dark theme, config naming, Python linting, and build workflow."
applyTo: "tutorindigo/**"
---

# tutor-indigo — Convenções e Padrões do Projeto

## Stack e Versões

- **Python**: 3.9+ (suporta 3.9, 3.10, 3.11, 3.12)
- **Tutor**: `>=21.0.0, <22.0.0` — nunca use APIs de versões fora desse range
- **tutor-mfe**: `>=21.0.0, <22.0.0`
- **Linter**: `ruff` (seleções: E, I, N, F401, F841, W292)
- **Build**: `hatch` com `hatchling` como backend
- **Versão do plugin**: definida em `tutorindigo/__about__.py`

## Arquitetura do Plugin

### Estrutura de diretórios

```
tutorindigo/
  __about__.py       # versão do plugin
  plugin.py          # ponto de entrada: hooks Tutor, config, patches MFE
  templates/
    indigo/
      lms/           # tema do LMS (alunos)
        static/sass/ # estilos SCSS organizados por componente
        templates/   # templates Mako (.html)
      cms/           # tema do Studio (autores)
        static/sass/
      tasks/
        init.sh      # script de inicialização (settheme)
```

### Como o plugin funciona

O plugin usa o sistema de hooks do Tutor v1 (`tutor.plugin.v1`). Os principais filtros usados são:

```python
hooks.Filters.ENV_TEMPLATE_ROOTS       # registra raiz dos templates
hooks.Filters.ENV_TEMPLATE_TARGETS     # define onde renderizar (build/openedx/themes)
hooks.Filters.ENV_PATTERNS_INCLUDE     # força renderização de _partials SCSS
hooks.Filters.CLI_DO_INIT_TASKS        # executa init.sh ao inicializar
hooks.Filters.CONFIG_DEFAULTS          # adiciona variáveis INDIGO_* com padrões
hooks.Filters.ENV_PATCHES              # injeta patches nos MFEs
```

## Configuração e Variáveis

### Nomenclatura obrigatória

Todas as configurações do plugin usam o prefixo `INDIGO_`:

```python
config["defaults"] = {
    "PRIMARY_COLOR": "#15376D",        # vira INDIGO_PRIMARY_COLOR no Tutor
    "WELCOME_MESSAGE": "...",
    "ENABLE_DARK_TOGGLE": True,
    "FOOTER_NAV_LINKS": [...],
}
```

- Nunca adicione configurações sem o prefixo `INDIGO_`
- Valores `defaults` são configuráveis pelo usuário via `tutor config save --set INDIGO_X=Y`

### Interpolação de variáveis em templates

Use sintaxe **Jinja2** para variáveis do Tutor dentro dos arquivos de template:

```scss
// _variables.scss
$primary: {{ INDIGO_PRIMARY_COLOR }};
```

```html
<!-- footer.html (Mako + Jinja2 misturados) -->
{% if INDIGO_FOOTER_NAV_LINKS %} {% for link in INDIGO_FOOTER_NAV_LINKS %}
<a href="{{ link['url'] }}">{{ link['title'] }}</a>
{% endfor %} {% endif %}
```

> Templates Mako usam `${}`, `%if`, `<%block>`, etc. Variáveis do Tutor usam `{{ }}` e `{% %}`.

## SCSS — Arquitetura e Convenções

### Estrutura de partials (LMS)

```
sass/partials/lms/theme/
  _variables.scss   # variáveis SCSS (usa {{ INDIGO_* }})
  _extras.scss      # regras principais do tema
  _fonts.scss       # declarações @font-face
```

### Variáveis SCSS principais (não remover/renomear)

| Variável             | Uso                          |
| -------------------- | ---------------------------- |
| `$primary`           | Cor principal (azul indigo)  |
| `$primary-light`     | Fundo claro, hover, badge    |
| `$primary-d`         | Cor principal no dark mode   |
| `$primary-light-d`   | Fundo escuro no dark mode    |
| `$dark`              | Texto escuro `#111827`       |
| `$light-dark`        | Texto secundário `#374151`   |
| `$body-bg-d`         | Fundo do dark mode `#0D0D0E` |
| `$text-color-d`      | Texto claro para dark mode   |
| `$font-family-title` | Inter (fonte principal)      |

### Dark theme

O dark mode é ativado via a classe `body.indigo-dark-theme`. **Todo bloco que altera estilos para o dark mode deve ser aninhado dentro desse seletor:**

```scss
.meu-componente {
  color: $dark;
  background: $primary-light;
}

body.indigo-dark-theme {
  .meu-componente {
    color: $text-color-d;
    background: $primary-light-d;
  }
}
```

- Nunca use `!important` para sobrescrever no dark mode, a menos que o estilo original já use `!important`
- Toque sempre nos dois contextos (light e dark) ao modificar estilos visuais

### Layout padrão

- **Max-width dos containers**: `1600px`
- **Padding lateral**: `15px` (mobile) via `padding: 0 15px`
- **Altura do header fixo**: `$header-height: 75px`
- **Breakpoints**: use os mixins do Bootstrap (`media-breakpoint-up(lg)`, etc.)

## Templates Mako

### Cabeçalho obrigatório

Todo arquivo de template Mako deve começar com:

```html
## mako <%page expression_filter="h"/>
```

### Comentários para mudanças vs. tema base

Ao modificar templates em relação ao tema base do Open edX, marque as alterações com:

```html
<!-- NEW IN INDIGO: descrição da mudança -->
```

Isso facilita rastrear o que foi customizado em relação ao upstream.

### Importações Django nos templates

```html
<%! from django.utils.translation import gettext as _ from django.urls import
reverse %>
```

## Python — Convenções

### Type hints

Use `typing` para anotações:

```python
import typing as t
config: t.Dict[str, t.Dict[str, t.Any]] = {}
```

### Hooks e prioridades

```python
@hooks.Filters.CONFIG_DEFAULTS.add(priority=hooks.priorities.LOW)
def _minha_funcao(items: list[tuple[str, t.Any]]) -> list[tuple[str, t.Any]]:
    ...
```

### Linting com ruff

O projeto usa `ruff` com as regras E, I, N, F401, F841, W292. O diretório `templates/` está **excluído** da análise:

```toml
# pyproject.toml
[tool.ruff]
exclude = ["templates", "docs/_ext"]
```

- Não deixe imports não utilizados (F401)
- Não deixe variáveis não utilizadas (F841)
- Arquivos devem terminar com newline (W292)
- Nomes devem seguir PEP8 (N)

## MFE Styling (Patches)

MFEs estilizados pelo Indigo:

```python
indigo_styled_mfes = ["learning", "learner-dashboard", "profile", "account", "discussions"]
```

Os patches são injetados via `hooks.Filters.ENV_PATCHES` usando arquivos em `tutorindigo/patches/`.

## Build e Desenvolvimento

### Testar uma mudança localmente

```bash
tutor config save          # re-renderiza templates
tutor images build openedx # rebuild da imagem (produção)
# ou, em modo dev:
tutor dev run lms bash     # não precisa rebuild
```

### Verificar linting antes de commitar

```bash
ruff check tutorindigo/
```

### Release e versionamento

- A versão está em `tutorindigo/__about__.py`
- O CHANGELOG usa `scriv` (ver `changelog.d/`)
- Branches de feature: use prefixo do cliente (ex: `inovatec/nome-da-feature`)
