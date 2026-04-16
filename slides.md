---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://cover.sli.dev
# some information about your slides (markdown enabled)
title: Добро РФ | Сообщество
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# apply UnoCSS classes to the current slide
class: text-center
# https://sli.dev/features/drawing
drawings:
  persist: false
# slide transition: https://sli.dev/guide/animations.html#slide-transitions
transition: slide-left
# enable MDC Syntax: https://sli.dev/features/mdc
mdc: true
# duration of the presentation
duration: 35min
---

# Async Protocols

WS + socket io, redis

<div @click="$slidev.nav.next" class="mt-12 py-1" hover:bg="white op-10">
  Начать презентацию <carbon:arrow-right />
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
transition: fade-out
---

# О чем эта презентация?

Обзор асинхронных протоколов взаимодействия между сервисами
<br>
<br>
- <div v-click>Websocket, устройство под капотом</div>
- <div v-click>Socket io</div>
- <div v-click>Io redis</div>
<br>
<br>

---
transition: fade-out
---

# Для чего нужен websocket?

Кейсы real-time приложений
<br>
<br>
- <div v-click>Чат и уведомления</div>
- <div v-click>Браузерные игры</div>
- <div v-click>Совместное редактирование документов и файлов в браузере</div>
- <div v-click>Мониторинг и логирование</div>
- <div v-click>Финтех и графики</div>
<br>
<br>

---
layout: center
class: text-center
---

# Реализуем простой сервак с поддержкой ws протокола

---
transition: fade-out
---

# Импорты

```go
package main

import (
    "log"
    "net/http"
    "github.com/gorilla/websocket"
)
```
---
transition: fade-out
---

# Апгрейд соединения

```go
var upgrader = websocket.Upgrader{
    CheckOrigin: func(r *http.Request) bool { return true },
}

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Print("upgrade failed:", err)
        return
    }
    defer conn.Close()
    for {
        _, msg, err := conn.ReadMessage()
        if err != nil {
            break
        }
        log.Printf("received: %s", msg)
        conn.WriteMessage(websocket.TextMessage, []byte("echo: "+string(msg)))
    }
}
```
---
transition: fade-out
---

# Запуск сервера

```go
func main() {
    http.HandleFunc("/ws", wsHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
---
transition: fade-out
---
# Что мы сделали?

Сервер
<br>
<br>
- <div v-click>Инициализировали сервер</div>
- <div v-click>Прикрутили обычный http-эндпоинт, но вызвали некий апгрейд соединения</div>
- <div v-click>Начали обрабатывать поступающие сообщения</div>
<br>

---
transition: fade-out
---

# Клиент нам сейчас не сильно важен, но вообще он выглядит вот так

```go
import (
    "github.com/gorilla/websocket"
)
func main() {
    conn, _, err := websocket.DefaultDialer.Dial("ws://<IP_сервера_A>:8080/ws", nil)
    if err != nil {
        log.Fatal("dial:", err)
    }
    defer conn.Close()

    err = conn.WriteMessage(websocket.TextMessage, []byte("hello from client"))
    if err != nil {
        log.Fatal("write:", err)
    }

    _, msg, err := conn.ReadMessage()
    if err != nil {
        log.Fatal("read:", err)
    }
    log.Printf("received: %s", msg)
}
```
---
transition: fade-out
---
# Главные вопросы
- <div v-click>Как обычный http-запрос превращается в постоянный двусторонний канал ws?</div>
- <div v-click>Что же происходит во время "апгрейда" соединения?</div>

---
transition: fade-out
---

# Сначала разберем теорию
Порядок выполнения
- <div v-click>Клиент (браузер или код) открывает TCP-соединение с сервером и отправляет HTTP-запрос, но с особыми заголовками</div>

---
transition: fade-out
---

# Запрос

```http
GET /chat HTTP/1.1
Host: example.com
Connection: Upgrade
Upgrade: websocket
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```
---
transition: fade-out
---

# Сначала разберем теорию
Порядок выполнения
- Клиент (браузер или код) открывает TCP-соединение с сервером и отправляет HTTP-запрос, но с особыми заголовками
- <div v-click>Если сервер поддерживает ws то он отвечает 101 кодом</div>
---
transition: fade-out
---

# Ответ

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```
---
transition: fade-out
---
# Сначала разберем теорию
Порядок выполнения
- Клиент (браузер или код) открывает TCP-соединение с сервером и отправляет HTTP-запрос, но с особыми заголовками
- Если сервер поддерживает ws то он отвечает 101 кодом
- <div v-click>После ответа 101м TCP-соединение остается открытым</div>
- <div v-click>Обе стороны перестают ориентироваться на http-запросы/ответы и обмениваются фреймами (бинарными пакетами)</div>
---
transition: fade-out
---

