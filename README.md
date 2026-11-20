# mynor-architecture: skill de arquitectura

Este repo guarda un **skill** para Cursor y Grok Bot con la arquitectura y el orden de carpetas que uso por defecto en proyectos nuevos:

- **Backend:** FastAPI + SQLModel + Alembic, en capas: Routes → Controllers → Services → Database, con SDK clients, workers, middlewares y `common/` para schemas y excepciones.
- **Frontend:** React 19 + Vite + TanStack Router/Query + Zustand + Tailwind/shadcn, organizado en `api/services`, `modules/`, `pages/`, `routes/`, `stores/` y `schemas/`.

Sale de dos proyectos reales: sus docs más una revisión del código. Separa lo que **ya se hace** (*Observed conventions*) de las **mejoras recomendadas** (*Recommended additions*). En un repo que ya tiene sus propias convenciones, mandan las del repo.

```
mynor-architecture/
├── SKILL.md               # resumen: árbol de carpetas, reglas por capa, cómo agregar una feature, checklist
└── references/
    ├── backend.md         # plantillas de código, sesiones, auth, SDK, workers, logging, tests, CI, gaps
    └── frontend.md        # cliente HTTP, endpoints, queries/mutations, rutas, forms, stores, lint, gaps
```

## Instalación

### En Cursor, para todos tus proyectos

```bash
mkdir -p ~/.cursor/skills
cp -R mynor-architecture ~/.cursor/skills/
```

### En un proyecto concreto

Así el skill viaja con el repo:

```bash
mkdir -p .cursor/skills
cp -R /ruta/a/este-repo/mynor-architecture .cursor/skills/
```

### Con git clone, en una línea

```bash
git clone https://github.com/<tu-usuario>/<este-repo>.git /tmp/mynor-arch && mkdir -p ~/.cursor/skills && cp -R /tmp/mynor-arch/mynor-architecture ~/.cursor/skills/ && rm -rf /tmp/mynor-arch
```

Para actualizarlo, vuelve a correr el mismo comando; reemplaza la carpeta.

## Uso

El agente lo carga solo cuando empiezas un proyecto nuevo, estructuras uno o agregas una feature de backend o frontend. También puedes pedirlo directamente: "usa el skill mynor-architecture".

## Nota

No contiene secretos, valores de `.env` ni dominios de clientes; todo está generalizado.
