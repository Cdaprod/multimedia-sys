Absolutely, adopting a modular, abstract, and extensible architecture is a wise approach for building a scalable and maintainable multimedia system. By factoring your system into atomic, generic components and interfacing them through a single facade adapter, you can ensure that your Cdaprod/multimedia-sys repository remains flexible and adaptable to future enhancements.

Below, I will guide you through designing such a system in Golang, incorporating best practices and design patterns to achieve your objectives.

1. Architectural Overview

1.1. Objectives

	•	Modularity: Break down the system into independent, reusable components.
	•	Abstraction: Define clear interfaces for each component to hide implementation details.
	•	Extensibility: Facilitate easy addition of new functionalities without altering existing code.
	•	Facade Pattern: Provide a unified interface to interact with the complex subsystem.
	•	Scalability: Ensure the system can handle increased load or expanded functionalities seamlessly.

1.2. High-Level Components

	1.	Streaming Service: Handles video/audio capture and streaming via FFmpeg.
	2.	Web Server: Serves web client files and APIs.
	3.	WebSocket Manager: Manages real-time communications with clients.
	4.	GPIO Manager: Interfaces with hardware buttons to control streaming.
	5.	Video Manager: Manages video files, including listing, serving, and archiving.
	6.	Facade Adapter: Provides a simplified interface to interact with all components.

2. Design Patterns and Principles

2.1. Interface Segregation Principle (ISP)

Define small, specific interfaces rather than large, general ones. This ensures that components only need to implement the functionalities they actually use.

2.2. Dependency Inversion Principle (DIP)

Depend on abstractions (interfaces) rather than concrete implementations. This promotes loose coupling and enhances testability.

2.3. Facade Pattern

Create a facade that provides a unified interface to a set of interfaces in a subsystem. This pattern simplifies interactions for the client by hiding the subsystem’s complexity.

3. Project Structure

Organize your project into clearly defined packages, each responsible for a specific functionality. Here’s a proposed directory structure:

multimedia-sys/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── streaming/
│   │   └── streaming.go
│   ├── webserver/
│   │   └── webserver.go
│   ├── websocket/
│   │   └── websocket.go
│   ├── gpio/
│   │   └── gpio.go
│   ├── videomanager/
│   │   └── videomanager.go
│   └── facade/
│       └── facade.go
├── pkg/
│   └── ... (for reusable packages, if any)
├── web_client/
│   ├── index.html
│   ├── styles.css
│   └── app.js
├── Dockerfile
├── go.mod
└── go.sum

4. Implementing the Components

4.1. Defining Interfaces

Start by defining interfaces for each component within their respective packages. This abstraction allows for easy swapping or mocking of implementations.

4.1.1. Streaming Interface (internal/streaming/streaming.go)

package streaming

// Streamer defines the methods required for streaming functionalities
type Streamer interface {
    StartStream() error
    StopStream() error
    IsStreaming() bool
}

4.1.2. WebSocket Interface (internal/websocket/websocket.go)

package websocket

// WebSocketManager defines methods to manage WebSocket connections
type WebSocketManager interface {
    RegisterClient(conn *Connection)
    UnregisterClient(conn *Connection)
    BroadcastMessage(message string)
}

4.1.3. GPIO Interface (internal/gpio/gpio.go)

package gpio

// GPIOManager defines methods to manage GPIO interactions
type GPIOManager interface {
    Init() error
    MonitorButton(callback func())
    Close() error
}

4.1.4. Video Manager Interface (internal/videomanager/videomanager.go)

package videomanager

// VideoManager defines methods to handle video file operations
type VideoManager interface {
    ListVideos() ([]string, error)
    ServeVideo(filename string, w http.ResponseWriter) error
    ArchiveVideo(filename string) error
}

4.2. Implementing Components

Now, implement the interfaces defined above. Here’s an example for each component.

4.2.1. Streaming Implementation (internal/streaming/streaming.go)

package streaming

import (
    "os/exec"
    "sync"
)

type FFmpegStreamer struct {
    cmd    *exec.Cmd
    mutex  sync.Mutex
    status bool
}