# Практическая реализация без библиотек, импорты

```go
import (
    "bufio"
    "crypto/sha1"
    "encoding/base64"
    "fmt"
    "log"
    "net/http"
    "strings"
)
```

---
transition: fade-out
---

# Практическая реализация без библиотек, обработчик

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    if r.Header.Get("Upgrade") != "websocket" || r.Header.Get("Connection") != "Upgrade" {
        http.Error(w, "Not a websocket handshake", 400)
        return
    }
    key := r.Header.Get("Sec-WebSocket-Key")
    if key == "" {
        http.Error(w, "Missing Sec-WebSocket-Key", 400)
        return
    }
    accept := computeAcceptKey(key)
    w.Header().Set("Upgrade", "websocket")
    w.Header().Set("Connection", "Upgrade")
    w.Header().Set("Sec-WebSocket-Accept", accept)
    w.WriteHeader(101)
    
    ...
```

---
transition: fade-out
---

# Практическая реализация без библиотек, обработчик

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    ...
    
    
    hj, ok := w.(http.Hijacker)
    if !ok {
        http.Error(w, "Hijacking not supported", 500)
        return
    }
    conn, _, err := hj.Hijack()
    if err != nil {
        log.Println("Hijack error:", err)
        return
    }
    defer conn.Close()
    
    ...
```
---
transition: fade-out
---
# Практическая реализация без библиотек, обработчик

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    ...
    
    
    for {
        _, msg, err := readWebSocketFrame(conn)
        if err != nil {
            break
        }
        log.Printf("Received: %s", msg)
        writeWebSocketFrame(conn, []byte("echo: "+string(msg)))
    }
}

