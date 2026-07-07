`https://miscape.github.io/ops-toolbox/`.

Da ora il flusso è questo:

```bash
cd ops-toolbox-docs
source .venv/bin/activate
mkdocs serve
```

modifichi i file Markdown dentro `docs/`, controlli in locale su:

```text
http://127.0.0.1:8000
```

poi quando vuoi pubblicare:

```bash
mkdocs build --strict
git status
git add .
git commit -m "Update docs"
git push
```

GitHub Actions ricostruisce il sito e GitHub Pages aggiorna automaticamente l’URL pubblico.

Piccola nota: assicurati che `site/` e `.venv/` siano nel `.gitignore`, così su GitHub mandi solo i sorgenti del sito, non i file generati.
