# Labels no WhatsApp com whatsmeow

## Visão Geral

O sistema de labels (etiquetas) do WhatsApp permite organizar conversas e mensagens através de tags coloridas e nomeadas. No whatsmeow, os labels são gerenciados através do **App State**, que sincroniza automaticamente entre dispositivos.

## Arquitetura dos Labels

### Tipos de Labels
- **Labels de Chat**: Aplicados a conversas inteiras
- **Labels de Mensagem**: Aplicados a mensagens específicas
- **Labels Predefinidos**: Criados pelo WhatsApp
- **Labels Personalizados**: Criados pelo usuário

### Estrutura de Dados

```go
type LabelInfo struct {
    ID           string
    Name         string
    Color        int32
    ColorHex     string
    Deleted      bool
    PredefinedID string
}

// Cores disponíveis no WhatsApp
const (
    LabelCorVermelha   int32 = 0  // #FF3333
    LabelCorLaranja    int32 = 1  // #FF9500
    LabelCorAmarela    int32 = 2  // #FFCC02
    LabelCorVerde      int32 = 3  // #34C759
    LabelCorAzul       int32 = 4  // #007AFF
    LabelCorRoxa       int32 = 5  // #AF52DE
    LabelCorRosa       int32 = 6  // #FF2D92
    LabelCorCinza      int32 = 7  // #8E8E93
)
```

## 1. Criando e Editando Labels

### Criar Label Básico

```go
func criarLabel(client *whatsmeow.Client, labelID, nome string, cor int32) error {
    // Usar appstate.BuildLabelEdit seguindo o padrão do projeto
    patch := appstate.BuildLabelEdit(labelID, nome, cor, false)
    
    err := client.SendAppState(patch)
    if err != nil {
        return fmt.Errorf("erro ao criar label: %v", err)
    }
    
    log.Printf("Label '%s' criado com sucesso", nome)
    return nil
}
```

### Deletar Label

```go
func deletarLabel(client *whatsmeow.Client, labelID string) error {
    // BuildLabelEdit com deleted=true
    patch := appstate.BuildLabelEdit(labelID, "", 0, true)
    
    err := client.SendAppState(patch)
    if err != nil {
        return fmt.Errorf("erro ao deletar label: %v", err)
    }
    
    log.Printf("Label '%s' deletado", labelID)
    return nil
}
```

### Gerenciador de Labels Completo

```go
type LabelManager struct {
    client *whatsmeow.Client
    labels map[string]*LabelInfo
    synced bool
}

func NewLabelManager(client *whatsmeow.Client) *LabelManager {
    lm := &LabelManager{
        client: client,
        labels: make(map[string]*LabelInfo),
        synced: false,
    }
    
    // Seguir padrão event-driven do projeto
    client.AddEventHandler(lm.handleAppStateEvent)
    return lm
}

func (lm *LabelManager) CriarLabelsComuns() error {
    labels := map[string]struct{
        nome string
        cor  int32
    }{
        "importante": {"⭐ Importante", LabelCorVermelha},
        "trabalho":   {"💼 Trabalho", LabelCorAzul},
        "familia":    {"👨‍👩‍👧‍👦 Família", LabelCorVerde},
        "urgente":    {"🚨 Urgente", LabelCorVermelha},
        "pendente":   {"⏳ Pendente", LabelCorLaranja},
        "concluido":  {"✅ Concluído", LabelCorVerde},
    }
    
    for labelID, info := range labels {
        patch := appstate.BuildLabelEdit(labelID, info.nome, info.cor, false)
        err := lm.client.SendAppState(patch)
        if err != nil {
            return fmt.Errorf("erro ao criar label %s: %v", labelID, err)
        }
    }
    
    return nil
}
```

## 2. Aplicando Labels a Chats

### Label em Chat Individual

```go
func aplicarLabelChat(client *whatsmeow.Client, numeroTelefone, labelID string) error {
    // Usar padrão JID do projeto
    chatJID := types.JID{
        User:   numeroTelefone, // Ex: "5511999999999"
        Server: types.DefaultUserServer,
    }
    
    // BuildLabelChat para etiquetar o chat
    patch := appstate.BuildLabelChat(chatJID, labelID, true)
    
    err := client.SendAppState(patch)
    if err != nil {
        return fmt.Errorf("erro ao aplicar label ao chat: %v", err)
    }
    
    log.Printf("Label '%s' aplicado ao chat %s", labelID, chatJID)
    return nil
}
```

### Label em Grupo

```go
func aplicarLabelGrupo(client *whatsmeow.Client, grupoID, labelID string) error {
    // JID de grupo usa server diferente
    chatJID := types.JID{
        User:   grupoID,
        Server: types.GroupServer, // "g.us"
    }
    
    patch := appstate.BuildLabelChat(chatJID, labelID, true)
    return client.SendAppState(patch)
}
```

### Remover Label de Chat

