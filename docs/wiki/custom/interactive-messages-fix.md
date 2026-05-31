# Interactive Messages Fix — NativeFlowMessage Migration

## Contexto

O WhatsApp descontinuou os formatos legados `ListMessage` e `CarouselMessage(1 card)` nos clientes modernos (2025+).
Clientes novos exibem "Não foi possível carregar a mensagem" (Web) ou "Versão não compatível" (mobile).

Este documento registra as correções aplicadas no fork `lc1868/evolution-go` e serve como guia para reaplicar as mudanças após syncs com o upstream `evolution-foundation/evolution-go`.

---

## Arquivos modificados

| Arquivo | Tipo de mudança |
|---|---|
| `pkg/sendMessage/service/send_service.go` | Lógica principal — migração de formato |
| `pkg/sendMessage/handler/send_handler.go` | Handler `/send/pix` dedicado + validação `review_and_pay` |
| `pkg/routes/routes.go` | Rota `POST /send/pix` |

---

## Mudança 1 — `/send/button`: CarouselMessage → NativeFlowMessage

### Problema
Botões do tipo `reply`, `url`, `call`, `copy` eram enviados como `CarouselMessage` com 1 card.
Esse formato não renderiza mais no WhatsApp Web nem em versões recentes do app.

### Localização
`pkg/sendMessage/service/send_service.go` — função `SendButton`, bloco `else` (tipos não-pix, não-review_and_pay).

### Antes
```go
msg = &waE2E.Message{
    InteractiveMessage: &waE2E.InteractiveMessage{
        InteractiveMessage: &waE2E.InteractiveMessage_CarouselMessage_{
            CarouselMessage: &waE2E.InteractiveMessage_CarouselMessage{
                Cards:          []*waE2E.InteractiveMessage{card},
                MessageVersion: proto.Int32(1),
            },
        },
        ContextInfo: &waE2E.ContextInfo{},
    },
    ...
}
```

### Depois
```go
msg = &waE2E.Message{
    InteractiveMessage: &waE2E.InteractiveMessage{
        Header: header,  // título + mídia direto no root
        Body:   &waE2E.InteractiveMessage_Body{Text: proto.String(data.Description)},
        InteractiveMessage: &waE2E.InteractiveMessage_NativeFlowMessage_{
            NativeFlowMessage: &waE2E.InteractiveMessage_NativeFlowMessage{
                Buttons:           buttons,
                MessageParamsJSON: messageParamsJSON,  // era ausente no path antigo
                MessageVersion:    proto.Int32(1),
            },
        },
        ContextInfo: &waE2E.ContextInfo{},
    },
    ...
}
```

### Regra de ouro
- `pix` / `review_and_pay` → `NativeFlowMessage` puro (sem header)
- `reply` / `url` / `call` / `copy` → `NativeFlowMessage` puro **com** header (título + mídia opcional)
- `CarouselMessage` → usado apenas em `/send/carousel` com múltiplos cards

---

## Mudança 2 — `/send/list`: ListMessage → NativeFlowMessage single_select

### Problema
`ListMessage` (formato legado) não renderiza em clientes modernos.

### Localização
`pkg/sendMessage/service/send_service.go` — função `SendList`.

### Antes
```go
msg := &waE2E.Message{
    ListMessage: &waE2E.ListMessage{
        Title:      proto.String(data.Title),
        ButtonText: proto.String(buttonText),
        Sections:   sections,  // []*waE2E.ListMessage_Section
        ...
    },
}
```

### Depois
```go
// Monta JSON das seções no formato NativeFlow
listJSON, _ := json.Marshal(nfList{Title: buttonText, Sections: nfSections})
paramsJSON, _ := nativeFlowButtonParamsJSON(map[string]any{"list": json.RawMessage(listJSON)})

msg = &waE2E.Message{
    InteractiveMessage: &waE2E.InteractiveMessage{
        Header: &waE2E.InteractiveMessage_Header{Title: proto.String(data.Title), ...},
        Body:   &waE2E.InteractiveMessage_Body{Text: proto.String(data.Description)},
        InteractiveMessage: &waE2E.InteractiveMessage_NativeFlowMessage_{
            NativeFlowMessage: &waE2E.InteractiveMessage_NativeFlowMessage{
                Buttons: []*waE2E.InteractiveMessage_NativeFlowMessage_NativeFlowButton{{
                    Name:             proto.String("single_select"),
                    ButtonParamsJSON: paramsJSON,
                }},
                MessageParamsJSON: messageParamsJSON,
                MessageVersion:    proto.Int32(1),
            },
        },
        ...
    },
}
```

