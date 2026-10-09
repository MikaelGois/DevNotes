# AGENTS.md

Este arquivo orienta agentes automatizados (como o OpenCode) que atuam neste repositório. O DevNotes é um site de artigos técnicos construído com **Hugo** + tema **Hextra**, com conteúdo em **português (pt-BR)** como idioma padrão e **inglês (en-US)** como tradução. O tema é genérico — não é dedicado a uma única área (Hadoop, redes, etc.); empreenda **notas técnicas de qualquer área**.

## Stack

- **Gerador estático:** Hugo (`hugo.yaml` na raiz; `defaultContentLanguage: pt`)
- **Tema:** Hextra (importado como módulo Go em `module.imports.path: github.com/imfing/hextra`)
- **Markdown:** GitHub Flavored Markdown via Goldmark (`markup.goldmark.renderer.unsafe: true`)
- **Shortcodes inline:** habilitados (`enableInlineShortcodes: true`) — logo, shortcodes como `{{% details ... %}}` e `{{< cards >}}` funcionam em qualquer artigo.
- **Conteúdo:** organizado em `content/articles/<ANO>/<MES>/`. Idioma padrão em `content/articles/...`; a versão inglesa fica em arquivos `.en.md` irmãos.

## Comandos úteis

```shell
# Servir localmente (com reload)
hugo server --logLevel info --disableFastRender -p 1313

# Build de produção (com GC e minify)
hugo --gc --minify --logLevel info

# Atualizar o tema Hextra
hugo mod get -u && hugo mod tidy
```

Antes de considerar o trabalho concluído, sempre rode `hugo --gc --minify` e confirme **0 erros**, além do número esperado de páginas PT/EN.

## Convenções dos artigos

### Estrutura de diretórios

```
content/
└── articles/
    ├── _index.md            # Lista geral de artigos por ano e mês
    ├── _index.en.md
    ├── <ANO>/
    │   ├── _index.md        # Lista de artigos do ano por mês
    │   ├── _index.en.md
    │   ├── <MES>/           # Janeiro=01, ..., Dezembro=12
    │   │   ├── _index.md    # Lista de artigos do mês (cards)
    │   │   ├── _index.en.md
    │   │   ├── 1-<slug>.md    # Artigo em PT
    │   │   ├── 1-<slug>.en.md  # Tradução EN (opcional)
    │   │   ├── 2-<slug>.md
    │   │   └── image-*.png      # Imagens usadas no(s) artigo(s)
    │   └── ...
    └── ...
```

### Frontmatter dos `_index.md`

#### `_index.md` do mês

```yaml
---
title: <Nome do mês por extenso>     # "Julho" / "July"
type: docs
weight: <peso>                       # Mês atual = 1, meses passados crescentes (2, 3, ...)
prev: /articles/<ANO>/<MM-MAIS-NOVO>/ # Aponta para o mês seguinte (mais novo) cronologicamente
next: /articles/<ANO>/<MM-MAIS-VELHO>/ # Aponta para o mês anterior (mais velho)
sidebar:
  open: true
---
```

- **Pesos:** sem valores negativos. Mês mais recente do ano = peso **1**; meses anteriores crescem a partir de 2. Na sidebar do Hextra (ordenada por `weight`), o menor peso fica embaixo — portanto o mês mais novo aparece no topo da ordem cronológica quando combinado com os `prev`/`next`. Sempre siga o padrão do mês anterior.
- `prev`/`next` seguem a ordem cronológica encadeada (Julho aponta para Agosto em `prev`, e para Junho em `next`). Janeiro de um ano novo aponta para Dezembro do ano anterior em `next`.
- **`sidebar.open: true`** só para o `_index.md` do **mês em andamento** do ano corrente (ex.: `2026/07/_index.md` em julho/2026). Todos os meses passados do ano corrente e todos os meses de anos passados **não** devem ter `sidebar.open` — a seção começa recolhida na sidebar.

#### `_index.md` do ano

```yaml
---
title: <ANO>
type: docs
weight: <peso>             # Ano atual = 1; anos passados crescem (2026=1, 2025=2, ...)
# prev: /articles/<ANO-MAIS-NOVO>/   # Comentado quando não há ano mais novo ainda
next: /articles/<ANO-MAIS-VELHO>/
sidebar:
  open: true               # Para o ano atual; omitir em anos passados
---
```

