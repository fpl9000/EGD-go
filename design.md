# Entropy Gathering Daemon (EGD) - Go Implementation Design

## Overview

This document outlines the design for a Go implementation of the Entropy Gathering Daemon (EGD) that behaves exactly like the existing Python version (`egd.py`). The Go application will be a dual-purpose binary that functions as both a command-line control utility and the daemon process itself.

## Key Requirements

- **Exact functional equivalence** to the Python version
- **Dual-purpose binary**: CLI client + daemon in one executable
- **TOML configuration** format (replacing Python config file)
- **Cross-platform compatibility** (Windows, Linux, macOS)
- **Robust error handling** and logging
- **Efficient entropy collection** and storage

## Architecture Overview

### Core Components

1. **Entropy Pool** - Stores collected entropy in chunks
2. **Entropy Sources** - Configurable sources for gathering entropy
3. **Daemon Process** - Background service that collects entropy
4. **TCP Server** - Handles control commands from CLI
5. **CLI Interface** - Command-line interface for daemon control
6. **Configuration System** - TOML-based configuration management

### Package Structure

```
egd/
├── main.go                 # Entry point and CLI commands
├── go.mod                  # Go module definition
├── go.sum                  # Dependency checksums
├── internal/
│   ├── config/
│   │   ├── config.go       # Configuration loading and validation
│   │   └── types.go        # Configuration type definitions
│   ├── entropy/
│   │   ├── pool.go         # EntropyPool implementation
│   │   ├── chunk.go        # PoolChunk implementation
│   │   ├── source.go       # EntropySource implementation
│   │   └── stirring.go     # Entropy stirring algorithm
│   ├── daemon/
│   │   ├── daemon.go       # Main daemon implementation
│   │   ├── server.go       # TCP server for control commands
│   │   └── lockfile.go     # Process lock file management
│   ├── compress/
│   │   └── compress.go     # Compression/decompression utilities
│   └── util/
│       ├── mutex.go        # Cross-process mutex implementation
│       ├── logging.go      # Logging configuration
│       └── singleton.go    # Singleton pattern helper
└── README.md
```

## Core Component Design

### 1. Entropy Pool (`internal/entropy/pool.go`)

```go
type EntropyPool struct {
    chunks           []*PoolChunk
    entropyByteCount int64
    maxEntropy       int64
    mutex            sync.RWMutex
}

type PoolChunk struct {
    id      int64
    entropy []byte
    maxSize int
}
```

**Key Methods:**
- `AddEntropy(data []byte) int` - Adds entropy to pool, returns bytes added
- `IsFull() bool` - Returns true if pool is at maximum capacity
- `EntropyCount() int64` - Returns current entropy byte count
- `ChunkCount() int` - Returns number of chunks
- `Persist(filename string) error` - Saves pool to disk
- `Load(filename string) error` - Loads pool from disk

### 2. Entropy Source (`internal/entropy/source.go`)

```go
type EntropySource struct {
    config       SourceConfig
    entropy      []byte
    fetched      bool
    compressed   bool
    stirred      bool
    failCount    int
    disabled     bool
    firstFetch   time.Time
    lastFetch    time.Time
    lastSubset   []byte
    client       *http.Client
}

type SourceConfig struct {
    Name        string        `toml:"name"`
    Interval    time.Duration `toml:"interval"`
    URL         string        `toml:"url,omitempty"`
    File        string        `toml:"file,omitempty"`
    Command     []string      `toml:"command,omitempty"`
    Size        int           `toml:"size,omitempty"`
    MinSize     int           `toml:"min_size,omitempty"`
    Scale       float64       `toml:"scale"`
    NoCompress  bool          `toml:"no_compress,omitempty"`
    InitDelay   time.Duration `toml:"init_delay,omitempty"`
    Prefetch    string        `toml:"prefetch,omitempty"`
    Disabled    bool          `toml:"disabled,omitempty"`
}
```

