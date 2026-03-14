# Source_Files/LibNAT/http.h

## File Purpose
Header file declaring HTTP client functions for constructing and sending GET/POST requests to a network device (router). Provides APIs for message generation, header manipulation, and request execution. Part of the LibNAT UPnP/NAT library for router communication.

## Core Responsibilities
- Declare constructors/destructors for HTTP GET and POST message structures
- Provide header field manipulation (Request and Entity headers for POST messages)
- Declare HTTP GET and POST request execution functions
- Manage memory allocation/deallocation for HTTP messages and responses

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| GetMessage | struct (opaque, forward-declared) | Encapsulates an HTTP GET request with host, resource, port |
| PostMessage | struct (opaque, forward-declared) | Encapsulates an HTTP POST request with host, resource, port, body, and headers |

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Generate_Http_Get
- **Signature:** `int LNat_Generate_Http_Get(const char * host, const char * resource, short int port, GetMessage ** gm)`
- **Purpose:** Allocate and initialize a GetMessage structure from connection parameters.
- **Inputs:** host (target address), resource (URI path), port (target port)
- **Outputs/Return:** gm (output parameter receiving allocated struct pointer), return code (OK on success, error otherwise)
- **Side effects:** Allocates memory for GetMessage; caller must call LNat_Destroy_Http_Get to free.
- **Calls:** Not inferable from this file (implementation elsewhere).
- **Notes:** Returns null in gm on failure; returns OK on success.

### LNat_Destroy_Http_Get
- **Signature:** `int LNat_Destroy_Http_Get(GetMessage ** gm)`
- **Purpose:** Free a GetMessage structure allocated by LNat_Generate_Http_Get.
- **Inputs:** gm (pointer to GetMessage pointer to be freed)
- **Outputs/Return:** Return code
- **Side effects:** Deallocates memory; gm becomes invalid.

### LNat_Generate_Http_Post
- **Signature:** `int LNat_Generate_Http_Post(const char * host, const char * resource, short int port, const char * body, PostMessage ** pm)`
- **Purpose:** Allocate and initialize a PostMessage structure with connection parameters and message body.
- **Inputs:** host, resource, port, body (message payload)
- **Outputs/Return:** pm (allocated PostMessage), return code
- **Side effects:** Allocates memory; caller must call LNat_Destroy_Http_Post.
- **Notes:** Caller can add headers after generation using LNat_Http_Post_Add_Request_Header and LNat_Http_Post_Add_Entity_Header.

### LNat_Http_Post_Add_Request_Header
- **Signature:** `int LNat_Http_Post_Add_Request_Header(PostMessage * pm, const char * token, const char * value)`
- **Purpose:** Append a Request Header Field to an existing PostMessage.
- **Inputs:** pm (PostMessage), token (header name), value (header value)
- **Outputs/Return:** Return code

### LNat_Http_Post_Add_Entity_Header
- **Signature:** `int LNat_Http_Post_Add_Entity_Header(PostMessage * pm, const char * token, const char * value)`
- **Purpose:** Append an Entity Header Field to an existing PostMessage.
- **Inputs:** pm, token, value
- **Outputs/Return:** Return code

### LNat_Http_Request_Get
- **Signature:** `int LNat_Http_Request_Get(GetMessage * gm, char ** response)`
- **Purpose:** Execute an HTTP GET request and retrieve the response.
- **Inputs:** gm (prepared GetMessage)
- **Outputs/Return:** response (allocated response body; caller must free with free()), return code (OK on success)
- **Side effects:** Network I/O; allocates response buffer.
- **Notes:** GetMessage must be prepared via LNat_Generate_Http_Get first.

### LNat_Http_Request_Post
- **Signature:** `int LNat_Http_Request_Post(PostMessage * pm, char ** response)`
- **Purpose:** Execute an HTTP POST request and retrieve the response.
- **Inputs:** pm (prepared PostMessage)
- **Outputs/Return:** response (allocated response body; caller must free), return code
- **Side effects:** Network I/O; allocates response buffer.
- **Notes:** PostMessage must be prepared via LNat_Generate_Http_Post first.

## Control Flow Notes
This header declares the HTTP communication layer, typically invoked during runtime when the application needs to send SOAP/HTTP commands to a UPnP-enabled router (e.g., during port mapping setup or device discovery). Lifecycle: generate message ΓåÆ (optionally add headers) ΓåÆ execute request ΓåÆ free response ΓåÆ destroy message.

## External Dependencies
- Standard C library (implied free() usage in comments)
- No includes visible; types are forward-declared to maintain encapsulation.
