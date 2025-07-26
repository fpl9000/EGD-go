# Entropy Gathering Daemon (EGD) - Go Implementation Design

## Table of Contents

- [Overview](#overview)
- [Key Requirements](#key-requirements)
- [Architecture Overview](#architecture-overview)
  - [Core Components](#core-components)
  - [Package Structure](#package-structure)
  - [Key Dependencies](#key-dependencies)
- [Core Component Design](#core-component-design)
  - [Configuration Package (`internal/config/`)](#configuration-package-internalconfig)
  - [Entropy Package (`internal/entropy/`)](#entropy-package-internalentropy)
  - [Daemon Package (`internal/daemon/`)](#daemon-package-internaldaemon)
  - [Compression Package (`internal/compress/`)](#compression-package-internalcompress)
  - [Utilities Package (`internal/util/`)](#utilities-package-internalutil)
- [TOML Configuration File Format (`egd.toml`)](#toml-configuration-file-format-egdtoml)
  - [Global Configuration Parameters](#global-configuration-parameters)
  - [Entropy Source Configuration](#entropy-source-configuration)
  - [Example Configuration](#example-configuration)
- [Embedded Scripting System](#embedded-scripting-system)
  - [Script Execution Environment](#script-execution-environment)
- [CLI Interface Design](#cli-interface-design)
- [Concurrency Model](#concurrency-model)
  - [Goroutine Usage](#goroutine-usage)
  - [Synchronization](#synchronization)
- [Implementation Details](#implementation-details)
  - [Entropy Stirring Algorithm](#entropy-stirring-algorithm)
  - [Error Handling](#error-handling)
  - [Platform Considerations](#platform-considerations)
  - [Logging](#logging)
  - [Security Considerations](#security-considerations)
- [Build and Deployment](#build-and-deployment)
  - [Build Configuration](#build-configuration)
  - [Installation](#installation)
- [Migration from Python Version](#migration-from-python-version)
  - [Configuration Migration](#configuration-migration)
  - [Data Compatibility](#data-compatibility)
- [Testing Strategy](#testing-strategy)
  - [Unit Tests](#unit-tests)
  - [Integration Tests](#integration-tests)
  - [Performance Tests](#performance-tests)
- [Documentation Strategy](#documentation-strategy)
  - [Required Documentation](#required-documentation)
  - [Documentation Standards](#documentation-standards)
  - [Documentation Deliverables](#documentation-deliverables)
- [Future Enhancements](#future-enhancements)

## Overview

This document outlines the design for a Go implementation of the Entropy Gathering Daemon (EGD) that behaves exactly like the existing Python version (see file `egd.py`). The Go application will be a dual-purpose binary that functions as both a command-line control utility and the daemon process itself.

## Key Requirements

- **Exact functional equivalence** to the Python version
- **Dual-purpose binary**: CLI client + daemon in one executable
- **TOML configuration** format (replacing Python config file)
- **Cross-platform compatibility** (Windows, Linux, macOS)
- **Robust error handling** and logging
- **Efficient entropy collection** and storage

## Architecture Overview

The EGD Go implementation is organized as a single Go module with distinct packages under the `internal/` directory. Each directory corresponds to a separate Go package that encapsulates related functionality.

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
├── internal/               # Internal packages
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

**Go Package Structure:**
- `main` - Entry point and CLI commands (`main.go`)
- `config` - Configuration parsing and validation (`internal/config/`)
- `entropy` - Entropy pool, sources, and stirring (`internal/entropy/`)
- `daemon` - Main daemon process and TCP server (`internal/daemon/`)
- `compress` - Data compression utilities (`internal/compress/`)
- `util` - Cross-cutting utilities and helpers (`internal/util/`)

### Key Dependencies

```go
// Standard library dependencies
log/slog                            // Structured logging (Go 1.21+)

// Required external dependencies
github.com/BurntSushi/toml          // TOML configuration parsing
github.com/spf13/cobra              // CLI framework
github.com/spf13/viper              // Configuration management
github.com/pierrec/lz4              // LZ4 compression (or compress/gzip)
golang.org/x/sys                    // System-specific operations
```

## Core Component Design

### Configuration Package (`internal/config/`)

The configuration package handles TOML configuration parsing, validation, and type definitions for the entire application.

#### `config.go` - Configuration Loading and Validation

```go
package config

import (
    "time"
)

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

**Key Methods:**
- `LoadConfig(filename string) (*Config, error)` - Parses TOML configuration file
- `(c *Config) ValidateConfig() error` - Validates configuration parameters and source definitions
- `GetDefaultConfig() *Config` - Returns configuration with default values
- `(c *Config) ExpandPaths() error` - Expands tilde and environment variables in file paths

**Global Configuration Validation Rules:**
- `log_level`: Must be one of "debug", "info", "warn", "error" (case-insensitive)
- `max_entropy`: Must be positive integer, recommended minimum 1MB (1048576 bytes)
- `persist_file`: Must be valid file path, parent directory must exist and be writable
- `persist_interval`: Must be valid duration string, minimum "10s", maximum "24h"
- `pool_chunk_max_entropy`: Must be positive integer, recommended range 1KB-64KB
- `tcp_port`: Must be valid port number (1024-65535 for non-root users, 1-65535 for root)

**Source Validation Dependencies:**
- At least one entropy source must be defined and enabled
- Source names must be unique and non-empty
- Each source must have exactly one data source method (URL, File, Command, or Script)

#### `types.go` - Configuration Type Definitions

```go
package config

import (
    "time"
)

type SourceConfig struct {
    Name                string        `toml:"name"`
    Interval            time.Duration `toml:"interval"`
    Scale               float64       `toml:"scale"`
    
    // Data source methods (mutually exclusive)
    URL                 string        `toml:"url,omitempty"`
    File                string        `toml:"file,omitempty"`
    Command             []string      `toml:"command,omitempty"`
    ScriptInterpreter   string        `toml:"script_interpreter,omitempty"`
    Script              string        `toml:"script,omitempty"`
    
    // Common options
    Size                int           `toml:"size,omitempty"`
    MinSize             int           `toml:"min_size,omitempty"`
    NoCompress          bool          `toml:"no_compress,omitempty"`
    InitDelay           time.Duration `toml:"init_delay,omitempty"`
    Prefetch            string        `toml:"prefetch,omitempty"`
    Disabled            bool          `toml:"disabled,omitempty"`
    
    // Custom fields accessible to scripts via environment variables
    CustomFields        map[string]interface{} `toml:",inline"`
}

type LogLevel int

const (
    LogLevelDebug LogLevel = iota
    LogLevelInfo
    LogLevelWarn
    LogLevelError
)

type SourceType int

const (
    SourceTypeURL SourceType = iota
    SourceTypeFile
    SourceTypeCommand
    SourceTypeScript
)
```

**Key Methods:**
- `(sc *SourceConfig) ValidateSourceConfig() error` - Validates individual source configuration
- `(sc *SourceConfig) GetSourceType() SourceType` - Determines source type (URL/File/Command/Script)
- `(sc *SourceConfig) ToEnvironmentVars() map[string]string` - Converts config to environment variables for scripts

**Source Configuration Validation Rules:**

**Required Fields:**
- `name`: Must be non-empty string, maximum 100 characters
- `interval`: Must be valid duration string, minimum "10s", maximum "168h" (7 days)
- `scale`: Must be float between 0.0 and 1.0 (inclusive)

**Data Source Method (Exactly One Required):**
- `url`: Must be valid HTTP/HTTPS URL, maximum 500 characters
- `file`: Must be valid file path, file must exist and be readable
- `command`: Must be non-empty array, first element must be executable command
- `script_interpreter` + `script`: Both required together, interpreter must be valid command

**Optional Field Validation:**
- `size`: If specified, must be positive integer, maximum 100MB (104857600 bytes)
- `min_size`: If specified, must be positive integer, must be ≤ `size` if both specified
- `no_compress`: Boolean, defaults to false
- `init_delay`: If specified, must be valid duration string, maximum "24h"
- `prefetch`: If specified, must be valid HTTP/HTTPS URL, only valid with `url` sources
- `disabled`: Boolean, defaults to false

**Cross-Field Validation Rules:**
- If `script_interpreter` is specified, `script` must also be specified
- If `script` is specified, `script_interpreter` must also be specified
- `prefetch` can only be used with `url` sources
- `size` and `min_size` are ignored for `command` and `script` sources
- Custom fields (for scripts) must have valid environment variable names (alphanumeric + underscore)

**Security Validation:**
- `script_interpreter`: Must be in allowed list or full path to verified executable
- `script`: Must not exceed 64KB in size
- `command`: Arguments must not contain null bytes or control characters
- `file`: Must not be a device file, must be regular file or named pipe
- `url`: Must use HTTPS for external hosts, HTTP only allowed for localhost

### Entropy Package (`internal/entropy/`)

The entropy package contains core logic for entropy pool management, source handling, and the cryptographic stirring algorithm.

#### `pool.go` - Entropy Pool Management

```go
package entropy

import (
    "log/slog"
    "sync"
)

type EntropyPool struct {
    chunks           []*PoolChunk
    entropyByteCount int64
    maxEntropy       int64
    mutex            sync.RWMutex
    logger           *slog.Logger
}
```

**Key Methods:**
- `NewEntropyPool(maxEntropy int64, chunkSize int) *EntropyPool` - Creates new entropy pool
- `(p *EntropyPool) AddEntropy(data []byte) int` - Adds entropy to pool, returns bytes added
- `(p *EntropyPool) IsFull() bool` - Returns true if pool is at maximum capacity
- `(p *EntropyPool) EntropyCount() int64` - Returns current entropy byte count
- `(p *EntropyPool) ChunkCount() int` - Returns number of chunks
- `(p *EntropyPool) Persist(filename string) error` - Saves pool to disk with atomic write
- `(p *EntropyPool) Load(filename string) error` - Loads pool from disk
- `(p *EntropyPool) GetEntropy(bytes int) []byte` - Extracts entropy from pool (future feature)

**Persistence Format Specification:**

**Binary File Format (Little Endian):**
```
Header (32 bytes):
- Magic Number: 4 bytes (0x45474400 = "EGD\0")
- Format Version: 4 bytes (uint32, current = 1)
- Pool Max Entropy: 8 bytes (int64)
- Chunk Max Size: 4 bytes (int32)
- Chunk Count: 4 bytes (uint32)
- Created Timestamp: 8 bytes (int64, Unix nanoseconds)

Per-Chunk Data:
- Chunk ID: 8 bytes (int64)
- Chunk Size: 4 bytes (uint32)
- Entropy Data: variable length (Chunk Size bytes)

Footer (32 bytes):
- Total Entropy Bytes: 8 bytes (int64)
- File Checksum: 8 bytes (CRC64-ISO)
- Magic Number: 4 bytes (0x45474400 = "EGD\0")
- Reserved: 12 bytes (zero-filled for future use)
```

**Atomic Write Implementation:**
```go
func (p *EntropyPool) Persist(filename string) error {
    // Write to temporary file first
    tempFile := filename + ".tmp." + strconv.FormatInt(time.Now().UnixNano(), 36)
    
    // Create temp file with restrictive permissions (0600)
    f, err := os.OpenFile(tempFile, os.O_CREATE|os.O_WRONLY|os.O_EXCL, 0600)
    if err != nil {
        return err
    }
    defer f.Close()
    
    // Write data with checksum calculation
    if err := p.writePoolData(f); err != nil {
        os.Remove(tempFile)
        return err
    }
    
    // Ensure data is written to disk
    if err := f.Sync(); err != nil {
        os.Remove(tempFile)
        return err
    }
    f.Close()
    
    // Atomic rename (platform-specific implementation)
    return os.Rename(tempFile, filename)
}
```

**Corruption Detection and Recovery:**
- **Header Validation**: Check magic numbers and version compatibility
- **Checksum Verification**: Verify CRC64 of entire file content
- **Size Consistency**: Verify chunk sizes match declared lengths
- **Range Validation**: Ensure all values are within reasonable bounds

**Backward Compatibility Strategy:**
- Version field allows format evolution
- Unknown versions are rejected with clear error message
- Future versions may support format conversion
- Recommend backing up pool files before daemon upgrades

**Error Handling:**
- **Disk Full**: Return specific error, do not leave partial files
- **Permission Denied**: Clear error message with file path
- **Corruption Detected**: Log detailed corruption information, refuse to load
- **Version Mismatch**: Specific error message indicating upgrade/downgrade needed

#### `chunk.go` - Pool Chunk Implementation

```go
package entropy

import (
    "sync"
    "time"
)

type PoolChunk struct {
    id          int64
    entropy     []byte
    maxSize     int
    createdAt   time.Time
    mutex       sync.RWMutex
}
```

**Key Methods:**
- `NewPoolChunk(id int64, maxSize int) *PoolChunk` - Creates new entropy chunk
- `(c *PoolChunk) AddData(data []byte) int` - Adds data to chunk, returns bytes added
- `(c *PoolChunk) IsFull() bool` - Returns true if chunk is at capacity
- `(c *PoolChunk) Size() int` - Returns current chunk size
- `(c *PoolChunk) Data() []byte` - Returns copy of chunk data
- `(c *PoolChunk) Serialize() ([]byte, error)` - Serializes chunk for persistence
- `(c *PoolChunk) Deserialize(data []byte) error` - Deserializes chunk from persistence

#### `source.go` - Entropy Source Implementation

```go
package entropy

import (
    "context"
    "log/slog"
    "net/http"
    "sync"
    "time"
    
    "egd/internal/config"
)

type EntropySource struct {
    config         config.SourceConfig
    entropy        []byte
    fetched        bool
    compressed     bool
    stirred        bool
    failCount      int
    disabled       bool
    firstFetch     time.Time
    lastFetch      time.Time
    lastSubset     []byte
    client         *http.Client
    mutex          sync.RWMutex
    logger         *slog.Logger
}
```

**Key Methods:**
- `NewEntropySource(config config.SourceConfig) *EntropySource` - Creates new entropy source
- `(s *EntropySource) Fetch(ctx context.Context) error` - Fetches data from configured source (URL, file, command, or script)
- `(s *EntropySource) FetchURL(ctx context.Context) error` - Fetches data from HTTP URL
- `(s *EntropySource) FetchFile() error` - Reads data from local file
- `(s *EntropySource) ExecuteCommand(ctx context.Context) error` - Runs command and captures output
- `(s *EntropySource) ExecuteScript(ctx context.Context) error` - Executes embedded script with environment variables
- `(s *EntropySource) Compress() error` - Compresses fetched data using LZ4
- `(s *EntropySource) Stir() error` - Applies cryptographic stirring algorithm
- `(s *EntropySource) GetEntropy() []byte` - Returns processed entropy data
- `(s *EntropySource) IsReady() bool` - Checks if source is ready for next fetch based on interval
- `(s *EntropySource) ShouldDisable() bool` - Checks if source should be disabled due to repeated failures

**Script Execution Security Boundaries:**

**Resource Limits:**
```go
type ScriptLimits struct {
    MaxExecutionTime    time.Duration // 30 seconds default
    MaxMemoryMB         int           // 50MB default
    MaxOutputSize       int           // 10MB default
    MaxFileDescriptors  int           // 64 default
    MaxCPUPercent       int           // 50% default (per core)
}
```

**Execution Environment:**
- **Working Directory**: Set to secure temporary directory, not user's home or daemon directory
- **Environment Variables**: Only EGD_SOURCE_* variables and minimal system variables (PATH, HOME, TEMP)
- **Process User**: Run with same user as daemon (no privilege escalation)
- **Network Access**: Unrestricted (scripts may need to fetch external data)
- **File System Access**: Limited to working directory and system temp directories

**Security Restrictions:**
- **Timeout Enforcement**: Hard termination after MaxExecutionTime using context.WithTimeout
- **Memory Monitoring**: Track memory usage via process monitoring (implementation-specific)
- **Output Size Limit**: Terminate script if stdout exceeds MaxOutputSize
- **Signal Handling**: Scripts cannot ignore SIGTERM/SIGKILL
- **Process Isolation**: Use process groups to ensure child processes are also terminated

**Error Handling:**
- **Timeout**: Treat as temporary failure, retry with exponential backoff
- **Memory Limit**: Treat as permanent failure, disable source after 3 consecutive failures
- **Output Too Large**: Treat as temporary failure, may indicate network issues
- **Exit Code != 0**: Treat as temporary failure unless exit code indicates permanent error (127 = command not found)

**Implementation Requirements:**
```go
// Script execution with security boundaries
func (s *EntropySource) ExecuteScript(ctx context.Context) error {
    // Create secure execution context
    scriptCtx, cancel := context.WithTimeout(ctx, s.limits.MaxExecutionTime)
    defer cancel()
    
    // Set up secure working directory
    workDir, err := s.createSecureWorkDir()
    if err != nil {
        return err
    }
    defer os.RemoveAll(workDir)
    
    // Prepare environment variables
    env := s.buildSecureEnvironment()
    
    // Execute with limits monitoring
    return s.executeWithLimits(scriptCtx, workDir, env)
}
```

**Allowed Script Interpreters:**
- `bash`, `sh` - Shell scripting
- `python3`, `python` - Python scripts
- `powershell` - Windows PowerShell (Windows only)
- `perl` - Perl scripts
- `ruby` - Ruby scripts
- Custom interpreters can be added via configuration validation

#### `stirring.go` - Entropy Stirring Algorithm

```go
package entropy

type Stirrer struct {
    windowSize int
    blockSize  int
}
```

**Key Methods:**
- `NewStirrer() *Stirrer` - Creates stirrer with default parameters (1024-byte window, 32-byte blocks)
- `(s *Stirrer) StirData(data []byte) []byte` - Applies stirring algorithm to entropy data
- `(s *Stirrer) processBlock(block []byte, hash []byte) []byte` - Processes single 32-byte block
- `(s *Stirrer) computeWindowHash(window []byte) []byte` - Computes SHA-256 hash over sliding window

### Daemon Package (`internal/daemon/`)

The daemon package implements the main daemon process, TCP control server, and process management utilities.

#### `daemon.go` - Main Daemon Implementation

```go
package daemon

import (
    "context"
    "log/slog"
    "os"
    "os/signal"
    "runtime"
    "sync"
    "syscall"
    "time"
    
    "egd/internal/config"
    "egd/internal/entropy"
)

type Daemon struct {
    config          *config.Config
    pool            *entropy.EntropyPool
    sources         []*entropy.EntropySource
    server          *TCPServer
    lockFile        string
    lastPersist     time.Time
    stopChan        chan struct{}
    stopped         chan struct{}
    mutex           sync.RWMutex
    logger          *slog.Logger
}
```

**Key Methods:**
- `NewDaemon(config *config.Config) *Daemon` - Creates new daemon instance
- `(d *Daemon) Start(ctx context.Context) error` - Starts the daemon process with initialization
- `(d *Daemon) Stop() error` - Gracefully stops the daemon and cleans up resources
- `(d *Daemon) MainLoop(ctx context.Context)` - Main entropy collection loop
- `(d *Daemon) collectFromSources(ctx context.Context)` - Collects entropy from all enabled sources
- `(d *Daemon) shouldPersist() bool` - Checks if pool should be persisted based on interval
- `(d *Daemon) persistPool() error` - Saves entropy pool to disk
- `(d *Daemon) setupGracefulShutdown()` - Sets up cross-platform signal handling for graceful shutdown

#### `server.go` - TCP Control Server

```go
package daemon

import (
    "log/slog"
    "net"
    "sync"
)

type TCPServer struct {
    daemon     *Daemon
    listener   net.Listener
    port       int
    clients    map[net.Conn]bool
    mutex      sync.RWMutex
    logger     *slog.Logger
}

type CommandResponse struct {
    StatusCode int    `json:"status_code"`
    StatusText string `json:"status_text"`
    Data       []byte `json:"data,omitempty"`
}

type Command struct {
    Name string            `json:"command"`
    Args map[string]string `json:"args,omitempty"`
}

// Protocol error codes
const (
    StatusOK                  = 200
    StatusBadRequest         = 400
    StatusCommandNotFound    = 404
    StatusInternalError      = 500
    StatusServiceUnavailable = 503
)
```

**TCP Protocol Specification:**

The daemon listens on localhost only (127.0.0.1) for security. Each connection handles a single command and is automatically closed after the response.

**Message Format:**
- **Request**: Single JSON line terminated with `\n`
- **Response**: Single JSON line terminated with `\n`
- **Encoding**: UTF-8
- **Connection**: One command per connection, server closes after response

**Request Format:**
```json
{"command": "status", "args": {}}
{"command": "persist", "args": {}}
{"command": "quit", "args": {}}
```

**Response Format:**
```json
{"status_code": 200, "status_text": "OK", "data": <base64_encoded_data>}
{"status_code": 400, "status_text": "Bad Request", "data": null}
```

**Supported Commands:**

1. **`status`** - Returns entropy pool status
   - Args: None
   - Response data: JSON object with pool statistics
   ```json
   {
     "entropy_bytes": 1048576,
     "max_entropy": 10485760,
     "chunk_count": 128,
     "is_full": false,
     "last_persist": "2024-01-15T10:30:00Z"
   }
   ```

2. **`persist`** - Forces immediate pool persistence
   - Args: None
   - Response data: JSON object with persistence result
   ```json
   {
     "bytes_written": 1048576,
     "file_path": "/home/user/.egdentropy",
     "persist_time": "2024-01-15T10:30:00Z"
   }
   ```

3. **`quit`** - Gracefully stops the daemon
   - Args: None
   - Response data: JSON object with shutdown status
   ```json
   {
     "message": "Daemon shutting down gracefully",
     "uptime_seconds": 86400
   }
   ```

**Error Responses:**
- **400 Bad Request**: Malformed JSON or invalid command
- **404 Command Not Found**: Unknown command name
- **500 Internal Error**: Server error during command execution
- **503 Service Unavailable**: Daemon is shutting down

**Security Considerations:**
- Server binds to localhost (127.0.0.1) only
- No authentication required (localhost access implies local user)
- Commands are read-only except for `persist` and `quit`
- No sensitive data exposed in responses
- Connection timeout: 30 seconds
- Maximum request size: 1KB

**Key Methods:**
- `NewTCPServer(daemon *Daemon, port int) *TCPServer` - Creates new TCP server
- `(t *TCPServer) Start() error` - Starts listening on configured port
- `(t *TCPServer) Stop() error` - Stops server and closes all connections
- `(t *TCPServer) handleConnection(conn net.Conn)` - Handles individual client connections
- `(t *TCPServer) processCommand(cmd Command) CommandResponse` - Processes control commands
- `(t *TCPServer) handleQuit()` - Gracefully stops daemon
- `(t *TCPServer) handleStatus()` - Returns entropy pool status
- `(t *TCPServer) handlePersist()` - Forces entropy pool persistence

#### `lockfile.go` - Process Lock File Management

```go
package daemon

import (
    "os"
    "sync"
)

type LockFile struct {
    path     string
    file     *os.File
    acquired bool
    mutex    sync.Mutex
}
```

**Key Methods:**
- `NewLockFile(path string) *LockFile` - Creates new lock file manager
- `(lf *LockFile) Acquire() error` - Acquires exclusive lock, prevents multiple daemon instances
- `(lf *LockFile) Release() error` - Releases lock and removes lock file
- `(lf *LockFile) IsLocked() bool` - Checks if lock is currently held
- `(lf *LockFile) GetPID() (int, error)` - Gets PID from existing lock file

### Compression Package (`internal/compress/`)

The compression package provides data compression utilities for entropy sources.

#### `compress.go` - Compression/Decompression Utilities

```go
package compress

import (
    "time"
)

type Compressor struct {
    algorithm string
    level     int
}

type CompressionStats struct {
    OriginalSize   int
    CompressedSize int
    CompressionTime time.Duration
    Ratio          float64
}
```

**Key Methods:**
- `NewCompressor(algorithm string, level int) *Compressor` - Creates compressor (LZ4, GZIP, LZMA)
- `(c *Compressor) Compress(data []byte) ([]byte, error)` - Compresses data using configured algorithm
- `(c *Compressor) Decompress(data []byte) ([]byte, error)` - Decompresses data
- `GetStats(original, compressed []byte) CompressionStats` - Returns compression statistics
- `ShouldCompress(data []byte) bool` - Heuristic to determine if compression is beneficial
- `DetectFormat(data []byte) string` - Detects compression format of data

### Utilities Package (`internal/util/`)

The utilities package contains cross-cutting concerns like logging, synchronization, and helper functions.

#### `mutex.go` - Cross-Process Mutex Implementation

```go
package util

import (
    "os"
    "sync"
)

type CrossProcessMutex struct {
    name     string
    lockFile string
    file     *os.File
    locked   bool
    mutex    sync.Mutex
}
```

**Key Methods:**
- `NewCrossProcessMutex(name string) *CrossProcessMutex` - Creates cross-process mutex
- `(m *CrossProcessMutex) Lock() error` - Acquires cross-process lock using file locking
- `(m *CrossProcessMutex) Unlock() error` - Releases cross-process lock
- `(m *CrossProcessMutex) TryLock() bool` - Attempts to acquire lock without blocking

#### `logging.go` - Logging Configuration

```go
package util

import (
    "io"
    "log/slog"
)

type LogConfig struct {
    Level      slog.Level
    Format     string  // "json" or "text"
    Output     io.Writer
    AddSource  bool
    TimeFormat string
}
```

**Key Methods:**
- `SetupLogging(config LogConfig) *slog.Logger` - Configures structured logging
- `NewFileLogger(filename string, level slog.Level) *slog.Logger` - Creates file-based logger
- `NewConsoleLogger(level slog.Level) *slog.Logger` - Creates console logger
- `ParseLogLevel(level string) slog.Level` - Parses log level string

#### `singleton.go` - Singleton Pattern Helper

```go
package util

import (
    "sync"
)

type Singleton struct {
    instance interface{}
    once     sync.Once
    mutex    sync.RWMutex
}
```

**Key Methods:**
- `NewSingleton() *Singleton` - Creates singleton container
- `(s *Singleton) GetOrCreate(factory func() interface{}) interface{}` - Gets existing or creates new instance
- `(s *Singleton) Get() interface{}` - Gets current instance (may be nil)
- `(s *Singleton) Reset()` - Resets singleton for testing

## TOML Configuration File Format (`egd.toml`)

### Global Configuration Parameters

The configuration file begins with global parameters that control the overall behavior of the EGD daemon:

- **`log_level`** (string): Sets the logging verbosity. Valid values: `"debug"`, `"info"`, `"warn"`, `"error"`. Default: `"info"`.
- **`max_entropy`** (integer): Maximum size of the entropy pool in bytes. Default: `10485760` (10MB).
- **`persist_file`** (string): File path where the entropy pool is periodically saved. Supports tilde expansion (`~`). Default: `"~/.egdentropy"`.
- **`persist_interval`** (duration string): How often to save the entropy pool to disk. Format: `"5m"`, `"1h30m"`, `"24h"`. Default: `"10m"`.
- **`pool_chunk_max_entropy`** (integer): Maximum number of bytes each entropy chunk can contain. Default: `8192`.
- **`tcp_port`** (integer): TCP port number for daemon control commands. Default: `2121`.

### Entropy Source Configuration

Each entropy source is defined as a TOML section with a unique name (e.g., `[source_name]`). All entropy sources support these common parameters:

#### Required Parameters
- **`name`** (string): Human-readable description of the entropy source.
- **`interval`** (duration string): Minimum time between fetches. Format: `"5m"`, `"2h"`, `"30s"`.
- **`scale`** (float): Scaling factor (0.0 to 1.0) applied to computed entropy before adding to pool. Lower values reduce the entropy contribution.

#### Data Source Methods (Mutually Exclusive)

Choose **exactly one** of these methods to specify how data is obtained:

- **`url`** (string): HTTP/HTTPS URL to fetch data from.
- **`file`** (string): Local file path to read data from.
- **`command`** (array of strings): Command and arguments to execute. Stdout becomes the entropy data.
- **`script_interpreter` + `script`** (string + multiline string): Embedded script execution.

#### Optional Parameters
- **`size`** (integer): Maximum number of bytes to read from the data source. If not specified, reads all available data.
- **`min_size`** (integer): Minimum number of bytes required for the fetch to be considered successful. Fetches returning less data are ignored.
- **`no_compress`** (boolean): Skip compression of the fetched data. Useful for high-entropy sources like `/dev/random`. Default: `false`.
- **`init_delay`** (duration string): Delay before making the first fetch after daemon startup. Default: no delay.
- **`prefetch`** (string): URL to fetch (and discard) before fetching the main data source. Used for session setup.
- **`disabled`** (boolean): Temporarily disable this entropy source without removing its configuration. Default: `false`.

#### Script-Specific Parameters
When using `script_interpreter` and `script`:
- **`script_interpreter`** (string): Command to execute the script (e.g., `"python3"`, `"bash"`, `"powershell"`).
- **`script`** (multiline string): The script code to execute. Must output binary data to stdout.
- **Custom fields**: Any additional key-value pairs become environment variables accessible to the script with the `EGD_SOURCE_` prefix.

#### Parameter Combinations and Restrictions

**Mutually Exclusive Groups:**
1. **Data Source Methods**: Only one of `url`, `file`, `command`, or `script_interpreter`/`script` may be used per source.

**Compatible Parameter Combinations:**
- **URL sources**: `url` + `size` + `min_size` + `prefetch`
- **File sources**: `file` + `size`
- **Command sources**: `command` (size limits not applicable)
- **Script sources**: `script_interpreter` + `script` + custom fields

**Parameter Dependencies:**
- `script` requires `script_interpreter`
- `prefetch` only works with `url` sources
- `size` and `min_size` are ignored for `command` and `script` sources (scripts control their own output size)

### Example Configuration

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
scale = 0.8
amount = 1000
limit = 10000
script_interpreter = "bash"
script = '''
#!/bin/bash

# Get configuration from environment variables
AMOUNT="${EGD_SOURCE_AMOUNT:-1000}"
LIMIT="${EGD_SOURCE_LIMIT:-10000}"

# Fetch random numbers and process them
curl -s "http://www.randomnumbers.info/cgibin/wqrng.cgi?amount=${AMOUNT}&limit=${LIMIT}" | \
    grep -o ' [0-9]+' | tr -d ' '
'''

[wikipedia_random]
name = "Random Wikipedia page (without images)"
interval = "40m"
url = "https://en.wikipedia.org/wiki/Special:Random"
scale = 0.2

[wikipedia_images]
name = "5 latest image thumbnails on Wikipedia"
interval = "30m"
scale = 0.2
max_images = 5
script_interpreter = "python3"
script = '''
import urllib.request
import re
import os
import sys

# Get configuration from environment variables
max_images = int(os.environ.get("EGD_SOURCE_MAX_IMAGES", "5"))

# Fetch recent images from Wikipedia
httpResponse = urllib.request.urlopen("https://en.wikipedia.org/wiki/Special:NewFiles")
data = httpResponse.read().decode(encoding="latin_1")
count = 0
allimgdata = b""

for quotedurl in re.findall('"//upload\.wikimedia\.org/[^ "]+\.jpg"', data):
    if count >= max_images:
        break
    url = "https:" + quotedurl.strip('"')
    try:
        httpResponse = urllib.request.urlopen(url)
        imgdata = httpResponse.read()
        allimgdata += imgdata
        count += 1
    except Exception:
        continue

# Output binary data to stdout
sys.stdout.buffer.write(allimgdata)
'''

[wikipedia_changes]
name = "Summary of last 50 changes to Wikipedia"
interval = "6h"
url = "https://en.wikipedia.org/w/api.php?hidebots=1&hidecategorization=1&days=7&limit=50&action=feedrecentchanges&feedformat=atom"
scale = 0.05

[apod]
name = "Astronomy Picture of the Day"
interval = "24h"
size = 150000
scale = 0.1
script_interpreter = "python3"
script = '''
import urllib.request
import re
import os
import sys

# Get APOD image URL
httpResponse = urllib.request.urlopen("http://apod.nasa.gov/apod/")
data = httpResponse.read().decode(encoding="latin_1")
match = re.search(r'<img src="([^"]+)"', data, re.IGNORECASE)

if match:
    image_url = "http://apod.nasa.gov/apod/" + match.group(1)
    try:
        # Get size limit from environment (if set)
        size_limit = int(os.environ.get("EGD_SOURCE_SIZE", "150000"))
        
        # Fetch the image
        httpResponse = urllib.request.urlopen(image_url)
        image_data = httpResponse.read(size_limit)
        
        # Output binary data to stdout
        sys.stdout.buffer.write(image_data)
    except Exception:
        sys.exit(1)
else:
    # APOD is showing a video, not an image
    sys.exit(1)
'''

[npr_audio]
name = "NPR hourly news audio"
interval = "1h40m"
url = "http://public.npr.org/anon.npr-mp3/npr/news/newscast.mp3"
size = 150000
scale = 0.1

[nrl_world_sat]
name = "NRL world visible/IR satellite image"
interval = "3h"
min_size = 30000
scale = 0.05
script_interpreter = "python3"
script = '''
import urllib.request
import time
import os
import sys

# Get configuration from environment
min_size = int(os.environ.get("EGD_SOURCE_MIN_SIZE", "30000"))

# Calculate URL for image from 4 hours ago
now = time.gmtime(time.time() - 4 * 3600)
base_url = "http://www.nrlmry.navy.mil/archdat/global/stitched/day_night_bm/{0}{1:02}{2:02}.{3:02}00.multisat.visir.bckgr.Global_Global_bm.DAYNGT.jpg"
url = base_url.format(now.tm_year, now.tm_mon, now.tm_mday, now.tm_hour // 3 * 3)

try:
    httpResponse = urllib.request.urlopen(url)
    image_data = httpResponse.read()
    
    # Check minimum size requirement
    if len(image_data) >= min_size:
        sys.stdout.buffer.write(image_data)
    else:
        sys.exit(1)
except Exception:
    sys.exit(1)
'''

[nist_beacon]
name = "NIST Hardware Random Numbers Beacon"
interval = "5m"
scale = 0.8
script_interpreter = "python3"
script = '''
import urllib.request
import re
import time
import sys

# Get the data from 80 seconds ago to ensure it exists
seconds = round(time.time() - 80)

try:
    httpResponse = urllib.request.urlopen("https://beacon.nist.gov/rest/record/last")
    data = httpResponse.read().decode()
    
    # Extract the random value from XML
    match = re.search(r"<outputvalue>([0-9a-f]+)</outputvalue>", data, re.IGNORECASE)
    if match:
        hex_value = match.group(1)
        # Convert hex string to binary and output
        binary_data = bytes.fromhex(hex_value)
        sys.stdout.buffer.write(binary_data)
    else:
        sys.exit(1)
except Exception:
    sys.exit(1)
'''

[hotbits]
name = "Hotbits Radioactive Decay Random Numbers"
interval = "6h"
init_delay = "1h"
scale = 0.8
script_interpreter = "python3"
script = '''
import urllib.request
import re
import sys

try:
    httpResponse = urllib.request.urlopen("https://www.fourmilab.ch/cgi-bin/Hotbits?nbytes=2048&fmt=hex")
    data = httpResponse.read().decode()
    
    # Extract hex data from HTML
    match = re.search(r"<pre>([0-9a-f\n]+)</pre>", data, re.IGNORECASE)
    if match:
        hex_data = match.group(1).replace("\n", "")
        # Convert hex string to binary and output
        binary_data = bytes.fromhex(hex_data)
        sys.stdout.buffer.write(binary_data)
    else:
        sys.exit(1)
except Exception:
    sys.exit(1)
'''

[cnn_rss]
name = "CNN Latest Stories RSS Feed"
interval = "6h"
url = "http://rss.cnn.com/rss/cnn_latest.rss"
scale = 0.05

# Example of a disabled source
[anu_quantum]
name = "ANU Quantum Random Numbers Server"
disabled = true
interval = "6h"
scale = 0.8
script_interpreter = "python3"
script = '''
import urllib.request
import json
import sys

try:
    url = "http://qrng.anu.edu.au/API/jsonI.php?type=hex16&length=1000&size=4"
    httpResponse = urllib.request.urlopen(url)
    data = httpResponse.read().decode()
    
    # Parse JSON response
    json_data = json.loads(data)
    if "data" in json_data and json_data["data"]:
        # Join hex values and convert to binary
        hex_string = "".join(json_data["data"])
        binary_data = bytes.fromhex(hex_string)
        sys.stdout.buffer.write(binary_data)
    else:
        sys.exit(1)
except Exception:
    sys.exit(1)
'''
```

## Embedded Scripting System

### Script Execution Environment

The Go implementation supports embedded scripts in entropy source configurations to replace the Python function-based sources from the original `.egdconf` file. Scripts have access to configuration parameters through environment variables.

**Environment Variable Mapping:**
- All TOML configuration keys are converted to environment variables with the prefix `EGD_SOURCE_`
- Keys are converted to uppercase with underscores
- Examples:
  - `name` → `EGD_SOURCE_NAME`
  - `max_images` → `EGD_SOURCE_MAX_IMAGES`
  - `min_size` → `EGD_SOURCE_MIN_SIZE`

**Script Requirements:**
- Scripts must output binary data to stdout
- Scripts should exit with status 0 on success, non-zero on failure
- Scripts have a 30-second timeout for execution
- Popular scripting languages supported: Python, Bash, PowerShell, Perl, Ruby, etc.

**Script Configuration Format:**
```toml
[source_name]
name = "Human readable name"
interval = "30m"
scale = 0.8
script_interpreter = "python3"  # or bash, powershell, etc.
custom_param = "value"           # Available as EGD_SOURCE_CUSTOM_PARAM
script = '''
# Script code here
# Access config via: os.environ["EGD_SOURCE_CUSTOM_PARAM"]
'''
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

## Implementation Details

### Entropy Stirring Algorithm

Implements the exact same algorithm as Python version:
- Process entropy in 32-byte subblocks
- Compute SHA-256 hash over sliding 1024-byte window
- XOR each subblock with corresponding hash bytes

### Error Handling

**Error Classification Strategy:**

All errors in the EGD system are categorized into three types to ensure consistent handling:

```go
type ErrorCategory int

const (
    ErrorTemporary ErrorCategory = iota  // Retry later with backoff
    ErrorPermanent                       // Disable source or fail fast  
    ErrorFatal                          // Stop daemon immediately
)

type EGDError struct {
    Category    ErrorCategory
    Source      string           // Component that generated error
    Message     string           // Human-readable error message
    Cause       error           // Underlying error (for error chains)
    Code        string          // Machine-readable error code
    Metadata    map[string]any  // Additional context data
}

func (e *EGDError) Error() string {
    return fmt.Sprintf("[%s] %s: %s", e.Source, e.Code, e.Message)
}
```

**Error Categories by Component:**

**Configuration Errors (Fatal):**
- Invalid TOML syntax → Fatal (CONFIG_INVALID_TOML)
- Missing required fields → Fatal (CONFIG_MISSING_REQUIRED)
- Invalid value ranges → Fatal (CONFIG_INVALID_VALUE)
- Mutually exclusive fields → Fatal (CONFIG_MUTUALLY_EXCLUSIVE)

**Entropy Source Errors:**
- Network timeouts → Temporary (SOURCE_NETWORK_TIMEOUT)
- HTTP 5xx errors → Temporary (SOURCE_SERVER_ERROR)
- HTTP 4xx errors → Permanent (SOURCE_CLIENT_ERROR)
- File not found → Permanent (SOURCE_FILE_NOT_FOUND)
- Permission denied → Permanent (SOURCE_PERMISSION_DENIED)
- Script timeout → Temporary (SOURCE_SCRIPT_TIMEOUT)
- Script memory limit → Permanent after 3 failures (SOURCE_SCRIPT_MEMORY)
- Command not found → Permanent (SOURCE_COMMAND_NOT_FOUND)

**Pool/Storage Errors:**
- Disk full → Temporary (STORAGE_DISK_FULL)
- Permission denied → Fatal (STORAGE_PERMISSION_DENIED)
- Corruption detected → Fatal (STORAGE_CORRUPTED)
- I/O errors → Temporary (STORAGE_IO_ERROR)

**Daemon/System Errors:**
- Lock file conflicts → Fatal (DAEMON_ALREADY_RUNNING)
- Port binding failed → Fatal (DAEMON_PORT_IN_USE)
- Signal handling errors → Fatal (DAEMON_SIGNAL_ERROR)

**Retry and Backoff Strategy:**

```go
type RetryConfig struct {
    MaxRetries      int           // Maximum retry attempts
    BaseDelay       time.Duration // Initial delay between retries
    MaxDelay        time.Duration // Maximum delay (capped exponential backoff)
    BackoffFactor   float64       // Exponential backoff multiplier
    Jitter          bool          // Add random jitter to prevent thundering herd
}

// Default retry configurations by error type
var RetryConfigs = map[ErrorCategory]RetryConfig{
    ErrorTemporary: {
        MaxRetries:    5,
        BaseDelay:     time.Second * 10,
        MaxDelay:      time.Minute * 5,
        BackoffFactor: 2.0,
        Jitter:        true,
    },
    ErrorPermanent: {
        MaxRetries: 0, // No retries for permanent errors
    },
    ErrorFatal: {
        MaxRetries: 0, // No retries for fatal errors
    },
}
```

**Source Failure Tracking:**

```go
type SourceFailureTracker struct {
    ConsecutiveFailures int
    LastFailureTime     time.Time
    TotalFailures       int64
    DisabledUntil      time.Time
    LastError          *EGDError
}

// Disable thresholds
const (
    MaxConsecutiveFailures = 5
    DisableDuration        = time.Hour * 1  // Temporary disable duration
    MaxDisableDuration     = time.Hour * 24 // Maximum disable duration
)
```

**Error Reporting and Logging:**

- **Structured Logging**: All errors logged with consistent fields (timestamp, level, source, code, message, metadata)
- **Error Aggregation**: Group similar errors to prevent log spam
- **Escalation**: Critical errors (Fatal category) trigger immediate alerts
- **Context Preservation**: Error chains maintain full context through wrapping
- **Metrics Integration**: Error rates and patterns tracked for monitoring

**Error Recovery Patterns:**

1. **Graceful Degradation**: Continue operation with reduced functionality when possible
2. **Circuit Breaker**: Temporarily disable failing components to prevent cascade failures  
3. **Fallback Sources**: Use alternative entropy sources when primary sources fail
4. **State Preservation**: Always persist entropy pool before handling fatal errors
5. **Clean Shutdown**: Ensure proper cleanup even during error conditions

**Implementation Requirements:**

- All public methods must return typed errors with proper categorization
- Error messages must be actionable and include relevant context
- Temporary failures must implement exponential backoff with jitter
- Permanent failures must log sufficient detail for troubleshooting
- Fatal errors must trigger graceful shutdown with state preservation

### Platform Considerations

- **Lock Files** - Platform-specific paths (`/tmp/egd.lck` on Unix, temp dir on Windows)
- **Signals** - Cross-platform graceful shutdown using `os.Interrupt` (all platforms) and `syscall.SIGTERM` (Unix only)
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
