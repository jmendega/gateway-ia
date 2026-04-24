# Spike: AI Gateway — Evaluación Técnica

**NTT DATA | PDI | Abril 2026**

Evaluación técnica de soluciones de AI Gateway como capa de gobernanza, seguridad y enrutamiento centralizado para workloads LLM en entornos enterprise.

El spike incluye **dos implementaciones**:
- **LiteLLM Proxy** (open source, MIT) — ejecutable con `docker compose up` sin licencias ✅
- **Solo.io Enterprise kgateway** — referencia de configuración Kubernetes (requiere licencia)

---

## Entregables

| Archivo | Descripción |
|---|---|
| `docker-compose.yml` | Stack LiteLLM listo para ejecutar (opción OSS) |
| `litellm/config.yaml` | Configuración completa del AI Gateway |
| `config/` | Manifiestos YAML de Kubernetes — referencia Solo.io |
| `docs/guia-implementacion.docx` | Guía técnica paso a paso |
| `docs/capacidades-gateway.docx` | Referencia de capacidades evaluadas |
| `docs/socializacion-spike.pptx` | Presentación de resultados para el equipo |

---

## Ejecución del Spike con LiteLLM (sin licencia)

### Prerrequisitos
- Docker Desktop instalado y corriendo
- API key de OpenAI: https://platform.openai.com/api-keys
- API key de Anthropic: https://console.anthropic.com/

### Inicio rápido

```bash
# 1. Clonar / ubicarse en el directorio del proyecto
cd C:\NTTDATA\PDI\gateway-ia

# 2. Crear el archivo de variables de entorno
cp litellm/.env.example .env
# Editar .env con las API keys reales (OPENAI_API_KEY y ANTHROPIC_API_KEY)

# 3. Crear el archivo prometheus.yml como archivo (obligatorio)
# Ya existe en el repositorio — no modificar

# 4. Levantar el stack completo
docker compose up -d

# 5. Verificar que todos los servicios estén healthy (~60s)
docker compose ps
```

### Servicios disponibles

| Servicio | URL | Descripción |
|---|---|---|
| LiteLLM API | http://localhost:4000 | AI Gateway (OpenAI-compatible) |
| LiteLLM UI | http://localhost:4000/ui | Panel de administración |
| Prometheus | http://localhost:9090 | Métricas |

### Pruebas de las capacidades

#### 1. Enrutamiento (OpenAI como primario)
```bash
curl -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer sk-spike-nttdata-2026" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hola, quien eres?"}]}'
```

#### 2. Crear virtual key con rate limit por tokens
```bash
curl -X POST http://localhost:4000/key/generate \
  -H "Authorization: Bearer sk-spike-nttdata-2026" \
  -H "Content-Type: application/json" \
  -d '{"models":["gpt-4o","claude-3-5-sonnet"],"tpm_limit":50000,"rpm_limit":100,"metadata":{"user":"equipo-pdi"}}'
# Copiar el campo "key" de la respuesta como USER_KEY
```

#### 3. Rate limiting por tokens
```bash
# Con la virtual key, los límites de tokens se aplican automáticamente
curl -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer $USER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Test rate limit"}]}'
# Headers de respuesta: x-ratelimit-remaining-tokens, x-ratelimit-limit-tokens
```

#### 4. Prompt Guard — enmascarado de PII
```bash
# Esperar ~30s a que Presidio esté healthy antes de esta prueba
curl -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer $USER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Mi tarjeta es 4111-1111-1111-1111 ayudame"}]}'
# El número de tarjeta se enmascara antes de llegar a OpenAI
```

#### 5. Cache semántico
```bash
# Primera llamada — cache MISS
curl -i -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer $USER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Cuantos dias tiene un anio?"}]}'
# Buscar header: x-litellm-cache: MISS

# Segunda llamada semánticamente similar — cache HIT
curl -i -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer $USER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Cuantos dias conforman un anio?"}]}'
# Buscar header: x-litellm-cache: HIT
```

#### 6. Failover automático
```bash
# Simular falla de OpenAI: editar .env con OPENAI_API_KEY=invalid
# Reiniciar solo el servicio LiteLLM:
docker compose restart litellm

# La solicitud debe resolverse con Anthropic
curl -X POST http://localhost:4000/chat/completions \
  -H "Authorization: Bearer $USER_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Test failover"}]}'
# Buscar header: x-litellm-model-id — debe mostrar el modelo de Anthropic
```

#### 7. Métricas Prometheus
```bash
# Métricas en texto plano
curl http://localhost:4000/metrics | grep litellm

# O consultar via Prometheus en http://localhost:9090
# Query útil: litellm_requests_total
```

### Teardown
```bash
docker compose down -v   # -v elimina los volúmenes (BD y cache)
```

---

