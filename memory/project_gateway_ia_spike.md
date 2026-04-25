---
name: AI Gateway Spike — LiteLLM + Solo.io
description: Spike tecnico de evaluacion de AI Gateway para NTT DATA PDI. Implementacion ejecutable con LiteLLM Proxy (Docker Compose, open source MIT) + configuracion de referencia Solo.io Enterprise kgateway (requiere licencia). Directorio C:\NTTDATA\PDI\gateway-ia.
type: project
originSessionId: ef5fb078-0544-4259-b62a-08c7e8a623ae
---

Spike de evaluacion de AI Gateway para el equipo PDI de NTT DATA (Abril 2026).

**Why:** Evaluar viabilidad tecnica como capa de gobernanza, seguridad y enrutamiento centralizado para workloads LLM. No se obtuvo licencia Enterprise de Solo.io, por lo que la implementacion ejecutable usa LiteLLM Proxy (open source MIT) con Docker Compose.

**How to apply:** Al retomar trabajo, el stack esta listo para arrancar con `docker compose up -d` desde C:\NTTDATA\PDI\gateway-ia. Requiere un archivo .env con al menos una API key (Groq es gratuita).

## Estructura del proyecto

```
C:\NTTDATA\PDI\gateway-ia\
├── docker-compose.yml          # Stack completo LiteLLM
├── prometheus.yml              # Scraping de metricas
├── litellm/
│   ├── config.yaml             # Configuracion del gateway (5 proveedores)
│   └── .env.example            # Plantilla de variables de entorno
├── config/                     # YAMLs Solo.io (referencia, requiere licencia)
│   └── 00-namespace.yaml ... 12-failover.yaml
└── docs/
    ├── guia-implementacion.docx  # Guia tecnica v2.0 (LiteLLM + Solo.io)
    ├── capacidades-gateway.docx
    └── socializacion-spike.pptx
```

## Stack Docker Compose (LiteLLM)

Servicios en docker-compose.yml:
- `ai-gateway-litellm` — puerto 4000, imagen ghcr.io/berriai/litellm:main-latest
  - Comando correcto: `["--config", "/app/config.yaml", "--port", "4000", "--num_workers", "2"]`
  - NOTA: el entrypoint de la imagen ya es `litellm`, NO incluir "litellm" en command
- `ai-gateway-postgres` — puerto 5432, virtual keys y spend tracking
- `ai-gateway-redis` — puerto 6379, cache semantico y rate limiting
- `ai-gateway-prometheus` — puerto 9090, metricas (sin depends_on litellm)
- `ai-gateway-presidio-analyzer` — puerto 5002, perfil "pii" OPCIONAL
- `ai-gateway-presidio-anonymizer` — puerto 5001, perfil "pii" OPCIONAL

Comandos de operacion:
```bash
# Stack principal
docker compose up -d
# Stack + Presidio PII (opcional, imagenes grandes ~500MB)
docker compose --profile pii up -d
# Ver logs
docker compose logs litellm --tail 50
# Teardown completo
docker compose down -v
```

## Cadena de failover configurada

Todos los proveedores usan el alias "gpt-4o". El campo `order` determina preferencia:
1. OpenAI gpt-4o (order:1) — de pago
2. Anthropic claude-3-5-sonnet-20241022 (order:2) — de pago
3. Groq llama-3.3-70b-versatile (order:3) — GRATUITO, 30 RPM, console.groq.com
4. Gemini 2.0 Flash (order:4) — GRATUITO, 15 RPM, ai.google.dev
5. Ollama llama3.2 (order:5) — LOCAL, sin internet, host.docker.internal:11434

Acceso directo por nombre: claude-3-5-sonnet, llama-3.3-70b, gemini-flash, llama-local

## Variables de entorno (.env)

Copiar litellm/.env.example → .env y rellenar:
- OPENAI_API_KEY, ANTHROPIC_API_KEY — de pago (opcionales si se usa Groq)
- GROQ_API_KEY — gratuita en console.groq.com (sin tarjeta)
- GEMINI_API_KEY — gratuita en ai.google.dev
- LITELLM_MASTER_KEY=sk-spike-nttdata-2026 (fijo, para admin)
- LITELLM_SALT_KEY=00000000000000000000000000000032 (fijo)
- DATABASE_URL, REDIS_HOST/PORT/PASSWORD, PRESIDIO_* (valores fijos de docker-compose)

## Capacidades configuradas

1. **Enrutamiento multi-proveedor** — 5 proveedores bajo alias gpt-4o con failover por order
2. **Failover automatico** — router_settings.fallbacks, allowed_fails:1, num_retries:2
3. **Virtual keys + rate limiting** — POST /key/generate con tpm_limit/rpm_limit, almacenado en Postgres
4. **Cache semantico** — Redis, embedding text-embedding-3-small, threshold 0.92, TTL 3600s
5. **Prompt Guard PII** — Presidio (perfil pii), guardrail presidio-pii-masking, default_on:false
   - Para activar: pasar `"guardrails": ["presidio-pii-masking"]` en el body de la peticion
6. **Observabilidad** — Prometheus callbacks, metricas en localhost:9090 y localhost:4000/metrics

## Autenticacion

- Master key (admin): `sk-spike-nttdata-2026`
- Virtual keys: crear con POST /key/generate usando la master key
- UI: http://localhost:4000/ui

## Errores conocidos y soluciones aplicadas

1. Presidio latest tag no disponible en MCR → imágenes completamente removidas de MCR, no hay tag disponible
2. litellm command duplicado ("litellm litellm ...") → el entrypoint de la imagen ya es litellm, solo pasar args
3. start_period muy corto (20s) → aumentado a 60s para dar tiempo a conexion con Postgres
4. Prometheus depends_on litellm → eliminado para evitar fallo en cascada
5. Guardrails rotos en litellm:main-latest (v1.82.6+) → `guardrails` dentro de `litellm_settings` usa API v1 (rota). Usar SIEMPRE formato lista al nivel raíz del YAML (API v2). Ver config.yaml sección "guardrails:"
6. Presidio reemplazado por `hide-secrets` (built-in LiteLLM, sin Docker adicional) — detecta API keys y tokens
7. Healthcheck: imagen nueva no tiene `curl` → usar `python3 -c "import urllib.request; urllib.request.urlopen('http://localhost:4000/health/liveliness')"`

## Documentos generados

- `docs/guia-implementacion.docx` v2.0 — Parte I LiteLLM (instalacion paso a paso, pruebas curl) + Parte II Solo.io (referencia)
- `docs/capacidades-gateway.docx` — Referencia de capacidades y comparativas
- `docs/socializacion-spike.pptx` — 16 diapositivas en colores NTT DATA (#0067B1)

## Solo.io Enterprise kgateway (referencia)

Los archivos config/00-*.yaml a config/12-*.yaml son la configuracion de referencia.
Requiere: Kubernetes 1.28+, Helm 3.12+, licencia Enterprise Solo.io (trial: solo.io/free-trial)
Instalacion: helm upgrade -i enterprise-kgateway ... --version 2.2.0
