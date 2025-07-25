# Arquitectura TCP Socket: Comunicación Cliente-Servidor

## Visión General

El proyecto TCP socket se basa en un modelo cliente-servidor donde el servidor escucha conexiones entrantes y los clientes se conectan para intercambiar datos. TCP proporciona comunicación confiable y ordenada a costa de mayor latencia comparado con UDP.

## Arquitectura del Servidor

El servidor TCP implementa los siguientes pasos fundamentales:

### 1. Creación y Configuración del Socket
```cpp
int server_fd = socket(AF_INET, SOCK_STREAM, 0);
```
- `AF_INET`: Especifica IPv4
- `SOCK_STREAM`: Indica protocolo TCP (conexión confiable)

### 2. Optimizaciones Críticas
```cpp
int opt = 1;
setsockopt(server_fd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```
- **TCP_NODELAY**: Deshabilita el algoritmo de Nagle para envío inmediato
- **SO_REUSEADDR**: Permite reutilizar la dirección sin esperar TIME_WAIT

### 3. Binding y Listening
```cpp
bind(server_fd, (struct sockaddr *)&address, sizeof(address));
listen(server_fd, 1);
```
- **bind()**: Asocia el socket con una dirección IP y puerto específicos
- **listen()**: Pone el socket en modo de escucha con backlog de 1 conexión

### 4. Aceptación de Conexiones
```cpp
int client_socket = accept(server_fd, (struct sockaddr *)&address, &addrlen);
```
- **accept()**: Bloquea hasta recibir una conexión

## Arquitectura del Cliente

### 1. Creación y Conexión
```cpp
int sock = socket(AF_INET, SOCK_STREAM, 0);
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
```
- Mismas optimizaciones que el servidor
- **connect()**: Inicia el 3-way handshake TCP

### 2. Intercambio de Datos con Medición
```cpp
auto inicio = std::chrono::high_resolution_clock::now();
send(sock, input.c_str(), input.length(), 0);
recv(sock, buffer, sizeof(buffer), 0);
auto fin = std::chrono::high_resolution_clock::now();
auto duracion = std::chrono::duration_cast<std::chrono::microseconds>(fin - inicio);
```

## Protocolo TCP: 3-Way Handshake

El establecimiento de conexión TCP requiere tres mensajes:

```
Cliente → Servidor: SYN (Solicitud de conexión)
Servidor → Cliente: SYN+ACK (Confirmación y solicitud)
Cliente → Servidor: ACK (Confirmación final)
```

**Importancia**: Este handshake ocurre solo UNA vez por conexión, no por mensaje.

## Conexión Persistente vs Conexión por Mensaje

### Conexión Persistente (Recomendado)
```cpp
connect();  // Una sola vez
while(true) {
    send();
    recv();
}
```
**Ventaja**: Elimina overhead de handshake en cada mensaje

### Conexión por Mensaje (Ineficiente)
```cpp
while(true) {
    connect();  // ¡Overhead innecesario!
    send();
    recv();
    close();
}
```
**Desventaja**: 3-way handshake en cada mensaje (~1-3ms adicionales)

## Estructuras de Datos Clave

### sockaddr_in
```cpp
struct sockaddr_in address;
address.sin_family = AF_INET;
address.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0 (todas las interfaces)
address.sin_port = htons(PORT);        // Network byte order
```

### Conversión de Byte Order
```cpp
htons(PORT);  // Host TO Network Short (16 bits)
inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);  // String a binario
```

## Optimizaciones de Latencia

### 1. TCP_NODELAY - La Más Crítica
- **Problema**: Algoritmo de Nagle agrupa paquetes pequeños
- **Efecto**: Delay de 40-200ms en aplicaciones interactivas
- **Solución**: Deshabilitar para envío inmediato
- **Impacto**: Latencia de ~2-5ms → ~200-500μs

### 2. High-Resolution Timing
```cpp
std::chrono::high_resolution_clock::now()
std::chrono::duration_cast<std::chrono::microseconds>()
```
- Precisión de nanosegundos para medición exacta
- Esencial para detectar optimizaciones de latencia

### 3. APIs Específicas de Socket
```cpp
send() / recv()  // Específicas para sockets
vs
write() / read() // Genéricas para file descriptors
```

## Comparación de Rendimiento

| Configuración | Latencia Típica | Uso Recomendado |
|---------------|-----------------|-----------------|
| TCP sin optimizar | 1-5 ms | ❌ Evitar |
| TCP + TCP_NODELAY | 200-500 μs | ✅ Aplicaciones normales |
| TCP + Conexión persistente | 100-300 μs | ✅ Aplicaciones críticas |
| UDP | 50-200 μs | ✅ Gaming, streaming |

## Stack de Protocolos

```
┌─────────────────┐
│   Aplicación    │ ← Socket API (send/recv)
├─────────────────┤
│   TCP           │ ← Reliability, flow control
├─────────────────┤
│   IP            │ ← Routing (127.0.0.1 = localhost)
├─────────────────┤
│   Loopback      │ ← Interface virtual en memoria
└─────────────────┘
```

## Casos de Uso Reales

### High-Frequency Trading
- **Requerimiento**: <100μs de latencia
- **Optimizaciones**: TCP_NODELAY + hardware especializado + kernel bypass
- **Protocolo**: TCP para órdenes, UDP multicast para market data

### Gaming Online
- **Requerimiento**: <50ms end-to-end
- **Estrategia**: UDP para game state, TCP para chat/lobby
- **Ejemplo**: Counter-Strike usa puertos TCP 27015,27036 y UDP 27015,27020,27031-27036

### IoT Sensors
- **MQTT sobre TCP**: Confiabilidad para datos críticos
- **CoAP sobre UDP**: Eficiencia energética para sensores battery-powered

## Debugging y Medición

### Herramientas de Red
```bash
netstat -tulpn | grep 8080    # Ver conexiones activas
tcpdump -i lo port 8080       # Capturar tráfico
ss -tuln                      # Estado de sockets
```

### Compilación Optimizada
```bash
g++ -O3 -march=native -flto -pthread programa.cpp -o programa
```

## Limitaciones y Consideraciones

### File Descriptor Limits
```bash
ulimit -n        # Ver límite actual (~1024)
ulimit -n 65536  # Aumentar límite
```

### Memory per Connection
- ~8-32KB por conexión TCP en kernel buffers
- Thread-per-connection: ~2-8MB stack por thread
- Event-driven: Memoria constante independiente de conexiones

### Escalabilidad
- **Thread-per-connection**: Hasta ~1000 conexiones
- **epoll/kqueue**: Hasta ~10000+ conexiones
- **Async I/O**: Escalabilidad limitada por CPU, no por I/O

### Consideración Key
En localhost (127.0.0.1), la pérdida de paquetes es prácticamente cero, por lo que UDP ofrece ventajas de velocidad sin sacrificar confiabilidad significativa.