func NewFFmpegStreamer() *FFmpegStreamer {
    return &FFmpegStreamer{}
}

func (s *FFmpegStreamer) StartStream() error {
    s.mutex.Lock()
    defer s.mutex.Unlock()

    if s.status {
        return nil // Already streaming
    }

    s.cmd = exec.Command("ffmpeg",
        "-f", "v4l2", "-i", "/dev/video0",
        "-f", "alsa", "-i", "hw:1,0",
        "-c:v", "libx264", "-preset", "veryfast", "-maxrate", "2000k", "-bufsize", "4000k",
        "-pix_fmt", "yuv420p", "-g", "50",
        "-c:a", "aac", "-b:a", "128k", "-ar", "44100",
        "-f", "hls",
        "-hls_time", "4",
        "-hls_list_size", "15",
        "-hls_flags", "delete_segments",
        "/tmp/hls/playlist.m3u8",
    )

    s.cmd.Stdout = nil
    s.cmd.Stderr = nil

    if err := s.cmd.Start(); err != nil {
        return err
    }

    s.status = true
    return nil
}

func (s *FFmpegStreamer) StopStream() error {
    s.mutex.Lock()
    defer s.mutex.Unlock()

    if !s.status || s.cmd == nil {
        return nil // Not streaming
    }

    if err := s.cmd.Process.Kill(); err != nil {
        return err
    }

    s.cmd = nil
    s.status = false
    return nil
}

func (s *FFmpegStreamer) IsStreaming() bool {
    s.mutex.Lock()
    defer s.mutex.Unlock()
    return s.status
}

4.2.2. WebSocket Implementation (internal/websocket/websocket.go)

package websocket

import (
    "log"
    "net/http"
    "sync"

    "github.com/gorilla/websocket"
)

type Connection struct {
    Conn *websocket.Conn
}

type WebSocketManagerImpl struct {
    clients map[*Connection]bool
    mutex   sync.Mutex
}

func NewWebSocketManager() *WebSocketManagerImpl {
    return &WebSocketManagerImpl{
        clients: make(map[*Connection]bool),
    }
}

func (wm *WebSocketManagerImpl) RegisterClient(conn *Connection) {
    wm.mutex.Lock()
    defer wm.mutex.Unlock()
    wm.clients[conn] = true
}

func (wm *WebSocketManagerImpl) UnregisterClient(conn *Connection) {
    wm.mutex.Lock()
    defer wm.mutex.Unlock()
    if _, ok := wm.clients[conn]; ok {
        delete(wm.clients, conn)
        conn.Conn.Close()
    }
}

func (wm *WebSocketManagerImpl) BroadcastMessage(message string) {
    wm.mutex.Lock()
    defer wm.mutex.Unlock()
    for conn := range wm.clients {
        err := conn.Conn.WriteMessage(websocket.TextMessage, []byte(message))
        if err != nil {
            log.Println("WebSocket write error:", err)
            delete(wm.clients, conn)
            conn.Conn.Close()
        }
    }
}

func (wm *WebSocketManagerImpl) HandleWebSocket(w http.ResponseWriter, r *http.Request) {
    upgrader := websocket.Upgrader{
        CheckOrigin: func(r *http.Request) bool { return true },
    }

    ws, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Println("WebSocket upgrade error:", err)
        return
    }

    conn := &Connection{Conn: ws}
    wm.RegisterClient(conn)
    defer wm.UnregisterClient(conn)

    for {
        _, _, err := ws.ReadMessage()
        if err != nil {
            log.Println("WebSocket read error:", err)
            break
        }
    }
}

4.2.3. GPIO Implementation (internal/gpio/gpio.go)

package gpio

import (
    "log"
    "sync"

    "github.com/stianeikeland/go-rpio/v4"
)

type GPIOManagerImpl struct {
    pin        rpio.Pin
    callback   func()
    debounceMs int
    mutex      sync.Mutex
}

