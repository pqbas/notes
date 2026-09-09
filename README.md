# notes

Apuntes personales publicados como digital garden con [Quartz v5](https://quartz.jzhao.xyz).

- Las notas viven en `content/`, una carpeta por tema y un `.md` por entrada.
- Cada `.md` se convierte en una página del sitio, con wikilinks `[[...]]`,
  backlinks, grafo y buscador.
- El sitio se publica en <https://pqbas.github.io/notes> con la GitHub Action
  de `.github/workflows/deploy.yml`, que corre en cada push a `main`.

## Uso local

```bash
npm ci
npx quartz build --serve   # http://localhost:8080
```

## Agregar una nota

Crear el `.md` dentro de `content/<tema>/` con frontmatter:

```markdown
---
title: Título de la nota
tags:
  - tema
---
```