- **`sidebar.open: true`** só para o `_index.md` do **ano corrente** (ex.: `2026/_index.md` em 2026). Anos passados **não** devem ter `sidebar.open` — a seção começa recolhida na sidebar.

### Estilo editorial

- Use **callouts GitHub**: `> [!NOTE]`, `> [!WARNING]`, `> [!IMPORTANT]`, `> [!TIP]`, `> [!CAUTION]`.
- Use o **shortcode Hextra** `{{% details title="..." closed="true" %}}...{{% /details %}}` para tabelas de explicação de código.
- Use blocos de código com atributo `{filename="..."}` para indicar o arquivo de cada snippet:
  ```` ```yaml {filename="provisionar_no_hadoop.yaml"} ````
- Frases-padrão: "Pressione `Ctrl + S` para salvar, `Ctrl + X` para sair." (em nano/vim).
- Não adicione comentários em código (`# ...`) a menos que o conteúdo do artigo explique cada linha.
- **Regras de capitalização em português:** meses do ano começam em **minúscula** no meio da frase ("Não houve publicações em **fevereiro**.", "A primeira metade do ano começa em abril."). Só começam em **maiúscula** no início de frase, em títulos (ex.: `### Julho:` em `_index` do ano; `title: Julho` no frontmatter) ou quando fazem parte de um nome próprio. **Atenção:** isto vale só para PT; em inglês os meses são **sempre maiúsculos** ("No articles were published in February.").

### Cards de artigos publicados

O uso do envelope `{{< cards cols="1" >}}...{{< /cards >}}` depende do nível do `_index`:

| Local | Quando usar `{{< cards >}}` |
| ----- | -------------------------- |
| `_index.md` do **mês** | Só quando há **artigos publicados**. Para meses sem publicações, deixe a frase sozinha (sem envelope) — o envelope gera um "botão" visual que não leva a lugar nenhum. |
| `_index.md` do **ano** e **lista geral** (`content/articles/_index.md`) | **Sempre** envolva cada seção mensal em `{{< cards >}}`, mesmo quando não há artigos — para manter consistência visual das seções. |

#### Com card de artigo publicado

```markdown
## Artigos publicados:

{{< cards cols="1" >}}
  {{< card link="1-<slug>" title="Título do artigo" tag="<Categoria>" tagColor="<cor>" >}}
{{< /cards >}}
```

- Em **`_index.md` do mês**, o `link` aponta para o slug relativo (ex.: `1-hadoop-cluster`).
- Em **`_index.md` do ano** e na **lista geral** (`content/articles/_index.md`), o `link` aponta para o caminho completo relativo ao ano (ex.: `07/1-hadoop-cluster`).
- Na versão **inglesa** (`.en.md`), o `link` aponta para a URL pública de produção quando o artigo não tem versão em inglês (ex.: `https://devnotes.msglabs.com.br/articles/2025/07/1-hadoop-cluster/`), com sufixo "(portuguese only)" no título.

#### Sem artigo publicado

**`_index.md` do mês** (frase sozinha, sem `cards`):

```markdown
## Artigos publicados:

Não houve publicações em fevereiro.
```

**`_index.md` do ano** e **lista geral** (frase dentro de `cards`):

```markdown
### Fevereiro:

{{< cards cols="1" >}}
  Não houve publicações em fevereiro.
{{< /cards >}}
```

### Cores das tags (categorias)

O `tagColor` é **customizado** — o `card` nativo do Hextra não suporta `tagColor`, somente `tagType` (`info`/`warning`/`error`). Para habilitar cores customizadas no badge, há **três overrides** em `layouts/`:

- `layouts/shortcodes/card.html` — lê `tagColor` do shortcode
- `layouts/partials/shortcodes/card.html` — repassa `tagColor` ao partial do badge
- `layouts/partials/shortcodes/badge.html` — mapa que aplica classes Tailwind (cores habilitadas no `tailwind.config.js` do Hextra: **red**, **orange**, **amber**, **yellow**, **green**, **blue**, **indigo**)

