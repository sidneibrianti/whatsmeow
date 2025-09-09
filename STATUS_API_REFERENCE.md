# WhatsApp Status API Reference - whatsmeow

Este documento serve como referência completa para implementar funcionalidades de Status (Stories) do WhatsApp usando a biblioteca whatsmeow em aplicações Go.

## Sumário

- [Visão Geral](#visão-geral)
- [Configuração Inicial](#configuração-inicial)
- [Status de Texto](#status-de-texto)
- [Status com Imagem](#status-com-imagem)
- [Status com Vídeo](#status-com-vídeo)
- [Formatos de Mídia Suportados](#formatos-de-mídia-suportados)
- [Configurações de Privacidade](#configurações-de-privacidade)
- [Exemplos Práticos](#exemplos-práticos)
- [Limitações e Considerações](#limitações-e-considerações)
- [Troubleshooting](#troubleshooting)

## Visão Geral

O WhatsApp Status (Stories) são mensagens temporárias visíveis para seus contatos por 24 horas. A biblioteca whatsmeow permite enviar status programaticamente usando o JID especial `types.StatusBroadcastJID`.

### Características dos Status:
- **Temporários**: Desaparecem automaticamente após 24 horas
- **Broadcast**: Enviados para todos os contatos (respeitando configurações de privacidade)
- **Tipos suportados**: Texto, imagem, vídeo, documentos
- **Visibilidade**: Controlada pelas configurações de privacidade do WhatsApp

## Configuração Inicial

### Dependências Necessárias

```go
import (
    "context"
    "log"
    
    "go.mau.fi/whatsmeow"
    "go.mau.fi/whatsmeow/store/sqlstore"
    "go.mau.fi/whatsmeow/types"
    waE2E "go.mau.fi/whatsmeow/binary/proto"
    waLog "go.mau.fi/whatsmeow/util/log"
    "google.golang.org/protobuf/proto"
)
```

### Inicialização do Cliente

```go
func initializeClient() (*whatsmeow.Client, error) {
    // Configurar logging
    dbLog := waLog.Stdout("Database", "INFO", true)
    clientLog := waLog.Stdout("Client", "INFO", true)
    
    // Configurar store SQLite
    container, err := sqlstore.New(context.Background(), 
        "sqlite3", 
        "file:whatsapp_store.db?_foreign_keys=on", 
        dbLog)
    if err != nil {
        return nil, fmt.Errorf("erro ao criar store: %v", err)
    }
    
    // Obter device store
    deviceStore, err := container.GetFirstDevice(context.Background())
    if err != nil {
        return nil, fmt.Errorf("erro ao obter device: %v", err)
    }
    
    // Criar cliente
    client := whatsmeow.NewClient(deviceStore, clientLog)
    
    return client, nil
}
```

## Status de Texto

### Status de Texto Simples

```go
func enviarStatusTexto(client *whatsmeow.Client, texto string) error {
    ctx := context.Background()
    
    _, err := client.SendMessage(ctx, types.StatusBroadcastJID, &waE2E.Message{
        Conversation: proto.String(texto),
    })
    
    if err != nil {
        return fmt.Errorf("erro ao enviar status de texto: %v", err)
    }
    
    return nil
}

// Exemplo de uso
err := enviarStatusTexto(client, "Meu status público! 🌟")
```

### Status de Texto Estendido

```go
func enviarStatusTextoEstendido(client *whatsmeow.Client, texto string) error {
    ctx := context.Background()
    
    _, err := client.SendMessage(ctx, types.StatusBroadcastJID, &waE2E.Message{
        ExtendedTextMessage: &waE2E.ExtendedTextMessage{
            Text: proto.String(texto),
            // Opcional: adicionar formatação, links, etc.
        },
    })
    
    if err != nil {
        return fmt.Errorf("erro ao enviar status estendido: %v", err)
    }
    
    return nil
}

// Exemplo com texto formatado
textoFormatado := `*Status em Negrito*
_Status em Itálico_
~Status Riscado~
\`\`\`Status em Código\`\`\`

🌟 Emojis funcionam perfeitamente!`

err := enviarStatusTextoEstendido(client, textoFormatado)
```

## Status com Imagem

### Upload e Envio de Imagem

```go
func enviarStatusImagem(client *whatsmeow.Client, imageData []byte, mimeType, caption string) error {
    ctx := context.Background()
    
    // 1. Upload da imagem
    uploaded, err := client.Upload(ctx, imageData, whatsmeow.MediaImage)
    if err != nil {
        return fmt.Errorf("erro no upload da imagem: %v", err)
    }
    
    // 2. Criar mensagem de imagem
    imageMsg := &waE2E.ImageMessage{
        URL:           proto.String(uploaded.URL),
        DirectPath:    proto.String(uploaded.DirectPath),
        MediaKey:      uploaded.MediaKey,
        Mimetype:      proto.String(mimeType),
        FileEncSHA256: uploaded.FileEncSHA256,
        FileSHA256:    uploaded.FileSHA256,
        FileLength:    proto.Uint64(uploaded.FileLength),
        Caption:       proto.String(caption), // Opcional
    }
    
    // 3. Enviar status
    _, err = client.SendMessage(ctx, types.StatusBroadcastJID, &waE2E.Message{
        ImageMessage: imageMsg,
    })
    
    if err != nil {
        return fmt.Errorf("erro ao enviar status com imagem: %v", err)
    }
    
    return nil
}
```

### Carregar Imagem de Arquivo

```go
import (
    "io/ioutil"
    "net/http"
    "path/filepath"
)

func carregarEEnviarImagem(client *whatsmeow.Client, caminhoArquivo, caption string) error {
    // Ler arquivo
    imageData, err := ioutil.ReadFile(caminhoArquivo)
    if err != nil {
        return fmt.Errorf("erro ao ler arquivo: %v", err)
    }
    
    // Detectar MIME type
    mimeType := http.DetectContentType(imageData)
    if !isValidImageMime(mimeType) {
        return fmt.Errorf("formato de imagem não suportado: %s", mimeType)
    }
    
    return enviarStatusImagem(client, imageData, mimeType, caption)
}

// Validação de MIME type
func isValidImageMime(mimeType string) bool {
    validTypes := []string{
        "image/jpeg",
        "image/jpg", 
        "image/png",
        "image/webp",
        "image/gif",
    }
    
    for _, validType := range validTypes {
        if mimeType == validType {
            return true
        }
    }
    return false
}
```

## Status com Vídeo

```go
func enviarStatusVideo(client *whatsmeow.Client, videoData []byte, mimeType, caption string) error {
    ctx := context.Background()
    
    // 1. Upload do vídeo
    uploaded, err := client.Upload(ctx, videoData, whatsmeow.MediaVideo)
    if err != nil {
        return fmt.Errorf("erro no upload do vídeo: %v", err)
    }
    
    // 2. Criar mensagem de vídeo
    videoMsg := &waE2E.VideoMessage{
        URL:           proto.String(uploaded.URL),
        DirectPath:    proto.String(uploaded.DirectPath),
        MediaKey:      uploaded.MediaKey,
        Mimetype:      proto.String(mimeType),
        FileEncSHA256: uploaded.FileEncSHA256,
        FileSHA256:    uploaded.FileSHA256,
        FileLength:    proto.Uint64(uploaded.FileLength),
        Caption:       proto.String(caption),
        Seconds:       proto.Uint32(30), // Duração em segundos (opcional)
    }
    
    // 3. Enviar status
    _, err = client.SendMessage(ctx, types.StatusBroadcastJID, &waE2E.Message{
        VideoMessage: videoMsg,
    })
    
    if err != nil {
        return fmt.Errorf("erro ao enviar status com vídeo: %v", err)
    }
    
    return nil
}
```

## Formatos de Mídia Suportados

### Imagens

| Formato | MIME Type | Extensão | Recomendação |
|---------|-----------|----------|--------------|
| JPEG | `image/jpeg` | `.jpg`, `.jpeg` | ✅ Recomendado para fotos |
| PNG | `image/png` | `.png` | ✅ Recomendado para imagens com transparência |
| WebP | `image/webp` | `.webp` | ✅ Boa compressão |
| GIF | `image/gif` | `.gif` | ⚠️ Limitado (tamanho/duração) |

### Vídeos

| Formato | MIME Type | Extensão | Recomendação |
|---------|-----------|----------|--------------|
| MP4 | `video/mp4` | `.mp4` | ✅ Recomendado |
| AVI | `video/avi` | `.avi` | ⚠️ Pode precisar conversão |
| MOV | `video/quicktime` | `.mov` | ⚠️ Pode precisar conversão |

### Limitações de Tamanho

```go
const (
    MaxImageSize = 16 * 1024 * 1024  // 16 MB
    MaxVideoSize = 64 * 1024 * 1024  // 64 MB
    MaxImageDimension = 4096         // 4096x4096 pixels
)

func validarTamanhoArquivo(data []byte, isVideo bool) error {
    size := len(data)
    maxSize := MaxImageSize
    
    if isVideo {
        maxSize = MaxVideoSize
    }
    
    if size > maxSize {
        return fmt.Errorf("arquivo muito grande: %d bytes (máximo: %d)", size, maxSize)
    }
    
    return nil
}
```

## Configurações de Privacidade

### Verificar Configurações Atuais

```go
import "go.mau.fi/whatsmeow/appstate"

func sincronizarConfiguracoes(client *whatsmeow.Client) error {
    ctx := context.Background()
    
    // Sincronizar configurações de privacidade
    err := client.FetchAppState(ctx, appstate.WAPatchName_critical_unblock_low, false, false)
    if err != nil {
        return fmt.Errorf("erro ao sincronizar configurações: %v", err)
    }
    
    return nil
}
```

### Opções de Privacidade (Configuradas no App)

As configurações de privacidade do status são controladas pelo aplicativo WhatsApp:

1. **Meus contatos** - Todos os contatos podem ver
2. **Meus contatos exceto...** - Todos exceto contatos específicos
3. **Compartilhar apenas com...** - Apenas contatos selecionados

⚠️ **Importante**: Essas configurações não podem ser alteradas programaticamente via API.

## Exemplos Práticos

### Exemplo 1: Sistema de Status Automático

```go
type StatusManager struct {
    client *whatsmeow.Client
}

func NewStatusManager(client *whatsmeow.Client) *StatusManager {
    return &StatusManager{client: client}
}

func (sm *StatusManager) EnviarStatusDiario(mensagem string) error {
    timestamp := time.Now().Format("15:04 - 02/01/2006")
    textoCompleto := fmt.Sprintf("%s\n\n📅 %s", mensagem, timestamp)
    
    return enviarStatusTexto(sm.client, textoCompleto)
}

func (sm *StatusManager) EnviarStatusComImagem(imagemPath, caption string) error {
    return carregarEEnviarImagem(sm.client, imagemPath, caption)
}

// Uso
statusManager := NewStatusManager(client)
err := statusManager.EnviarStatusDiario("Bom dia! 🌅")
```

### Exemplo 2: Status com Diferentes Tipos de Mídia

```go
func exemploStatusVariados(client *whatsmeow.Client) {
    // Status de texto
    enviarStatusTexto(client, "Status de texto simples")
    
    // Status com formatação
    textoFormatado := "*Negrito* _Itálico_ `Código`"
    enviarStatusTextoEstendido(client, textoFormatado)
    
    // Status com imagem
    carregarEEnviarImagem(client, "foto.jpg", "Legenda da foto")
    
    // Status com vídeo
    videoData, _ := ioutil.ReadFile("video.mp4")
    enviarStatusVideo(client, videoData, "video/mp4", "Vídeo interessante!")
}
```

### Exemplo 3: Status Programado

```go
import "time"

func agendarStatus(client *whatsmeow.Client, mensagem string, quando time.Time) {
    duracao := time.Until(quando)
    if duracao <= 0 {
        log.Println("Horário já passou")
        return
    }
    
    timer := time.NewTimer(duracao)
    go func() {
        <-timer.C
        err := enviarStatusTexto(client, mensagem)
        if err != nil {
            log.Printf("Erro ao enviar status programado: %v", err)
        } else {
            log.Println("Status programado enviado com sucesso!")
        }
    }()
}

// Agendar para 15:30 de hoje
agendamento := time.Date(time.Now().Year(), time.Now().Month(), time.Now().Day(), 15, 30, 0, 0, time.Local)
agendarStatus(client, "Status programado! ⏰", agendamento)
```

## Limitações e Considerações

### Limitações Técnicas

1. **Status é Experimental**: A funcionalidade pode não funcionar perfeitamente
2. **Listas Grandes**: Pode falhar com muitos contatos
3. **Rate Limiting**: WhatsApp pode limitar envios frequentes
4. **Sem Confirmação**: Não há callback de entrega para status

### Boas Práticas

```go
// 1. Sempre verificar erros
func enviarStatusSeguro(client *whatsmeow.Client, texto string) {
    err := enviarStatusTexto(client, texto)
    if err != nil {
        log.Printf("Falha ao enviar status: %v", err)
        // Implementar retry ou notificação
    }
}

// 2. Validar entrada
func validarTextoStatus(texto string) error {
    if len(texto) == 0 {
        return fmt.Errorf("texto não pode estar vazio")
    }
    if len(texto) > 700 { // Limite aproximado
        return fmt.Errorf("texto muito longo: %d caracteres", len(texto))
    }
    return nil
}

// 3. Implementar retry
func enviarStatusComRetry(client *whatsmeow.Client, texto string, maxTentativas int) error {
    var ultimoErro error
    
    for i := 0; i < maxTentativas; i++ {
        err := enviarStatusTexto(client, texto)
        if err == nil {
            return nil
        }
        
        ultimoErro = err
        time.Sleep(time.Duration(i+1) * time.Second) // Backoff
    }
    
    return fmt.Errorf("falha após %d tentativas: %v", maxTentativas, ultimoErro)
}
```

## Troubleshooting

### Problemas Comuns

#### 1. Status não aparece para contatos

```go
// Verificar se está usando o JID correto
if jid.String() != "status@broadcast" {
    log.Printf("JID incorreto: %s (deveria ser status@broadcast)", jid.String())
}

// Verificar configurações de privacidade
sincronizarConfiguracoes(client)
```

#### 2. Erro de upload de mídia

```go
func debugUpload(client *whatsmeow.Client, data []byte, mediaType whatsmeow.MediaType) {
    log.Printf("Tentando upload: %d bytes, tipo: %v", len(data), mediaType)
    
    uploaded, err := client.Upload(context.Background(), data, mediaType)
    if err != nil {
        log.Printf("Erro de upload: %v", err)
        return
    }
    
    log.Printf("Upload bem-sucedido: URL=%s", uploaded.URL)
}
```

#### 3. Cliente desconectado

```go
func verificarConexao(client *whatsmeow.Client) error {
    if !client.IsConnected() {
        log.Println("Cliente desconectado, tentando reconectar...")
        return client.Connect()
    }
    return nil
}

// Usar antes de enviar status
if err := verificarConexao(client); err != nil {
    return fmt.Errorf("falha na conexão: %v", err)
}
```

### Logging e Debug

```go
// Habilitar logs detalhados
clientLog := waLog.Stdout("Client", "DEBUG", true)
client := whatsmeow.NewClient(deviceStore, clientLog)

// Handler de eventos para debug
client.AddEventHandler(func(evt interface{}) {
    switch v := evt.(type) {
    case *events.Connected:
        log.Println("Cliente conectado e pronto")
    case *events.Disconnected:
        log.Printf("Cliente desconectado: %v", v.Reason)
    }
})
```

## Referências

- **Documentação oficial**: [whatsmeow GitHub](https://github.com/tulir/whatsmeow)
- **Protocol Buffers**: Definições em `binary/proto/`
- **Tipos JID**: Arquivo `types/jid.go`
- **Media Types**: Constantes em `client.go`

---

**Versão**: whatsmeow v0.0.0-dev  
**Atualizado em**: Setembro 2025  
**Compatibilidade**: Go 1.19+