## Referencia: Solo.io Enterprise kgateway (requiere licencia)

> Los archivos en `config/` son la configuración de referencia para Solo.io Enterprise kgateway.
> Requieren Kubernetes 1.28+, Helm 3.12+ y licencia Enterprise de Solo.io.
> Ver documentación completa en `docs/guia-implementacion.docx`.

---

## Prerrequisitos (Solo.io kgateway)

- Kubernetes 1.28+ (cloud: GKE/AKS/EKS · local: Kind o k3s)
- Helm 3.12+
- kubectl configurado contra el clúster objetivo
- Licencia Enterprise de Solo.io — trial gratuito en https://www.solo.io/free-trial
- API keys: OpenAI y Anthropic

---

## Instalación del Gateway

### 1. Instalar Gateway API CRDs (estándar CNCF)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
```

### 2. Instalar Enterprise kgateway via Helm

```bash
export LICENSE_KEY="<TU_LICENCIA_SOLO_IO>"

helm upgrade -i enterprise-kgateway \
  oci://us-docker.pkg.dev/solo-public/enterprise-kgateway/charts/enterprise-kgateway \
  -n kgateway-system --create-namespace \
  --version 2.2.0 \
  --set-string licensing.licenseKey=$LICENSE_KEY
```

### 3. Verificar instalación

```bash
kubectl get pods -n kgateway-system
kubectl get crd | grep kgateway
```

---

## Aplicar configuración del spike

```bash
# Editar primero los secrets con tus API keys reales
cp config/03-secrets.yaml config/03-secrets.local.yaml
# Editar config/03-secrets.local.yaml con las claves reales

# Aplicar en orden
kubectl apply -f config/00-namespace.yaml
kubectl apply -f config/01-gateway-parameters.yaml
kubectl apply -f config/02-gateway.yaml
kubectl apply -f config/03-secrets.local.yaml     # usar el archivo local, NO commitear
kubectl apply -f config/04-openai-backend.yaml
kubectl apply -f config/05-anthropic-backend.yaml
kubectl apply -f config/06-httproute-llm.yaml
kubectl apply -f config/07-routeoption-ratelimit.yaml
kubectl apply -f config/08-routeoption-prompt-guard.yaml
kubectl apply -f config/09-routeoption-semantic-cache.yaml
kubectl apply -f config/10-routeoption-rag.yaml
kubectl apply -f config/11-observability.yaml
kubectl apply -f config/12-failover.yaml
```

### Instalación local con Kind

```bash
# Crear clúster local
kind create cluster --name ai-gateway-spike

# El resto de pasos son idénticos al despliegue cloud
# Para acceder al gateway desde localhost:
kubectl port-forward svc/ai-gateway 8080:80 -n ai-gateway
```

---

## Pruebas rápidas

```bash
# Obtener la IP del gateway (cloud)
GATEWAY_IP=$(kubectl get gateway ai-gateway -n ai-gateway -o jsonpath='{.status.addresses[0].value}')

# Test básico — OpenAI
curl -s -X POST http://$GATEWAY_IP/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-user-id: usuario-prueba" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Hola, ¿cómo estás?"}]}'

# Test prompt guard — debe retornar 403
curl -s -X POST http://$GATEWAY_IP/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-user-id: usuario-prueba" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"Mi tarjeta de crédito es 4111-1111-1111-1111"}]}'

# Test cache semántico — la segunda llamada debe tener x-semantic-cache: hit
curl -s -X POST http://$GATEWAY_IP/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "x-user-id: usuario-prueba" \
  -d '{"model":"gpt-4o","messages":[{"role":"user","content":"¿Cuántos días tiene un año?"}]}'

# Test métricas Prometheus
kubectl port-forward svc/otel-collector 8889:8889 -n ai-gateway &
curl -s http://localhost:8889/metrics | grep ai_gateway
```

---

## Teardown

```bash
kubectl delete namespace ai-gateway
helm uninstall enterprise-kgateway -n kgateway-system
kubectl delete -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml
# Local Kind:
kind delete cluster --name ai-gateway-spike
```

---

## Versiones evaluadas

| Componente | Versión |
|---|---|
| Enterprise kgateway | 2.2.0 |
| Kubernetes Gateway API | v1.2.0 |
| Envoy Proxy | bundled con kgateway 2.2.0 |
| Kubernetes mínimo | 1.28 |

---

## Proveedores LLM evaluados

| Proveedor | Modelo | Rol en el spike |
|---|---|---|
| OpenAI | gpt-4o | Proveedor primario |
| Anthropic | claude-3-5-sonnet-20241022 | Failover / alternativo |

> Otros proveedores soportados (Azure OpenAI, Google Gemini, Amazon Bedrock, Mistral) están documentados en `docs/capacidades-gateway.docx` pero no configurados en este spike.
