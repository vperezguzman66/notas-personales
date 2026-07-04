---
proyecto: "ProvDent-limpio"
ruta: "ProvDent-limpio"
cliente: "Propio"
stack: "Python + FastAPI + PostgreSQL + Docker"
estado: "Pausado — reinicio limpio, solo health check"
---

[[Varios]]

Reinicio limpio de [[ProvDent]]: solo `models.py` (`Paciente` simplificado, sin `apellido`), sin `schemas.py` ni `db.py` sueltos, sin auth, sin migraciones. Solo expone `GET /`. Punto de partida si en algún momento se quiere refactorizar ProvDent desde cero.
