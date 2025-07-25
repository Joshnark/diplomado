# Arquitectura TCP Socket: Comunicación Cliente-Servidor

## Visión General

El proyecto TCP socket se basa en un modelo cliente-servidor local donde el servidor escucha conexiones entrantes y un cliente se conecta para intercambiar datos.

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
- **listen()**: Pone el socket en modo de escucha y espera por una unica conexion

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

## Optimizaciones de Latencia

### 1. TCP_NODELAY - La Más Crítica
- **Problema**: Algoritmo de Nagle agrupa paquetes pequeños
- **Efecto**: Delay de 40-200ms en aplicaciones interactivas
- **Solución**: Deshabilitar para envío inmediato
- **Impacto**: Latencia de ~2-5ms

### Conexión Persistente
- **Problema**: 3-way handshake se realiza durante la conexion
- **Efecto**: Este handshake genera un overhead que se traduce en mas delay en la comunicacion
- **Solución**: Conectar una sola vez, mantener la escucha en bucle
- **Impacto**: Latencia de ~1-2ms

Ejemplo:
```cpp
connect();  // Una sola vez
while(true) {
    send();
    recv();
}
```

## Comparación de Rendimiento

| Configuración | Latencia Típica | Uso Recomendado |
|---------------|-----------------|-----------------|
| TCP sin optimizar | 1-5 ms | ❌ Evitar |
| TCP + TCP_NODELAY | 200-500 μs | ✅ Aplicaciones normales |
| TCP + Conexión persistente | 100-300 μs | ✅ Aplicaciones críticas |
| UDP | 50-200 μs | ✅ Gaming, streaming |

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

## Limitaciones y Consideraciones

### Memory per Connection
- ~8-32KB por conexión TCP en kernel buffers
- Thread-per-connection: ~2-8MB stack por thread
- Event-driven: Memoria constante independiente de conexiones

### Escalabilidad
- **Thread-per-connection**: Hasta ~1000 conexiones
- **epoll/kqueue**: Hasta ~10000+ conexiones
- **Async I/O**: Escalabilidad limitada por CPU, no por I/O

### Consideración Clave
En localhost (127.0.0.1), la pérdida de paquetes es prácticamente cero, por lo que UDP ofrece ventajas de velocidad sin sacrificar confiabilidad significativa, no obstante se decidió usar TCP ya que es la base para protocolos que son standard en el internet del dia a dia como el protocolo RTSP
