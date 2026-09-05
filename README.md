# ci

Wiederverwendbare GitHub-Actions-Workflows für die Schulungs-Repositories der
Organisation `it-erben`.

## Verwendung

Aufruf mit exaktem Tag, damit Renovate die Version anheben kann:

```yaml
jobs:
  lint:
    uses: it-erben/ci/.github/workflows/lint.yml@1.0.0
```

`release.yml` und `pages.yml` erhalten ihre Rechte vom Aufrufer. Ein
gerufener Workflow kann sich selbst keine höheren Rechte geben, der
`permissions`-Block gehört deshalb in den aufrufenden Job.

## Workflows

### `lint.yml`

markdownlint-cli2, yamllint und lychee, je ein Job.

| Eingabe           | Vorgabe | Bedeutung                            |
| ----------------- | ------- | ------------------------------------ |
| `markdown`        | `true`  | markdownlint-cli2 ausführen          |
| `yaml`            | `true`  | yamllint ausführen                   |
| `yamllint-config` | s. u.   | Konfiguration als Inline-YAML        |
| `yamllint-path`   | `.`     | geprüfter Pfad                       |
| `lychee`          | `true`  | Links in Markdown-Dateien prüfen     |

Vorgabe für `yamllint-config`:

```text
{extends: relaxed, ignore: '*.md', rules: {line-length: {max: 140}}}
```

markdownlint liest `.markdownlint.json`, `.markdownlint.yaml` oder
`.markdownlint-cli2.yaml` aus dem aufrufenden Repository. lychee liest
`.lycheeignore`.

### `slides.yml`

Baut je Unterverzeichnis von `source-dir` mit `slides.md` ein PDF über
marp-cli, kopiert bereits vorhandene PDFs daneben und erzeugt aus
`slides/template.html` eine `index.html`. Das Ergebnis liegt unter `public/`
und wird als Artefakt `folien` hochgeladen, mit `pages: true` zusätzlich als
Pages-Artefakt.

| Eingabe        | Vorgabe  |
| -------------- | -------- |
| `source-dir`   | `slides` |
| `marp-version` | `4.2.3`  |
| `pages`        | `true`   |

Fehlt `source-dir`, endet der Job ohne Arbeit.

### `maven.yml`

| Eingabe        | Vorgabe |
| -------------- | ------- |
| `java-version` | `25`    |
| `maven-args`   | s. u.   |

Vorgabe für `maven-args`:

```text
--batch-mode --errors --fail-at-end --show-version --no-transfer-progress verify
```

Docker steht auf dem Runner nativ zur Verfügung, Testcontainers laufen ohne
weitere Einrichtung.

### `release.yml`

semantic-release mit dem Preset `conventionalcommits`, Tag-Format
`${version}`, Release-Branches `main` und `master`. Ein `docs`-Commit ergibt
ein Patch-Release. Bei `pull_request` läuft `--dry-run`.

Ausgaben: `published` ist `true`, wenn ein Release entstanden ist, `version`
trägt dessen Versionsnummer. Die Authentifizierung läuft über das eingebaute
`GITHUB_TOKEN`.

Der Aufrufer erteilt `contents: write`, `issues: write` und
`pull-requests: write`.

### `pages.yml`

Veröffentlicht das Pages-Artefakt aus `slides.yml` über
`actions/deploy-pages`. Setzt voraus, dass GitHub Pages im aufrufenden
Repository auf `build_type: workflow` steht.

Der Aufrufer erteilt `pages: write` und `id-token: write`.

## Zusammenspiel

```yaml
jobs:
  lint:
    uses: it-erben/ci/.github/workflows/lint.yml@1.0.0
  slides:
    uses: it-erben/ci/.github/workflows/slides.yml@1.0.0
  release:
    needs: [lint, slides]
    permissions:
      contents: write
      issues: write
      pull-requests: write
    uses: it-erben/ci/.github/workflows/release.yml@1.0.0
  pages:
    needs: [slides, release]
    if: needs.release.outputs.published == 'true'
    permissions:
      pages: write
      id-token: write
    uses: it-erben/ci/.github/workflows/pages.yml@1.0.0
```