### Estrutura JSON do paramsJSON para single_select
```json
{
  "list": {
    "title": "Ver Menu",
    "sections": [
      {
        "title": "Planos",
        "highlight_label": "",
        "rows": [
          { "header": "Básico", "title": "Básico", "description": "R$ 29,90", "id": "row_0" }
        ]
      }
    ]
  }
}
```

---

## Mudança 3 — `/send/pix` como rota dedicada

### Problema
PIX existia apenas como tipo de botão dentro de `/send/button`. A Impa365 e a expectativa do mercado é ter endpoint dedicado.

### Adicionado
- `pkg/routes/routes.go`: `routes.POST("/send/pix", ..., r.sendHandler.SendPix)`
- `pkg/sendMessage/handler/send_handler.go`: handler `SendPix`

### Payload esperado
```json
{
  "number": "5511999999999",
  "headerTitle": "Pagamento PIX",
  "bodyText": "Realize o pagamento via PIX",
  "footerText": "Pagamento seguro",
  "merchantName": "Loja LTDA",
  "pixKey": "12345678901",
  "keyType": "CPF"
}
```
`keyType` aceita: `CPF`, `CNPJ`, `EMAIL`, `PHONE`, `EVP`

---

## Como reaplicar após sync com upstream

### Cenário: upstream lançou nova versão e você fez `git merge origin/main`

1. **Verificar conflitos** nos 3 arquivos acima
2. **Em `send_service.go`**, garantir:
   - Função `SendButton` bloco `else` usa `NativeFlowMessage` (não `CarouselMessage`)
   - Função `SendList` usa `InteractiveMessage + NativeFlowMessage single_select` (não `ListMessage`)
   - `messageParamsJSON` presente em todos os paths de `SendButton`
3. **Em `send_handler.go`**, garantir que `SendPix` existe e valida os campos obrigatórios
4. **Em `routes.go`**, garantir `POST /send/pix` registrado

### Checklist de teste pós-sync
```
[ ] POST /send/button com type=reply    → renderiza botão no Web e celular
[ ] POST /send/button com type=url      → abre link
[ ] POST /send/button com type=call     → mostra ligação
[ ] POST /send/button com type=copy     → copia código
[ ] POST /send/button com imageUrl      → renderiza com imagem no header
[ ] POST /send/button com type=pix      → botão PIX nativo
[ ] POST /send/button com type=review_and_pay → revisão de pedido
[ ] POST /send/pix                      → botão PIX dedicado
[ ] POST /send/list                     → lista abre no celular e Web
[ ] POST /send/carousel                 → carrossel renderiza (não mexido)
```

---

## Referência — formatos NativeFlow por tipo

| Tipo | `name` do botão | Campos obrigatórios em params |
|---|---|---|
| reply | `quick_reply` | `display_text`, `id` |
| url | `cta_url` | `display_text`, `url`, `merchant_url` |
| call | `cta_call` | `display_text`, `phone_number` |
| copy | `cta_copy` | `display_text`, `copy_code` |
| pix | `payment_info` | `currency`, `name`, `keyType`, `key` |
| review_and_pay | `review_and_pay` | (ver `buildReviewAndPayParams`) |
| lista | `single_select` | `list.title`, `list.sections[].rows[]` |

---

## Docker Hub

Imagem: `lc1868/evolution-go:latest`
Plataformas: `linux/amd64`, `linux/arm64`

Para rebuild após mudanças:
```bash
docker buildx build --platform linux/amd64 -t lc1868/evolution-go:amd64 --push .
docker buildx build --platform linux/arm64 -t lc1868/evolution-go:arm64 --push .
docker buildx imagetools create -t lc1868/evolution-go:latest lc1868/evolution-go:amd64 lc1868/evolution-go:arm64
```