func NewGPIOManager(pinNumber int, debounceMs int, callback func()) *GPIOManagerImpl {
    return &GPIOManagerImpl{
        pin:        rpio.Pin(pinNumber),
        callback:   callback,
        debounceMs: debounceMs,
    }
}

func (g *GPIOManagerImpl) Init() error {
    if err := rpio.Open(); err != nil {
        return err
    }
    g.pin.Input()
    g.pin.PullUp()
    return nil
}

func (g *GPIOManagerImpl) MonitorButton() {
    var lastState rpio.State
    var lastDebounceTime int64
    debounceDuration := int64(g.debounceMs) * 1e6 // Convert ms to ns

    for {
        state := g.pin.Read()
        if state != lastState {
            lastDebounceTime = time.Now().UnixNano()
        }

        if time.Now().UnixNano()-lastDebounceTime > debounceDuration {
            if state != lastState {
                lastState = state
                if state == rpio.Low {
                    log.Println("GPIO Button Pressed")
                    g.callback()
                }
            }
        }
        time.Sleep(10 * time.Millisecond)
    }
}

func (g *GPIOManagerImpl) Close() error {
    rpio.Close()
    return nil
}

4.2.4. Video Manager Implementation (internal/videomanager/videomanager.go)

package videomanager

import (
    "errors"
    "net/http"
    "os"
    "path/filepath"
    "strings"
)

type VideoManagerImpl struct {
    storageDir string
}

func NewVideoManager(storageDir string) *VideoManagerImpl {
    return &VideoManagerImpl{
        storageDir: storageDir,
    }
}

func (vm *VideoManagerImpl) ListVideos() ([]string, error) {
    var videos []string
    files, err := os.ReadDir(vm.storageDir)
    if err != nil {
        return nil, err
    }

    for _, file := range files {
        if !file.IsDir() && (strings.HasSuffix(file.Name(), ".mp4") || strings.HasSuffix(file.Name(), ".flv")) {
            videos = append(videos, file.Name())
        }
    }
    return videos, nil
}

func (vm *VideoManagerImpl) ServeVideo(filename string, w http.ResponseWriter) error {
    filePath := filepath.Join(vm.storageDir, filename)
    if _, err := os.Stat(filePath); os.IsNotExist(err) {
        return errors.New("file does not exist")
    }
    http.ServeFile(w, nil, filePath)
    return nil
}

func (vm *VideoManagerImpl) ArchiveVideo(filename string) error {
    // Implement archiving logic, e.g., moving to another directory or cloud storage
    return nil
}

4.2.5. Facade Implementation (internal/facade/facade.go)

The facade provides a simplified interface to interact with all underlying components.

package facade

import (
    "github.com/Cdaprod/multimedia-sys/internal/gpio"
    "github.com/Cdaprod/multimedia-sys/internal/streaming"
    "github.com/Cdaprod/multimedia-sys/internal/websocket"
    "github.com/Cdaprod/multimedia-sys/internal/videomanager"
)

type MultimediaFacade interface {
    StartStream() error
    StopStream() error
    ListVideos() ([]string, error)
    ServeVideo(filename string, w http.ResponseWriter) error
    RegisterWebSocket(w http.ResponseWriter, r *http.Request)
    InitGPIO() error
    MonitorGPIO()
}

type multimediaFacade struct {
    streamer       streaming.Streamer
    wsManager      websocket.WebSocketManager
    videoManager   videomanager.VideoManager
    gpioManager    gpio.GPIOManager
}

func NewMultimediaFacade(streamer streaming.Streamer, wsManager websocket.WebSocketManager, videoManager videomanager.VideoManager, gpioManager gpio.GPIOManager) MultimediaFacade {
    return &multimediaFacade{
        streamer:     streamer,
        wsManager:    wsManager,
        videoManager: videoManager,
        gpioManager:  gpioManager,
    }
}

func (m *multimediaFacade) StartStream() error {
    return m.streamer.StartStream()
}

func (m *multimediaFacade) StopStream() error {
    return m.streamer.StopStream()
}

func (m *multimediaFacade) ListVideos() ([]string, error) {
    return m.videoManager.ListVideos()
}

