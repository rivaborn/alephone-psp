# Source_Files/LibNAT/error.h

## File Purpose
Defines error and status codes for the LibNAT (Network Address Translation) library across socket, HTTP, SSDP, and UPnP subsystems. Provides a centralized error-code enumeration system where all LibNAT functions return error codes that can be printed to stderr via a utility function.

## Core Responsibilities
- Define success/failure status codes (OK, LNAT_ERROR)
- Define generic errors (memory allocation, parameter validation)
- Define socket-level errors (connection, send/recv timeouts, socket operations)
- Define HTTP protocol errors (header parsing, content-length, URL validation)
- Define SSDP discovery errors (header parsing, missing location tags)
- Define UPnP errors (description parsing, public IP discovery)
- Provide error-to-text printing via LNat_Print_Internal_Error()

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### LNat_Print_Internal_Error
- Signature: `void LNat_Print_Internal_Error(int error);`
- Purpose: Print human-readable error message to stderr for a given error code
- Inputs: `int error` ΓÇô error code defined in this header (negative integers, or OK=0)
- Outputs/Return: `void` (output to stderr)
- Side effects: I/O to stderr
- Calls: Not inferable from this file (implementation elsewhere)
- Notes: Expected to be called by library clients after receiving an error code from any LibNAT function. Implementation must map error codes to descriptive messages.

## Control Flow Notes
This is a passive error-definition module. All LibNAT functions propagate error codes up the call stack to the caller, which can then invoke LNat_Print_Internal_Error() to diagnose failures. No initialization or shutdown required.

## External Dependencies
- Implicitly uses standard C library (stderr output in LNat_Print_Internal_Error implementation)
- No explicit includes in this header
