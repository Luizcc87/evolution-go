# Proxy Health (Logs + Status por Instância)

O Evolution GO suporta configuração de proxy por instância. Esta funcionalidade adiciona um monitor periódico que verifica a conectividade do proxy e registra o resultado nos logs do container (stdout), além de expor o status via API.

---

## O que aparece nos logs

O monitor escreve linhas com o prefixo:

- `[proxy-health]`

Ele registra:

- Um evento por instância quando houver mudança de status, ou quando o status estiver `slow`/`error`
- Um resumo ao final de cada ciclo

Exemplos:

```text
[proxy-health] instance=<uuid> proxy=socks5://proxy.exemplo.com:1080 status=active latency_ms=120
[proxy-health] instance=<uuid> proxy=http://proxy.exemplo.com:8080 status=slow latency_ms=2200 threshold_ms=1500
[proxy-health] instance=<uuid> proxy=http://proxy.exemplo.com:8080 status=error latency_ms=40 error=proxy authentication failed (HTTP 407)
[proxy-health] cycle complete total=10 active=7 slow=1 error=2 inactive=0 duration_ms=85
```

Notas:

- `proxy` é exibido como `protocol://host:port` (não inclui usuário/senha)
- `latency_ms` é o tempo de conexão/handshake até o proxy

---

## Como habilitar

Por padrão, o monitor fica desabilitado. Para habilitar, configure as variáveis no serviço/container do Evolution GO:

```env
PROXY_HEALTH_ENABLED=true
PROXY_HEALTH_INTERVAL_S=60
PROXY_HEALTH_TIMEOUT_MS=3000
PROXY_HEALTH_MAX_LAT_MS=1500
PROXY_HEALTH_HTTP_URL=http://example.invalid/
```

---

## Como consultar via API

Além de logs, o status é persistido e pode ser consultado por endpoints:

- Com token da instância (Auth):
  - `GET /instance/proxy/status`
- Admin (GLOBAL_API_KEY / AuthAdmin):
  - `GET /instance/proxy/status/:instanceId`
  - `GET /instance/proxy/status/all`

Campos retornados:

- `instanceId`
- `proxyAddress`
- `status` (`inactive|active|slow|error`)
- `lastCheck`
- `latencyMs`
- `error`
- `thresholdMs`

---

## Exemplo de Stack (Portainer / Docker Swarm)

Um exemplo completo de stack para Swarm/Portainer está disponível em:

- `docker/examples/stack-portainer-swarm-proxy-health.yml`