func (m *multimediaFacade) ServeVideo(filename string, w http.ResponseWriter) error {
    return m.videoManager.ServeVideo(filename, w)
}

func (m *multimediaFacade) RegisterWebSocket(w http.ResponseWriter, r *http.Request) {
    m.wsManager.HandleWebSocket(w, r)
}

func (m *multimediaFacade) InitGPIO() error {
    return m.gpioManager.Init()
}

func (m *multimediaFacade) MonitorGPIO() {
    m.gpioManager.MonitorButton()
}

4.3. Updating the Facade Package

Ensure that the internal/facade/facade.go file correctly references the local modules. Adjust the import paths as necessary based on your project structure.

4.4. Main Application (cmd/server/main.go)

Now, implement the main application that ties everything together via the facade.

package main

import (
    "log"
    "net/http"

    "github.com/Cdaprod/multimedia-sys/internal/facade"
    "github.com/Cdaprod/multimedia-sys/internal/gpio"
    "github.com/Cdaprod/multimedia-sys/internal/streaming"
    "github.com/Cdaprod/multimedia-sys/internal/videomanager"
    "github.com/Cdaprod/multimedia-sys/internal/websocket"
    "github.com/gorilla/mux"
)

func main() {
    // Initialize Components
    streamer := streaming.NewFFmpegStreamer()
    wsManager := websocket.NewWebSocketManager()
    videoManager := videomanager.NewVideoManager("/mnt/nas/videos")
    gpioManager := gpio.NewGPIOManager(18, 500, func() {
        // Callback function when button is pressed
        if streamer.IsStreaming() {
            if err := streamer.StopStream(); err != nil {
                log.Println("Error stopping stream:", err)
            }
        } else {
            if err := streamer.StartStream(); err != nil {
                log.Println("Error starting stream:", err)
            }
        }
    })

    // Create Facade
    mf := facade.NewMultimediaFacade(streamer, wsManager, videoManager, gpioManager)

    // Initialize GPIO
    if err := mf.InitGPIO(); err != nil {
        log.Fatalf("Failed to initialize GPIO: %v", err)
    }
    defer gpioManager.Close()

    // Setup Router
    r := mux.NewRouter()

    // API Endpoints
    r.HandleFunc("/start-stream", func(w http.ResponseWriter, r *http.Request) {
        if err := mf.StartStream(); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        mf.wsManager.BroadcastMessage("Stream started")
        w.WriteHeader(http.StatusOK)
    }).Methods("GET")

    r.HandleFunc("/stop-stream", func(w http.ResponseWriter, r *http.Request) {
        if err := mf.StopStream(); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        mf.wsManager.BroadcastMessage("Stream stopped")
        w.WriteHeader(http.StatusOK)
    }).Methods("GET")

    r.HandleFunc("/list-videos", func(w http.ResponseWriter, r *http.Request) {
        videos, err := mf.ListVideos()
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        jsonResponse, err := json.Marshal(videos)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        w.Write(jsonResponse)
    }).Methods("GET")

    r.HandleFunc("/videos/{filename}", func(w http.ResponseWriter, r *http.Request) {
        vars := mux.Vars(r)
        filename := vars["filename"]
        if err := mf.ServeVideo(filename, w); err != nil {
            http.Error(w, err.Error(), http.StatusNotFound)
            return
        }
    }).Methods("GET")

    r.HandleFunc("/ws", mf.RegisterWebSocket).Methods("GET")

    // Serve HLS and Web Client
    r.PathPrefix("/hls/").Handler(http.StripPrefix("/hls/", http.FileServer(http.Dir("/tmp/hls"))))
    r.PathPrefix("/").Handler(http.FileServer(http.Dir("./web_client")))

    // Start GPIO Monitoring in a separate goroutine
    go mf.MonitorGPIO()

    // Start HTTP Server
    log.Println("Server started on :8080")
    if err := http.ListenAndServe(":8080", r); err != nil {
        log.Fatalf("Server failed: %v", err)
    }
}