func main() {
    http.HandleFunc("/ws", wsHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
---
transition: fade-out
---
# Что мы сделали?
Порядок выполнения
- <div v-click>Проверили, что клиент хочет вебсокет открыть соединение</div>
- <div v-click>Выставили корректные заголовки и 101й статус</div>
- <div v-click>Применили метод <span v-mark.red="5">hijack</span></div>
- <div v-click>В горутине начали прослушивать соединение на предмет сообщений</div>
---
transition: fade-out
---
# Что за метод hijack?
Hijack
- <div v-click>Он "угоняет" соединение у http.Server и передает над ним контроль</div>
- <div v-click>Отключает автоматическую запись ответа</div>
- <div v-click>Держит соединение открытым, но заставляет самим его закрывать</div>
---
transition: fade-out
---
# Рассмотрим ещё раз процесс передачи и приема сообщений

```go
func wsHandler(w http.ResponseWriter, r *http.Request) {
    ...
    
    
    for {
        _, msg, err := readWebSocketFrame(conn)
        if err != nil {
            break
        }
        log.Printf("Received: %s", msg)
        writeWebSocketFrame(conn, []byte("echo: "+string(msg)))
    }
}
```
---
transition: fade-out
---
# Что нас здесь может смущать?
Чего то не хватает
- <div v-click>Нет типизированных ws клиента и ws сервера</div>
- <div v-click>Подключение ни чем не отличается от передачи сообщения</div>
- <div v-click>Нет привычных конструкций .on и .emit</div>

---
transition: fade-out
---
# Кто решает за нас эти проблемы?
И это...
- <div v-click>Библиотека <span v-mark.red="2">socket.io</span></div>

---
transition: fade-out
---
# Socket.IO
Это инструмент...
- <div v-click><span v-mark.red="1">Socket.IO</span> - это библиотека, которая делает всю рутинную работу (реконнект, группировка клиентов по комнатам, броадкасты, события, парсинг сообщений) и предоставляет api</div>
- <div v-click>Сам websocket - это протокол, основа и база</div>
- <div v-click>Выбор между чистым ws и socket.io - это всегда дилемму между производительностью и контролем vs скорость разработки и удобство</div>
- <div v-click>В вебпрактике вам в <span v-mark.red="4">99.999%</span> случаев будет достаточно socket.io</div>
---
transition: fade-out
---
# Что конктретно он умеет делать?
socket.io
- <div v-click>Подключение и реконнект</div>
- <div v-click>Событийный "роутинг" при отправке и получении сообщений</div>
- <div v-click>Группировка клиентов</div>
- <div v-click>Рассылка (широковещательная отправка сообщения всем клиентам)</div>
---
transition: fade-out
---
# Как выглядит код при использовании socket.io (сервер)

```go
import (
    socketio "github.com/googollee/go-socket.io" 
)

func main() {
    server := socketio.NewServer(nil)
    server.OnConnect("/", func(s socketio.Conn) error {
        log.Println("Connected:", s.ID())
        return nil
    })
    server.OnEvent("/", "new message", func(s socketio.Conn, msg string) {
        log.Println("Message:", msg)
        s.BroadcastTo("/", "chat message", msg)
    })
    server.OnDisconnect("/", func(s socketio.Conn, reason string) {
        log.Println("Disconnected:", reason)
    })
    go server.Serve()
    defer server.Close()
    http.Handle("/socket.io/", server)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```
---
transition: fade-out
---
# Какие проблемы могут возникнуть с ws серверами в базовой реализации?
ws
- <div v-click><span v-mark.red="1">Масштабирование</span></div>
- <div v-click>Иными словами, <span v-mark.red="2">общее</span> хранилище всех клиентов, событий и структур, которое шарилось бы между серваками бэка</div>
---
transition: fade-out
---
# IO redis / Redis adapter
redis
- <div v-click>Проблема ясна, socket io сам по себе оставляет множественные сервера <span v-mark.red="1">глухими</span></div>
- <div v-click>Что делает redis adapter?</div>
- <div v-click>Он создает в redis специальные каналы, на которые подписываются сервера socket io</div>
---
transition: fade-out
---
# IO redis / Redis adapter (каналы)
Каналы бывают
- <div v-click>Канал на каждый namespace</div>
- <div v-click>Канал на каждый namespace + на каждую комнату (room)</div>
---
transition: fade-out
---
# IO redis / Redis adapter (как это работает)
В общем виде
- <div v-click>У нас есть 2 сервера А и Б.</div>
- <div v-click>Клиент 1 присоединяется к namespace В. Он добавляется в массив клиентов сервера Б.</div>
- <div v-click>Клиент 2 тоже присоединяется к namespace В. Он добавляется в массив клиентов сервера А.</div>
- <div v-click>Клиент 1 отсылает сообщение, которое согласно нашей бизнес-логике должен получить Клиент 2.</div>
- <div v-click>Сообщение принимает Сервер Б. У него в массиве подключенных клиентов нет Клиента 2, поэтому он сразу же пересылает сообщение в канал redis.</div>
- <div v-click>Подписанный на канал Сервер А получает сообщение и отправляет его Клиенту 2.</div>
- <div v-click>Успех!</div>
---
transition: fade-out
---
# Лайфхак для PHP
От Ивана Поддубного
- <div v-click>Зная протокол каналов, можно заставлять event модель дергаться из php</div>
- <div v-click>Добавляя сообщения в нужный канал с нужным форматом</div>
---
layout: center
class: text-center
---

# Конец