Cores já estabelecidas para categorias do DevNotes:

| Categoria (PT/EN) | `tagColor` | `tag` (PT) | `tag` (EN) |
| ------------------ | ---------- | ---------- | ---------- |
| Arquitetura / Architecture | `yellow` | `Arquitetura` | `Architecture` |
| Redes / Network            | `red`    | `Redes`       | `Network` |
| Automação / Automation     | `blue`   | `Automação`   | `Automation` |

Ao criar uma nova categoria, escolha uma cor ainda não usada e — se a cor não estiver habilitada no `tailwind.config.js` do Hextra — adicione-a ao mapa em `layouts/partials/shortcodes/badge.html`.

### Meses sem publicações

Para meses sem artigos publicados, use a frase abaixo. No **`_index.md` do mês**, a frase aparece **sozinha** (sem envelope `{{< cards >}}`); no **`_index.md` do ano** e na **lista geral**, a frase aparece **dentro de** `{{< cards cols="1" >}}...{{< /cards >}}` (conforme tabela da seção anterior).

- PT: `Não houve publicações em <mês>.` (ou `Não houve publicações até o momento.` para o mês em andamento). Ex.: "Não houve publicações em maio."
- EN: `No articles were published in <Month>.` (ou `There have been no publications so far.` para o mês em andamento). Ex.: "No articles were published in May."

## Convenções técnicas gerais

- **Extensões de arquivo:** sempre `.yaml` para arquivos de configuração e playbooks. O artigo pratica o uso de `.yaml` (não `.yml`), inclusive nos blocos ``` ```yaml {filename="..."} ```.
- **Configuração de laboratório:** quando um artigo documentar um projeto real do autor, mantenha os IPs/paths/hostnames **consistentes com o referencial do repositório** — não invente outros números na mesma série.
- **Referências:** ao criar um novo artigo que é continuação de uma série, mencione o(s) anterior(es) em uma nota no início (ex.: "Este é o segundo artigo de uma série de X artigos") e referencie-os por links internos.

## Frontmatter dos artigos

- `title`, `type: docs`, `weight: 1` (default dentro do mês), `editURL` (quando aplicável), `next` (para o próximo artigo da série).
- Não inclua `date` manualmente — `enableGitInfo: true` lê `Lastmod` do Git automaticamente.

## Validação

Antes de publicar um artigo novo ou retrabalhado:

1. **YAML válido:** extraia todos os blocos ``` ```yaml {filename="..."} ``` ` e valide-os com `python -c "import yaml,sys; list(yaml.safe_load_all(sys.stdin))"`. Quando um playbook é apresentado em vários trechos, concatene todos os trechos do mesmo `filename` antes de validar.
2. **Build do Hugo:** `hugo --gc --minify` deve terminar com `0 errors` e o número esperado de páginas PT/EN.
3. **Anchors:** confirme pelo `grep` no HTML renderizado (`public/articles/.../index.html`) que os links internos `#...` existem como `id=` em algum heading (note que o Hextra pode renderizar o `id` dentro de elementos `<a>` wrapper, e caracteres acentuados são codificados como `%XX`).

## Permissões de edição

O `opencode.json` na raiz autoriza `edit` apenas para arquivos `*.md` no workspace, usando a chave `permission` (singular) com sintaxe de objeto: `"edit": { "*.md": "allow" }`. Para editar outros tipos de arquivo (YAML, configs), use o `bash` (PowerShell) como workaround ou solicite ao usuário que amplie o pattern.

## Referências do projeto

- `content/articles/2025/07/_index.md` — referência de estilo para cards de artigos publicados e estrutura do mês.
- `content/articles/2025/_index.md` — referência para `_index.md` do ano (cards, seções mensais).
- `content/articles/_index.md` — referência para a lista de artigos geral.
- `content/articles/2026/07/_index.md` — referência de estilo para o mês corrente (categoria Automação, `sidebar.open: true`).
- `content/articles/2026/07/1-ansible.md` — exemplo de artigo da categoria Automação (português-only, com `editURL` e `next` encadeado).
- `hugo.yaml` — configurações do site.
- `opencode.json` — regras de permissão do agente.