Explanation:

	•	Component Initialization: Creates instances of each component and injects them into the facade.
	•	GPIO Callback: Defines a callback function that toggles the stream when the GPIO button is pressed.
	•	Router Setup: Uses gorilla/mux to define API endpoints for starting/stopping streams, listing videos, serving videos, and handling WebSocket connections.
	•	Serve HLS and Web Client: Serves HLS segments and the web client files.
	•	GPIO Monitoring: Starts monitoring the GPIO button in a separate goroutine.
	•	Server Startup: Starts the HTTP server on port 8080.

5. Dockerfile Refinement

Now, update your Dockerfile to accommodate the new modular structure and ensure all dependencies are correctly installed.

# Stage 1: Build the Go application
FROM golang:1.19-alpine AS builder

# Install necessary packages
RUN apk add --no-cache git gcc musl-dev

# Set work directory
WORKDIR /app

# Copy go.mod and go.sum files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy the source code
COPY cmd/server/main.go ./cmd/server/
COPY internal/ ./internal/
COPY web_client/ ./web_client/

# Build the Go app
RUN CGO_ENABLED=1 GOOS=linux go build -o multimedia-sys ./cmd/server/main.go

# Stage 2: Build Nginx with RTMP module
FROM alpine:3.16 AS nginx-builder

# Install dependencies for building Nginx and RTMP module
RUN apk add --no-cache --virtual .build-deps \
    build-base \
    openssl-dev \
    pcre-dev \
    zlib-dev \
    git \
    curl \
    tar

# Clone the nginx-rtmp-module repository
RUN git clone https://github.com/arut/nginx-rtmp-module.git /tmp/nginx-rtmp-module

# Download and extract Nginx source
RUN curl -SL http://nginx.org/download/nginx-1.19.3.tar.gz -o /tmp/nginx-1.19.3.tar.gz \
    && tar -zxvf /tmp/nginx-1.19.3.tar.gz -C /tmp \
    && cd /tmp/nginx-1.19.3 \
    && ./configure --with-http_ssl_module --add-module=/tmp/nginx-rtmp-module \
    && make \
    && make install

# Stage 3: Final image
FROM alpine:3.16

# Install necessary packages: FFmpeg, Nginx, and other tools
RUN apk add --no-cache \
    ffmpeg \
    bash \
    libc6-compat \
    ca-certificates \
    openssl \
    curl \
    tini

# Copy the compiled Nginx from nginx-builder
COPY --from=nginx-builder /usr/local/nginx /usr/local/nginx

# Copy the Go application from builder
COPY --from=builder /app/multimedia-sys /usr/local/bin/multimedia-sys

# Copy Nginx configuration
COPY nginx.conf /usr/local/nginx/conf/nginx.conf

# Copy the web client files
COPY web_client/ /usr/share/nginx/html/

# Create necessary directories for HLS and video storage
RUN mkdir -p /tmp/hls /mnt/nas/videos /run/nginx

# Expose RTMP and HTTP ports
EXPOSE 1935 80 8080

# Use Tini as the entrypoint for better signal handling
ENTRYPOINT ["/sbin/tini", "--"]

# Start Nginx and the Go application using a supervisord-like script
COPY scripts/start.sh /start.sh
RUN chmod +x /start.sh

CMD ["/start.sh"]

5.1. scripts/start.sh

Create a start.sh script to run both Nginx and the Go application.

#!/bin/bash

# Start Nginx
/usr/local/nginx/sbin/nginx

# Start the Go application
exec /usr/local/bin/multimedia-sys

Ensure that start.sh is executable:

chmod +x scripts/start.sh

6. Facade Adapter Interface

The facade acts as a single entry point for interacting with all underlying components. This simplifies client interactions and abstracts the complexities of the subsystem.

6.1. Benefits of Using a Facade

	•	Simplified Interface: Clients interact with a single interface rather than multiple complex ones.
	•	Loose Coupling: Changes in subsystems do not affect the clients.
	•	Easier Maintenance: Enhancements or modifications are confined within the facade.

6.2. Using the Facade

