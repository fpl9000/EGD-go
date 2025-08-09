1. Entropy Stirring Algorithm Implementation

Issue: The stirring algorithm is mentioned but not fully specified.
- How exactly does the "sliding 1024-byte window" work when data is shorter than 1024 bytes?
- What happens with the last incomplete 32-byte block?
- Should the algorithm match Python's behavior exactly? (Need Python reference implementation
  details)

Resolution:
- The Go implementation of the stirring algorithm must exactly match the Python implementation in egd.py

2. CLI Command Implementation

Issue: CLI commands are listed but implementation details are missing.
- How does egd status connect to the daemon and parse responses?
- What's the exact output format for each command?
- How should CLI handle daemon connection failures?
- Should CLI commands have timeout values?

Resolution:
- CLI commands connect via TCP using the port specified in .egdconf configuration file
- CLI reads subset of TOML configuration to determine connection parameters
- Responses are parsed as JSON according to TCP Protocol Specification in design.md
- CLI output format must match the Python version's format (exact format TBD when merging into main design)
- CLI reports detailed error messages on connection failures including technical details (connection refused, no route to host, etc.)
- CLI uses 30-second timeout as specified in design.md with user feedback showing remaining time

3. HTTP Client Configuration

Issue: Entropy sources using URLs lack HTTP client details.
- What User-Agent string should be used?
- Timeout values for different operations (connect, read, total)?
- Retry logic for HTTP failures?
- SSL/TLS certificate validation policies?
- Proxy support requirements?

Resolution:
- User-Agent string: EGD-Go/1.0
- HTTP timeout: 60 seconds, with timeout reset to 60 seconds when first byte is received
- No retry logic for individual HTTP requests
- Entropy source disabled after 5 consecutive failures with log message generated
- SSL/TLS certificate validation: strict by default, with per-source configuration option to disable (TOML syntax TBD when merging)
- No proxy support (add to Future Enhancements in design.md)

4. Script Execution Implementation Details

Issue: Security boundaries are defined but implementation specifics are unclear.
- How to actually monitor and enforce memory limits cross-platform?
- How to create "secure temporary directories"?
- Which specific environment variables should be preserved (PATH, HOME, TEMP)?
- How to handle script interpreter discovery and validation?

Resolution:
- TBD

5. Platform-Specific File Paths

Issue: Lock files and temp directories mentioned but paths not specified.
- Exact lock file paths for each platform?
- How to determine appropriate temp directory locations?
- File permission requirements (mentioned 0600 but not consistently)?

Resolution:
- TBD

6. Entropy Pool Chunk Management

Issue: Pool chunk allocation and lifecycle unclear.
- When are new chunks created vs. reusing existing chunks?
- How are chunk IDs assigned and managed?
- What triggers chunk consolidation or cleanup?

Resolution:
- TBD

7. Cross-Platform Signal Handling Details

Issue: General approach described but missing implementation specifics.
- Conditional compilation strategy (build tags vs. runtime checks)?
- Windows service integration requirements?
- How to handle platform-specific signal differences?

Resolution:
- TBD

8. Testing Strategy Specifics

Issue: Testing categories mentioned but no specific test requirements.
- What constitutes "exact functional equivalence" testing with Python version?
- How to test entropy stirring algorithm correctness?
- Mock HTTP server requirements for network source testing?
- How to test script execution security boundaries?

Resolution:
- TBD

9. Dependency Version Requirements

Issue: External dependencies listed but no version constraints.
- Which versions of github.com/spf13/cobra, github.com/BurntSushi/toml, etc.?
- Minimum Go version requirement (mentioned Go 1.21+ for slog but not consistently)?

Resolution:
- TBD

10. Performance Requirements

Issue: No quantitative performance targets specified.
- Expected entropy collection rates?
- Memory usage targets?
- CPU usage limits during normal operation?
- Acceptable latency for TCP commands?

Resolution:
- TBD