```go
func removerLabelChat(client *whatsmeow.Client, chatJID types.JID, labelID string) error {
    // BuildLabelChat com labeled=false
    patch := appstate.BuildLabelChat(chatJID, labelID, false)
    
    err := client.SendAppState(patch)
    if err != nil {
        return fmt.Errorf("erro ao remover label: %v", err)
    }
    
    log.Printf("Label '%s' removido do chat %s", labelID, chatJID)
    return nil
}
```

## 3. Aplicando Labels a Mensagens

### Label em Mensagem Específica

```go
func aplicarLabelMensagem(client *whatsmeow.Client, chatJID types.JID, labelID, messageID string) error {
    // BuildLabelMessage para mensagens específicas
    patch := appstate.BuildLabelMessage(chatJID, labelID, messageID, true)
    
    err := client.SendAppState(patch)
    if err != nil {
        return fmt.Errorf("erro ao aplicar label à mensagem: %v", err)
    }
    
    log.Printf("Label '%s' aplicado à mensagem %s", labelID, messageID)
    return nil
}
```

### Sistema Automático de Labels para Mensagens

```go
type MessageLabelManager struct {
    client *whatsmeow.Client
    rules  map[string]string // palavra-chave -> labelID
}

func NewMessageLabelManager(client *whatsmeow.Client) *MessageLabelManager {
    mlm := &MessageLabelManager{
        client: client,
        rules: map[string]string{
            "urgente":    "urgente",
            "importante": "importante", 
            "spam":       "spam",
            "promoção":   "promocao",
        },
    }
    
    // Seguir padrão event-driven
    client.AddEventHandler(mlm.handleMessage)
    return mlm
}

func (mlm *MessageLabelManager) handleMessage(evt interface{}) {
    switch v := evt.(type) {
    case *events.Message:
        // Aplicar regras automáticas
        texto := strings.ToLower(v.Message.GetConversation())
        messageID := v.Info.ID
        chatJID := v.Info.Chat
        
        for palavra, labelID := range mlm.rules {
            if strings.Contains(texto, palavra) {
                mlm.aplicarLabelAuto(chatJID, labelID, string(messageID))
                break // Aplicar apenas o primeiro match
            }
        }
        
        // Regras baseadas em padrões
        if strings.HasSuffix(texto, "?") {
            mlm.aplicarLabelAuto(chatJID, "pendente", string(messageID))
        }
    }
}

func (mlm *MessageLabelManager) aplicarLabelAuto(chatJID types.JID, labelID, messageID string) {
    patch := appstate.BuildLabelMessage(chatJID, labelID, messageID, true)
    err := mlm.client.SendAppState(patch)
    if err != nil {
        log.Printf("Erro ao aplicar label automático: %v", err)
    }
}
```

## 4. Listando Labels Existentes

### Handler de App State para Labels

```go
func (lm *LabelManager) handleAppStateEvent(evt interface{}) {
    switch v := evt.(type) {
    case *events.AppState:
        // Processar mudanças nos labels
        for _, patch := range v.Patches {
            for _, mutation := range patch.Mutations {
                if len(mutation.Index) >= 2 && mutation.Index[0] == "label_edit" {
                    lm.processLabelMutation(mutation)
                }
            }
        }
        lm.synced = true
    }
}

func (lm *LabelManager) processLabelMutation(mutation appstate.MutationInfo) {
    labelID := mutation.Index[1]
    
    if mutation.Value != nil && mutation.Value.LabelEditAction != nil {
        action := mutation.Value.LabelEditAction
        
        // Buscar ou criar label
        if lm.labels[labelID] == nil {
            lm.labels[labelID] = &LabelInfo{ID: labelID}
        }
        
        label := lm.labels[labelID]
        
        // Atualizar dados seguindo padrão de unwrapping
        if action.Name != nil {
            label.Name = action.GetName()
        }
        if action.Color != nil {
            label.Color = action.GetColor()
            label.ColorHex = colorToHex(action.GetColor())
        }
        if action.Deleted != nil {
            label.Deleted = action.GetDeleted()
        }
    }
}
```

### Listar Labels com Sincronização

```go
func (lm *LabelManager) ListarLabels(ctx context.Context) ([]LabelInfo, error) {
    // Sincronizar app state se necessário
    if !lm.synced {
        err := lm.client.FetchAppState(ctx, appstate.WAPatchNameRegularLow, false, false)
        if err != nil {
            return nil, fmt.Errorf("erro ao sincronizar: %v", err)
        }
        
        // Aguardar sincronização com timeout
        timeout := time.NewTimer(5 * time.Second)
        ticker := time.NewTicker(100 * time.Millisecond)
        defer timeout.Stop()
        defer ticker.Stop()
        
        for !lm.synced {
            select {
            case <-timeout.C:
                return nil, fmt.Errorf("timeout aguardando sincronização")
            case <-ticker.C:
                // Continue aguardando
            }
        }
    }
    
    // Retornar labels não deletados
    var result []LabelInfo
    for _, label := range lm.labels {
        if !label.Deleted {
            result = append(result, *label)
        }
    }
    
    return result, nil
}

func colorToHex(color int32) string {
    colors := map[int32]string{
        0: "#FF3333", 1: "#FF9500", 2: "#FFCC02", 3: "#34C759",
        4: "#007AFF", 5: "#AF52DE", 6: "#FF2D92", 7: "#8E8E93",
    }
    
    if hex, exists := colors[color]; exists {
        return hex
    }
    return "#000000"
}
```