Clients (e.g., web controllers) can now interact with the entire system through the facade without needing to know the details of each component.

Example Usage in API Handlers:

// Start Stream API Handler
r.HandleFunc("/start-stream", func(w http.ResponseWriter, r *http.Request) {
    if err := mf.StartStream(); err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    mf.wsManager.BroadcastMessage("Stream started")
    w.WriteHeader(http.StatusOK)
}).Methods("GET")

This approach ensures that API handlers remain clean and focused on their primary responsibilities, delegating complex operations to the facade.

7. Extensibility and Adding New Components

With this architecture, adding new functionalities becomes straightforward. Here’s how you can extend the system:

7.1. Adding a New Component (e.g., Recording Service)

	1.	Define an Interface:

package recording

type Recorder interface {
    StartRecording() error
    StopRecording() error
    IsRecording() bool
}

	2.	Implement the Interface:

package recording

import (
    "os/exec"
    "sync"
)

type FFmpegRecorder struct {
    cmd    *exec.Cmd
    mutex  sync.Mutex
    status bool
}

func NewFFmpegRecorder() *FFmpegRecorder {
    return &FFmpegRecorder{}
}

func (r *FFmpegRecorder) StartRecording() error {
    r.mutex.Lock()
    defer r.mutex.Unlock()

    if r.status {
        return nil // Already recording
    }

    r.cmd = exec.Command("ffmpeg",
        "-i", "/tmp/hls/playlist.m3u8",
        "-c", "copy",
        "/mnt/nas/videos/recording_$(date +%s).mp4",
    )

    r.cmd.Stdout = nil
    r.cmd.Stderr = nil

    if err := r.cmd.Start(); err != nil {
        return err
    }

    r.status = true
    return nil
}

func (r *FFmpegRecorder) StopRecording() error {
    r.mutex.Lock()
    defer r.mutex.Unlock()

    if !r.status || r.cmd == nil {
        return nil // Not recording
    }

    if err := r.cmd.Process.Kill(); err != nil {
        return err
    }

    r.cmd = nil
    r.status = false
    return nil
}

func (r *FFmpegRecorder) IsRecording() bool {
    r.mutex.Lock()
    defer r.mutex.Unlock()
    return r.status
}

	3.	Integrate with Facade:

package facade

import (
    "github.com/Cdaprod/multimedia-sys/internal/recording"
    // other imports
)

type MultimediaFacade interface {
    // existing methods
    StartRecording() error
    StopRecording() error
}

type multimediaFacade struct {
    // existing fields
    recorder recording.Recorder
}

func NewMultimediaFacade(/* existing params */, recorder recording.Recorder) MultimediaFacade {
    return &multimediaFacade{
        // existing initializations
        recorder: recorder,
    }
}

func (m *multimediaFacade) StartRecording() error {
    return m.recorder.StartRecording()
}

func (m *multimediaFacade) StopRecording() error {
    return m.recorder.StopRecording()
}

	4.	Update Main Application:

func main() {
    // Initialize Components
    streamer := streaming.NewFFmpegStreamer()
    wsManager := websocket.NewWebSocketManager()
    videoManager := videomanager.NewVideoManager("/mnt/nas/videos")
    gpioManager := gpio.NewGPIOManager(18, 500, func() {
        // Callback function when button is pressed
        if streamer.IsStreaming() {
            if err := streamer.StopStream(); err != nil {
                log.Println("Error stopping stream:", err)
            }
        } else {
            if err := streamer.StartStream(); err != nil {
                log.Println("Error starting stream:", err)
            }
        }
    })
    recorder := recording.NewFFmpegRecorder()

    // Create Facade
    mf := facade.NewMultimediaFacade(streamer, wsManager, videoManager, gpioManager, recorder)

    // Rest of the setup
}

This approach ensures that new components can be added with minimal changes to existing code, maintaining the system’s scalability and maintainability.

8. Comprehensive Example: main.go

Combining all the components and the facade, here’s an updated main.go that showcases the modular architecture.

package main

