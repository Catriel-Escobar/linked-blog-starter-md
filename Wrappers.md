## 🎁 4. Wrapper Pattern - Decorating Objects

### Concepto
Envolver un objeto para agregar funcionalidad sin modificar el original.

### Problema Resuelto
En el middleware de timeout, necesitábamos:
- Detectar si el handler ya escribió una respuesta
- Evitar que timeout escriba si handler ya escribió
- Thread-safe (goroutines concurrentes)

### Implementación

```go
// Wrapper que envuelve http.ResponseWriter
type timeoutWriter struct {
    http.ResponseWriter  // Composición (embedded field)
    mu          sync.Mutex
    wroteHeader bool     // Estado adicional
}

// Override del método Write
func (tw *timeoutWriter) Write(b []byte) (int, error) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    
    if !tw.wroteHeader {
        tw.wroteHeader = true
    }
    
    return tw.ResponseWriter.Write(b)
}

// Override del método WriteHeader
func (tw *timeoutWriter) WriteHeader(code int) {
    tw.mu.Lock()
    defer tw.mu.Unlock()
    
    if !tw.wroteHeader {
        tw.wroteHeader = true
        tw.ResponseWriter.WriteHeader(code)
    }
}
```

### Uso en Middleware

```go
// Crear wrapper
tw := &timeoutWriter{ResponseWriter: w, wroteHeader: false}

// Pasar wrapper al handler (en lugar del ResponseWriter original)
next.ServeHTTP(tw, r)

// Verificar si ya se escribió
if !tw.wroteHeader {
    http.Error(tw.ResponseWriter, "Timeout", 408)
}
```

### Por qué funciona
1. **Composición en Go**: `timeoutWriter` "hereda" todos los métodos de `http.ResponseWriter`
2. **Override selectivo**: Solo reemplazamos `Write()` y `WriteHeader()`
3. **Transparente**: El handler no sabe que está usando un wrapper

### Otros Usos de Wrappers
```go
// Logging wrapper
type loggingWriter struct {
    http.ResponseWriter
    statusCode int
}

// Compression wrapper
type gzipWriter struct {
    http.ResponseWriter
    Writer *gzip.Writer
}

// Metrics wrapper
type metricsWriter struct {
    http.ResponseWriter
    bytesWritten int
}
```

### Conceptos relacionados
- [[Decorator Pattern]]
- [[Composition Over Inheritance]]
- [[Open/Closed Principle]]