# Source_Files/TCPMess/MessageHandler.cpp

## File Purpose
Implementation stub for the MessageHandler system. All meaningful code is defined inline in the header; this file contains only conditional compilation guards and the header include.

## Core Responsibilities
- Include MessageHandler.h declarations (conditionally compiled when networking is enabled)
- Serve as the implementation unit for the message handling callback system

## Key Types / Data Structures
None (all types defined in MessageHandler.h).

| Name | Kind | Purpose |
|------|------|---------|
| MessageHandler | abstract class | Base interface for message dispatch callbacks |
| TypedMessageHandlerFunction | template class | Wraps function pointers as MessageHandler implementations |
| MessageHandlerMethod | template class | Wraps member function pointers as MessageHandler implementations |
| MessageHandlerFunction | typedef | Specialization of TypedMessageHandlerFunction for generic Message |

## Global / File-Static State
None.

## Key Functions / Methods

### handle (MessageHandler)
- Signature: `virtual void handle(Message* inMessage, CommunicationsChannel* inChannel) = 0`
- Purpose: Pure virtual; subclasses override to dispatch incoming messages to callbacks
- Inputs: Generic Message pointer, CommunicationsChannel pointer
- Outputs/Return: void
- Notes: Implemented by TypedMessageHandlerFunction and MessageHandlerMethod

### handle (TypedMessageHandlerFunction)
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)` (override)
- Purpose: Casts generic pointers to specific types and invokes wrapped function pointer
- Inputs: Generic Message and CommunicationsChannel pointers
- Outputs/Return: void
- Side effects: Calls mFunction if non-null
- Calls: `dynamic_cast`, mFunction
- Notes: Silent cast failure (no validation after cast)

### handle (MessageHandlerMethod)
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)` (override)
- Purpose: Casts generic pointers to specific types and invokes wrapped member function
- Inputs: Generic Message and CommunicationsChannel pointers
- Outputs/Return: void
- Side effects: Calls mMethod on mObject if both non-null
- Calls: `dynamic_cast`, member function via pointer-to-member operator
- Notes: Silent cast failure; checks both object and method pointers before invocation

### newMessageHandlerMethod
- Signature: `template<...> static inline MessageHandlerMethod<...>* newMessageHandlerMethod(...)`
- Purpose: Factory function for creating type-safe MessageHandlerMethod instances
- Inputs: Target object pointer, target member function pointer
- Outputs/Return: Heap-allocated MessageHandlerMethod pointer
- Side effects: Dynamic allocation
- Notes: Deduces template parameters from arguments

## Control Flow Notes
Part of a TCP messaging/event dispatch system. Callbacks are registered as MessageHandler subclasses; the dispatcher calls `handle()` with incoming messages, which route to the wrapped function/method. Fits into an async message-receive flow.

## External Dependencies
- `#include <cstdlib>` (header)
- Forward declared: `Message`, `CommunicationsChannel` (defined elsewhere)
- Conditional: `DISABLE_NETWORKING` guard wraps entire file
