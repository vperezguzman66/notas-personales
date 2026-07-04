---
proyecto: "ProvDent"
ruta: "ProvDent"
cliente: "Propio"
stack: "Python + FastAPI + PostgreSQL + Docker"
estado: "En desarrollo — CRUD pacientes + JWT implementado"
---

[[Varios]]

Backend dental (pacientes, autenticación JWT). La versión más completa de las 3 variantes del proyecto — usar esta para continuar desarrollo. Tiene `db.py`, `schemas.py`, `models.py` (`Paciente` con nombre+apellido+email, `User` con hash bcrypt), `test_endpoints.py`, 2 migraciones Alembic, Docker + docker-compose. Endpoints: CRUD completo de pacientes + `/register` `/token` `/users/me` (JWT).

Ver también [[ProvDent-limpio]] (reinicio limpio) y [[ProvDent-limpio-historial]] (snapshot de referencia).