import (
    "encoding/json"
    "log"
    "net/http"

    "github.com/Cdaprod/multimedia-sys/internal/facade"
    "github.com/Cdaprod/multimedia-sys/internal/gpio"
    "github.com/Cdaprod/multimedia-sys/internal/recording"
    "github.com/Cdaprod/multimedia-sys/internal/streaming"
    "github.com/Cdaprod/multimedia-sys/internal/videomanager"
    "github.com/Cdaprod/multimedia-sys/internal/websocket"
    "github.com/gorilla/mux"
)

func main() {
    // Initialize Components
    streamer := streaming.NewFFmpegStreamer()
    wsManager := websocket.NewWebSocketManager()
    videoManager := videomanager.NewVideoManager("/mnt/nas/videos")
    recorder := recording.NewFFmpegRecorder()
    gpioManager := gpio.NewGPIOManager(18, 500, func() {
        // Toggle streaming on button press
        if streamer.IsStreaming() {
            if err := streamer.StopStream(); err != nil {
                log.Println("Error stopping stream:", err)
            }
        } else {
            if err := streamer.StartStream(); err != nil {
                log.Println("Error starting stream:", err)
            }
        }
    })

    // Create Facade
    mf := facade.NewMultimediaFacade(streamer, wsManager, videoManager, gpioManager, recorder)

    // Initialize GPIO
    if err := mf.InitGPIO(); err != nil {
        log.Fatalf("Failed to initialize GPIO: %v", err)
    }
    defer gpioManager.Close()

    // Setup Router
    r := mux.NewRouter()

    // API Endpoints
    r.HandleFunc("/start-stream", func(w http.ResponseWriter, r *http.Request) {
        if err := mf.StartStream(); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        mf.wsManager.BroadcastMessage("Stream started")
        w.WriteHeader(http.StatusOK)
    }).Methods("GET")

    r.HandleFunc("/stop-stream", func(w http.ResponseWriter, r *http.Request) {
        if err := mf.StopStream(); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        mf.wsManager.BroadcastMessage("Stream stopped")
        w.WriteHeader(http.StatusOK)
    }).Methods("GET")

    r.HandleFunc("/start-recording", func(w http.ResponseWriter, r *http.Request) {
        if err := mf.StartRecording(); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        mf.wsManager.BroadcastMessage("Recording started")
        w.WriteHeader(http.StatusOK)
    }).Methods("GET")

    r.HandleFunc("/stop-recording", func(w http.ResponseWriter, r *http.Request) {
        if err := mf.StopRecording(); err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        mf.wsManager.BroadcastMessage("Recording stopped")
        w.WriteHeader(http.StatusOK)
    }).Methods("GET")

    r.HandleFunc("/list-videos", func(w http.ResponseWriter, r *http.Request) {
        videos, err := mf.ListVideos()
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        jsonResponse, err := json.Marshal(videos)
        if err != nil {
            http.Error(w, err.Error(), http.StatusInternalServerError)
            return
        }
        w.Header().Set("Content-Type", "application/json")
        w.Write(jsonResponse)
    }).Methods("GET")

    r.HandleFunc("/videos/{filename}", func(w http.ResponseWriter, r *http.Request) {
        vars := mux.Vars(r)
        filename := vars["filename"]
        if err := mf.ServeVideo(filename, w); err != nil {
            http.Error(w, err.Error(), http.StatusNotFound)
            return
        }
    }).Methods("GET")

    r.HandleFunc("/ws", mf.RegisterWebSocket).Methods("GET")

    // Serve HLS and Web Client
    r.PathPrefix("/hls/").Handler(http.StripPrefix("/hls/", http.FileServer(http.Dir("/tmp/hls"))))
    r.PathPrefix("/").Handler(http.FileServer(http.Dir("./web_client")))

    // Start GPIO Monitoring in a separate goroutine
    go mf.MonitorGPIO()

    // Start HTTP Server
    log.Println("Server started on :8080")
    if err := http.ListenAndServe(":8080", r); err != nil {
        log.Fatalf("Server failed: %v", err)
    }
}