**Key Methods:**
- `Fetch(ctx context.Context) error` - Fetches data from configured source
- `Compress() error` - Compresses fetched data
- `Stir() error` - Applies stirring algorithm
- `GetEntropy() []byte` - Returns processed entropy data
- `IsReady() bool` - Checks if source is ready for next fetch

### 3. Daemon (`internal/daemon/daemon.go`)

```go
type Daemon struct {
    config          *config.Config
    pool            *entropy.EntropyPool
    sources         []*entropy.EntropySource
    server          *TCPServer
    lockFile        string
    lastPersist     time.Time
    stopChan        chan struct{}
    stopped         chan struct{}
    logger          *slog.Logger
}
```

**Key Methods:**
- `Start(ctx context.Context) error` - Starts the daemon process
- `Stop() error` - Gracefully stops the daemon
- `MainLoop(ctx context.Context)` - Main entropy collection loop
- `CreateLockFile() error` - Creates process lock file
- `RemoveLockFile() error` - Removes process lock file

### 4. TCP Server (`internal/daemon/server.go`)

```go
type TCPServer struct {
    daemon   *Daemon
    listener net.Listener
    port     int
    logger   *slog.Logger
}

type CommandResponse struct {
    StatusCode int    `json:"status_code"`
    StatusText string `json:"status_text"`
    Data       []byte `json:"data,omitempty"`
}
```

**Supported Commands:**
- `quit` - Gracefully stop daemon (status 100)
- `status` - Get entropy pool status (status 100)
- `persist` - Force persistence of pool (status 100/300)

### 5. Configuration System (`internal/config/config.go`)

```go
type Config struct {
    // Global configuration parameters
    LogLevel            string        `toml:"log_level"`
    MaxEntropy          int64         `toml:"max_entropy"`
    PersistFile         string        `toml:"persist_file"`
    PersistInterval     time.Duration `toml:"persist_interval"`
    PoolChunkMaxEntropy int           `toml:"pool_chunk_max_entropy"`
    TCPPort             int           `toml:"tcp_port"`
    
    // Entropy sources - each source becomes a direct TOML section
    Sources map[string]SourceConfig
}
```

## TOML Configuration File Format (`egd.toml`)

```toml
# Global EGD Configuration Parameters
log_level = "info"                    # Log level: debug, info, warn, error
max_entropy = 10485760               # Maximum entropy pool size (10MB)
persist_file = "~/.egdentropy"       # File to persist entropy pool
persist_interval = "10m"             # How often to persist pool
pool_chunk_max_entropy = 8192       # Maximum size per entropy chunk
tcp_port = 2121                      # TCP port for daemon control

# Entropy Sources
[dev_random]
name = "1000 bytes from /dev/random"
interval = "5m"
file = "/dev/random"
size = 1000
no_compress = true
scale = 0.8

[random_org]
name = "10,000 random bytes from random.org"
interval = "4h"
url = "https://www.random.org/cgi-bin/randbyte?nbytes=10000&format=h"
scale = 0.9

[random_numbers_info]
name = "1000 random numbers from RandomNumbers.info"
interval = "30m"
command = ["/bin/sh", "-c", "curl -s 'http://www.randomnumbers.info/cgibin/wqrng.cgi?amount=1000&limit=10000' | grep -o ' [0-9]+' | tr -d ' ' | head-1000"]
scale = 0.8

[wikipedia_random]
name = "Random Wikipedia page (without images)"
interval = "40m"
url = "https://en.wikipedia.org/wiki/Special:Random"
scale = 0.2

[wikipedia_images]
name = "5 latest image thumbnails on Wikipedia"
interval = "30m"
url = "TODO: replace getWikipediaRecentImages function"
scale = 0.2

[wikipedia_changes]
name = "Summary of last 50 changes to Wikipedia"
interval = "6h"
url = "https://en.wikipedia.org/w/api.php?hidebots=1&hidecategorization=1&days=7&limit=50&action=feedrecentchanges&feedformat=atom"
scale = 0.05

[apod]
name = "Astronomy Picture of the Day"
interval = "24h"
url = "TODO: replace getApodImageURL function"
size = 150000
scale = 0.1

[npr_audio]
name = "NPR hourly news audio"
interval = "1h40m"
url = "http://public.npr.org/anon.npr-mp3/npr/news/newscast.mp3"
size = 150000
scale = 0.1

[nrl_world_sat]
name = "NRL world visible/IR satellite image"
interval = "3h"
url = "TODO: replace getNRLWorldSatelliteURL function"
min_size = 30000
scale = 0.05

[nist_beacon]
name = "NIST Hardware Random Numbers Beacon"
interval = "5m"
url = "TODO: replace nistBeacon function"
scale = 0.8

[hotbits]
name = "Hotbits Radioactive Decay Random Numbers"
interval = "6h"
init_delay = "1h"
url = "TODO: replace getHotbitsNumbers function"
scale = 0.8

[cnn_rss]
name = "CNN Latest Stories RSS Feed"
interval = "6h"
url = "http://rss.cnn.com/rss/cnn_latest.rss"
scale = 0.05

# Example of a disabled source
[anu_quantum]
name = "ANU Quantum Random Numbers Server"
interval = "6h"
url = "TODO: replace getANUNumbers function"
scale = 0.8
disabled = true
```