## 5. Exemplo de Uso Completo

```go
package main

import (
    "context"
    "fmt"
    "log"
    "time"

    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/store/sqlstore"
    waLog "go.mau.fi/whatsmeow/util/log"
)

func main() {
    // Inicialização seguindo padrão do projeto
    dbLog := waLog.Stdout("Database", "INFO", true)
    container, err := sqlstore.New("sqlite3", "file:store.db?_foreign_keys=on", dbLog)
    if err != nil {
        panic(err)
    }

    deviceStore, err := container.GetFirstDevice()
    if err != nil {
        panic(err)
    }

    clientLog := waLog.Stdout("Client", "INFO", true)
    client := whatsmeow.NewClient(deviceStore, clientLog)

    // Conectar usando padrão de QR
    if client.Store.ID == nil {
        qrChan, _ := client.GetQRChannel(context.Background())
        err = client.Connect()
        if err != nil {
            panic(err)
        }
        for evt := range qrChan {
            if evt.Event == "code" {
                fmt.Println("QR code:", evt.Code)
            }
        }
    } else {
        err = client.Connect()
        if err != nil {
            panic(err)
        }
    }

    // Aguardar conexão
    for !client.IsConnected() {
        time.Sleep(100 * time.Millisecond)
    }

    // Criar gerenciadores
    labelMgr := NewLabelManager(client)
    messageLabelMgr := NewMessageLabelManager(client)

    ctx := context.Background()

    // 1. Criar labels comuns
    err = labelMgr.CriarLabelsComuns()
    if err != nil {
        log.Printf("Erro ao criar labels: %v", err)
    }

    // 2. Aguardar sincronização
    time.Sleep(3 * time.Second)

    // 3. Listar labels existentes
    labels, err := labelMgr.ListarLabels(ctx)
    if err != nil {
        log.Printf("Erro ao listar: %v", err)
    } else {
        fmt.Printf("=== LABELS ENCONTRADOS (%d) ===\n", len(labels))
        for _, label := range labels {
            fmt.Printf("• %s (%s) - %s\n", label.Name, label.ID, label.ColorHex)
        }
    }

    // 4. Aplicar label a chat
    chatJID := types.JID{
        User:   "5511999999999", // Substitua por número real
        Server: types.DefaultUserServer,
    }
    
    err = aplicarLabelChat(client, chatJID.User, "importante")
    if err != nil {
        log.Printf("Erro ao aplicar label: %v", err)
    }

    // 5. Aplicar label a mensagem específica
    messageID := "3EB0123456789ABCDEF" // Substitua por ID real
    err = aplicarLabelMensagem(client, chatJID, "urgente", messageID)
    if err != nil {
        log.Printf("Erro ao etiquetar mensagem: %v", err)
    }

    // Manter rodando para processar eventos
    time.Sleep(30 * time.Second)
    client.Disconnect()
}
```

## 6. Monitoramento de Eventos de Label

```go
func handleLabelEvents(evt interface{}) {
    switch v := evt.(type) {
    case *events.LabelAssociationMessage:
        action := "adicionado"
        if !v.Action.GetLabeled() {
            action = "removido"
        }
        
        log.Printf("Label %s %s da mensagem %s no chat %s", 
            v.LabelID, action, v.MessageID, v.JID)
            
    case *events.LabelAssociationChat:
        action := "adicionado"
        if !v.Action.GetLabeled() {
            action = "removido"
        }
        
        log.Printf("Label %s %s do chat %s", 
            v.LabelID, action, v.JID)
    }
}

// Adicionar ao cliente
client.AddEventHandler(handleLabelEvents)
```

## Pontos Importantes

### Limitações e Considerações

1. **Cores Limitadas**: WhatsApp suporta apenas 8 cores predefinidas (0-7)
2. **Sincronização**: Labels são sincronizados via App State entre dispositivos
3. **IDs Únicos**: Cada label deve ter um ID único como string
4. **Performance**: Use batch operations com delays para evitar rate limiting
5. **Persistência**: Labels são mantidos no banco de dados do WhatsApp

### Melhores Práticas

- Sempre verificar `client.Store.ID == nil` antes de operações
- Usar event handlers para monitorar mudanças em tempo real
- Implementar timeouts em operações de sincronização
- Tratar erros de conexão com `*whatsmeow.DisconnectedError`
- Usar o padrão JID correto para diferentes tipos de chat

### Debugging

```go
// Habilitar logs de debug para app state
clientLog := waLog.Stdout("Client", "DEBUG", true)

// Monitorar mudanças de app state
client.AddEventHandler(func(evt interface{}) {
    if appState, ok := evt.(*events.AppState); ok {
        log.Printf("App state atualizado: %d patches", len(appState.Patches))
    }
})
```

Os labels no whatsmeow seguem o padrão event-driven e App State do projeto, permitindo organização eficiente de conversas e mensagens com sincronização automática entre dispositivos.
