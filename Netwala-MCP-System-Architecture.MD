# Netwala MCP V3.0 --- System Architecture

This document describes the documented high-level architecture, request
lifecycle, browser-management model, deployment assumptions, and
engineering considerations for Netwala MCP.

> **Implementation note:** This document is based on the
> project-provided architecture description. Confirm
> implementation-specific details against the current source code and
> runtime before treating them as verified guarantees.

## 1. High-level architecture

Netwala uses an MCP client/server architecture. In the Android
deployment described by the project, the MCP client runs in Termux while
the MCP server and Playwright runtime run inside Debian PRoot. They
communicate over HTTP through the shared loopback network namespace.

``` text
+----------------------------- ANDROID DEVICE ------------------------------+
|                                                                            |
|  +---------------------------+       +---------------------------------+   |
|  | TERMUX HOST               |       | DEBIAN PROOT                    |   |
|  |                           |       |                                 |   |
|  | +-----------------------+ | HTTP  | +-----------------------------+ |   |
|  | | Hermes / MCP Client   | |------>| | netwala.py / FastMCP        | |   |
|  | +-----------------------+ |       | | 54 custom tools              | |   |
|  |                           |       | +--------------+--------------+ |   |
|  |                           |       |                |                |   |
|  |                           |       | +--------------v--------------+ |   |
|  |                           |       | | browser_manager.py          | |   |
|  |                           |       | | Playwright async API        | |   |
|  |                           |       | +--------------+--------------+ |   |
|  |                           |       |                |                |   |
|  |                           |       |       +--------+--------+       |   |
|  |                           |       |       |                 |       |   |
|  |                           |       |  +----v------+    +-----v----+  |   |
|  |                           |       |  | Firefox   |    | Chromium |  |   |
|  |                           |       |  +-----------+    +----------+  |   |
|  +---------------------------+       +---------------------------------+   |
|                                                                            |
+----------------------------------------------------------------------------+
```

## 2. Core components

### A. Transport layer

-   Protocol: Model Context Protocol over Streamable HTTP.
-   Documented default address: `127.0.0.1:8765`.
-   Documented endpoint: `/mcp`.
-   Deployment purpose: bridge an MCP client in Termux to a server
    running inside Debian PRoot.

The loopback arrangement relies on the host and PRoot environment
sharing the expected network namespace and routing behavior. Validate
connectivity in the actual deployment.

### B. MCP server --- `netwala.py`

-   Framework: Python MCP/FastMCP.
-   Custom tool inventory: 54 tools.
-   Standard MCP meta-tools: 4.
-   Responsibility: expose tool schemas, receive MCP calls, validate
    arguments, dispatch operations, and return results.
-   Execution model: asynchronous workflows using Python's async
    facilities, as implemented by the server.

### C. Browser manager --- `browser_manager.py`

-   Browser automation: Playwright asynchronous API.
-   Engines: Firefox and Chromium, where compatible binaries and
    dependencies are installed.
-   Responsibilities: browser lifecycle, page management, context
    creation, and browser operations.
-   Context management: create and dispose isolated browser contexts for
    independent sessions or agent tasks.

Exact Playwright and browser versions should be read from the
checked-out dependency configuration and runtime. Version numbers can
change between releases.

## 3. Request lifecycle

Example user request: "Navigate to an authorized test application and
inspect its security headers."

1.  **LLM planning:** The MCP client determines which Netwala tool and
    arguments are appropriate.
2.  **Transport:** The client sends an MCP request to the configured
    Streamable HTTP endpoint.
3.  **MCP dispatch:** `netwala.py` receives the request and validates
    the tool arguments against its schema.
4.  **Tool execution:** The selected tool calls the relevant
    browser-manager or network-inspection operation.
5.  **Browser operation:** Playwright performs the requested action in
    the selected browser context.
6.  **Result construction:** Netwala returns structured output, errors,
    or evidence references.
7.  **Agent reasoning:** The client interprets the result and chooses
    the next authorized step.

A successful HTTP response alone does not prove that the requested
browser operation succeeded. The tool result and evidence must also be
checked.

## 4. Browser context isolation

Browser contexts can provide separate browser state, including cookies
and local storage.

Typical use cases:

-   Separate test accounts.
-   Independent application sessions.
-   Parallel reconnaissance tasks.
-   Testing different device or locale configurations.

Important boundaries:

-   Browser contexts are not operating-system sandboxes.
-   Contexts may still share the same browser process and host
    resources.
-   Saved storage state can contain reusable authentication material.
-   Parallel contexts can increase CPU and memory pressure.

Destroy contexts when they are no longer needed, and protect any
exported storage state.

## 5. Network and proxy configuration

The project describes several possible network configuration layers:

1.  **Host network:** Android or Linux routing, VPN, and host-level
    proxy configuration.