## CLI Interface Design

The application will use the Cobra CLI library for command handling:

```bash
# Start the daemon
egd start [--force]

# Stop the daemon
egd stop

# Get daemon status
egd status

# Persist entropy pool
egd persist

# Get entropy (future enhancement)
egd getentropy <bytes>

# List configured entropy sources
egd sources

# Validate configuration
egd config validate

# Show configuration
egd config show
```

## Concurrency Model

### Goroutine Usage

1. **Main Daemon Loop** - Single goroutine collecting from all sources
2. **TCP Server** - Goroutine per connection for handling commands
3. **HTTP Fetches** - Individual goroutines with context timeouts
4. **Entropy Source Processing** - Pipeline pattern for fetch/compress/stir

### Synchronization

- **Entropy Pool** - RWMutex for concurrent access
- **Configuration** - Read-only after initialization
- **Daemon State** - Channels for shutdown coordination
- **File Operations** - File locking for persistence

## Key Dependencies

```go
// Required external dependencies
github.com/BurntSushi/toml          // TOML configuration parsing
github.com/spf13/cobra              // CLI framework
github.com/spf13/viper              // Configuration management
github.com/pierrec/lz4              // LZ4 compression (or compress/gzip)
golang.org/x/sys                    // System-specific operations
```

## Implementation Details

### Entropy Stirring Algorithm

Implements the exact same algorithm as Python version:
- Process entropy in 32-byte subblocks
- Compute SHA-256 hash over sliding 1024-byte window
- XOR each subblock with corresponding hash bytes

### Error Handling

- **Source Failures** - Track failure count, disable after 5 failures
- **Network Timeouts** - 30-second timeout for HTTP operations
- **File Operations** - Detailed error reporting with file paths
- **Daemon Crashes** - Graceful recovery with entropy pool persistence

### Platform Considerations

- **Lock Files** - Platform-specific paths (`/tmp/egd.lck` on Unix, temp dir on Windows)
- **Signals** - SIGTERM/SIGINT handling for graceful shutdown
- **File Paths** - Cross-platform path handling using `filepath` package
- **Process Management** - Platform-specific daemon backgrounding

### Logging

- **Structured Logging** - Using `log/slog` package
- **Log Levels** - DEBUG, INFO, WARN, ERROR
- **Log Rotation** - External log rotation (logrotate on Linux)
- **Format** - JSON format for structured log analysis

### Security Considerations