9. Leveraging Generics for Enhanced Flexibility

Go’s generics, introduced in Go 1.18, allow for more flexible and reusable code. While the current system might not heavily rely on generics, they can be beneficial for creating generic utility functions or data structures.

Example: Generic Response Wrapper

package facade

type Response[T any] struct {
    Data  T      `json:"data,omitempty"`
    Error string `json:"error,omitempty"`
}

func NewResponse[T any](data T, err error) Response[T] {
    if err != nil {
        return Response[T]{Error: err.Error()}
    }
    return Response[T]{Data: data}
}

Usage in API Handler:

r.HandleFunc("/list-videos", func(w http.ResponseWriter, r *http.Request) {
    videos, err := mf.ListVideos()
    response := facade.NewResponse(videos, err)
    respondJSON(w, response)
}).Methods("GET")

This generic approach reduces boilerplate and enhances code reusability.

10. Finalizing Docker Configuration

Ensure your Dockerfile and scripts are correctly set up to handle the new architecture.

10.1. Building the Docker Image

From the root of your project (multimedia-sys), build the Docker image:

docker build -t multimedia-sys .

10.2. Running the Docker Container

Run the Docker container with necessary permissions and volume mounts:

docker run --privileged -d \
    -p 80:80 \
    -p 1935:1935 \
    -p 8080:8080 \
    --device /dev/video0:/dev/video0 \
    --device /dev/snd:/dev/snd \
    --mount type=bind,source=/mnt/nas/videos,target=/mnt/nas/videos \
    --name multimedia-sys \
    multimedia-sys

Notes:

	•	--privileged: Grants the container access to GPIO and other hardware interfaces. Use with caution.
	•	--device: Maps the video and audio devices into the container.
	•	--mount: Binds the NAS videos directory into the container for persistent storage.

11. Best Practices and Further Enhancements

11.1. Configuration Management

	•	Environment Variables: Use environment variables for configurable parameters like NAS credentials, RTMP server URLs, etc.
	•	Configuration Files: Implement configuration files (e.g., YAML, JSON) to manage settings more efficiently.

11.2. Logging and Monitoring

	•	Structured Logging: Use structured logging libraries like logrus or zap for better log management.
	•	Monitoring Tools: Integrate monitoring tools like Prometheus and Grafana to monitor system health and performance.

11.3. Error Handling and Recovery

	•	Graceful Shutdown: Implement graceful shutdown procedures to ensure that all processes terminate correctly.
	•	Retries and Backoff: Implement retry mechanisms for transient failures, especially for critical operations like streaming.

11.4. Security Enhancements

	•	Authentication and Authorization: Protect API endpoints with authentication (e.g., JWT tokens).
	•	HTTPS: Secure the web server with SSL/TLS certificates.
	•	Input Validation: Rigorously validate all user inputs to prevent security vulnerabilities.

11.5. Testing

	•	Unit Tests: Write unit tests for each component to ensure functionality.
	•	Integration Tests: Implement integration tests to validate interactions between components.
	•	Mocking Interfaces: Use mocking for interfaces to facilitate testing without relying on actual hardware or external services.

12. Conclusion

By refactoring your system into modular components with well-defined interfaces and using a facade pattern, you create a scalable, maintainable, and extensible multimedia system. This architecture not only simplifies current functionalities but also paves the way for integrating future enhancements seamlessly.

Here’s a summary of the steps you followed:

	1.	Modular Design: Broke down the system into atomic components with specific responsibilities.
	2.	Abstraction: Defined clear interfaces for each component to promote loose coupling.
	3.	Facade Pattern: Implemented a facade to provide a unified interface to the system.
	4.	Dockerization: Configured Docker to build and run the system efficiently.
	5.	Extensibility: Demonstrated how to add new components without disrupting existing functionalities.
	6.	Best Practices: Incorporated design principles, error handling, security measures, and testing strategies.

This comprehensive approach ensures that your Cdaprod/multimedia-sys repository serves as a robust foundation for future Golang package developments and system expansions.

Feel free to reach out if you need further assistance or have additional questions!