2.  **Browser context:** Per-context proxy settings where supported by
    the Playwright/browser configuration.
3.  **Request handling:** Browser routing and interception for
    inspecting or modifying selected HTTP traffic.

The effective network path depends on the host, browser, context, and
proxy configuration. Validate the observed egress route before testing.

Only use proxy routing and request modification for authorized
assessments. Do not assume that a browser-level setting changes all
traffic generated by the host.

## 6. Browser automation and fingerprinting

Netwala provides browser interaction features such as
framework-compatible input handling and configurable mouse movement.

These features are automation techniques, not guarantees of stealth or
bot-protection bypass.

-   Firefox does not guarantee a particular JA3/JA4 TLS fingerprint.
-   TLS fingerprints depend on the actual TLS implementation and network
    stack.
-   Browser fingerprinting can use many signals beyond the browser
    engine.
-   Framework-compatible input events do not guarantee that application
    automation checks will accept an interaction.
-   Mouse movement simulation does not make an automated session
    indistinguishable from a human.

Treat detection behavior as target-specific and validate it in an
authorized test environment.

## 7. Tool groups

The 54 custom tools are organized into these groups:

  Group                            Count
  ----------------------------- --------
  Core & Navigation                   10
  Interaction & Input                  8
  Vision, State & Frames               9
  Recon & Crawling                     8
  Exploitation & Interception          7
  Network & Secrets                    5
  Spoofing, Swarm & Storage            7
  **Custom tools**                **54**
  Standard MCP meta-tools              4
  **Total**                       **58**

See [`README.md`](README.md) for the complete tool-by-tool inventory.

## 8. Documented repository layout

The project-provided layout describes the following files and locations.
Confirm the actual current tree before relying on these paths:

``` text
Termux host
├── start_netwala.sh
├── stop_netwala.sh
├── netwala-CHEATSHEET.txt
└── .hermes/
    └── client configuration

Debian PRoot
└── Netwala repository
    ├── netwala.py
    ├── browser_manager.py
    ├── backup/
    └── tests/
```

## 9. Resource considerations

The project notes an approximate Firefox context memory footprint of
around 150 MB. This is an observation, not a fixed allocation: actual
memory usage depends on the browser version, pages, extensions, target
application, and workload.

On memory-constrained Android devices:

-   Limit concurrent browser contexts.
-   Close unused pages and contexts.
-   Avoid capturing unnecessarily large artifacts.
-   Monitor memory and process termination.
-   Prefer small, controlled batches over unbounded parallel execution.

## 10. Security boundaries

Netwala can handle authenticated sessions and sensitive network
artifacts. Deployment should account for:

-   MCP endpoint access control.
-   Loopback-only binding by default.
-   Authentication and transport protection for any remote deployment.
-   Secure handling of cookies, tokens, HAR files, screenshots, and
    storage-state files.
-   Isolation between unrelated targets and test accounts.
-   Explicit authorization and scope enforcement.
-   Redaction of secrets in logs and reports.

A loopback bind reduces network exposure but is not a substitute for
authentication or host security when the deployment changes.

## 11. Design decisions

  -----------------------------------------------------------------------
  Decision                            Rationale
  ----------------------------------- -----------------------------------
  Playwright                          Provides a unified automation API
                                      for supported browser engines.

  Streamable HTTP                     Supports a client/server
                                      arrangement across the documented
                                      Termux and PRoot setup.

  Loopback bind                       Keeps the default listener local to
                                      the expected environment.

  Separate browser contexts           Helps isolate browser session state
                                      between tasks.

  Focused tool inventory              Exposes distinct operations to the
                                      MCP client without requiring one
                                      oversized tool.

  Evidence-oriented inspection        Screenshots, snapshots, network
                                      captures, and structured outputs
                                      help make results reviewable.
  -----------------------------------------------------------------------

## 12. Known limitations

-   Playwright browser support varies by OS and CPU architecture.
-   Android PRoot may need platform-specific installation steps.
-   Browser contexts do not provide OS-level isolation.
-   Parallel browser sessions can exhaust memory.
-   Browser automation and fingerprint behavior are target-dependent.
-   Tool schemas and implementation behavior can change between
    releases.

## 13. Future architecture work

-   Camoufox evaluation and integration, subject to compatibility
    testing.
-   Web dashboard for browser and agent monitoring.
-   Burp Suite request-export bridge.
-   Configurable proxy-provider integrations.
-   Docker deployment.
-   Resource-aware multi-agent scheduling.
-   Structured evidence storage and regression verification.

------------------------------------------------------------------------

**Project:** Netwala MCP\
**Repository:** https://github.com/Piyankali/Netwala-\
**Version:** 3.0\
**Tool inventory:** 54 custom + 4 MCP meta-tools