- **Lock File Permissions** - 0600 (owner read/write only)
- **Persist File Permissions** - 0600 (owner read/write only)
- **Network Binding** - localhost only for TCP server
- **Input Validation** - Strict validation of configuration parameters

## Build and Deployment

### Build Configuration

```makefile
# Build targets
build:
    go build -o egd main.go

build-release:
    CGO_ENABLED=0 go build -ldflags="-s -w" -o egd main.go

install:
    go install

test:
    go test ./...

lint:
    golangci-lint run
```

### Installation

1. **Binary Installation** - Single executable deployment
2. **Configuration** - Default config file in `$HOME/.egd.toml`
3. **Service Integration** - systemd service file for Linux
4. **Windows Service** - Optional Windows service wrapper

## Migration from Python Version

### Configuration Migration

- **Conversion Tool** - Utility to convert `.egdconf` to `egd.toml`
- **Validation** - Ensure all entropy sources are correctly converted
- **Testing** - Side-by-side testing with Python version

### Data Compatibility

- **Entropy Pool Format** - New format (not compatible with Python pickle)
- **Migration Script** - Extract entropy from Python pool for Go version
- **Fresh Start** - Recommendation to start with empty pool

## Testing Strategy

### Unit Tests

- **Entropy Pool** - Test chunk management and persistence
- **Entropy Sources** - Mock HTTP responses and file operations
- **Configuration** - Test TOML parsing and validation
- **Stirring Algorithm** - Test against known vectors

### Integration Tests

- **End-to-End** - Start daemon, add entropy, query status, stop
- **Network Sources** - Test with mock HTTP servers
- **File Sources** - Test with temporary files
- **Error Conditions** - Test failure handling and recovery

### Performance Tests

- **Memory Usage** - Monitor entropy pool memory consumption
- **CPU Usage** - Profile stirring algorithm performance
- **Network Efficiency** - Test concurrent HTTP operations
- **Persistence Speed** - Measure pool save/load times

## Documentation Strategy

### Required Documentation

1. **README.md** - Primary project documentation
   - Project overview and purpose
   - Installation instructions for multiple platforms
   - Quick start guide with basic usage examples
   - Configuration file format documentation
   - Building from source instructions
   - Troubleshooting common issues

2. **User Manual** - Comprehensive usage guide
   - Detailed CLI command reference
   - Configuration parameter documentation
   - Entropy source configuration examples
   - Daemon management and monitoring
   - Migration guide from Python version
   - Best practices and security considerations

3. **API Documentation** - Technical reference
   - Go package documentation using `godoc`
   - Internal API documentation for developers
   - Configuration file schema reference
   - TCP command protocol specification

4. **Deployment Guides** - Platform-specific instructions
   - Linux systemd service setup
   - Windows service installation
   - macOS launchd configuration
   - Docker deployment guide
   - Security hardening recommendations

### Documentation Standards

- **Format**: Markdown for all user-facing documentation
- **Code Examples**: Include working examples for all features
- **Cross-References**: Link between related documentation sections
- **Versioning**: Document version compatibility and breaking changes
- **Screenshots**: Include CLI output examples where helpful

### Documentation Deliverables

1. `README.md` - Project root documentation
2. `docs/user-guide.md` - Comprehensive user manual
3. `docs/configuration.md` - Configuration reference
4. `docs/deployment/` - Platform-specific deployment guides
5. `docs/migration.md` - Python to Go migration guide
6. `docs/troubleshooting.md` - Common issues and solutions
7. `CHANGELOG.md` - Version history and breaking changes

## Future Enhancements

1. **Web Dashboard** - HTTP interface for monitoring
2. **Metrics Export** - Prometheus metrics endpoint
3. **Plugin System** - Dynamic entropy source loading
4. **Distributed Mode** - Multiple daemons sharing entropy
5. **Hardware Integration** - TRNG device support

---

This design document provides the complete blueprint for implementing the Go version of EGD while maintaining exact functional compatibility with the Python version.