# Source_Files/TCPMess/MessageHandler.h

## File Purpose
Defines the abstract interface and template-based implementations for message handlers in a TCP messaging system. Enables type-safe dispatch of incoming messages to registered handler functions or methods via a polymorphic handler framework.

## Core Responsibilities
- Define abstract `MessageHandler` base class for message dispatching
- Provide `TypedMessageHandlerFunction` template for wrapping free functions as message handlers
- Provide `MessageHandlerMethod` template for wrapping class methods as message handlers
- Support dynamic casting of messages and channels to their concrete types
- Enable flexible handler registration without explicit subclassing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `MessageHandler` | class (abstract base) | Interface for all message handlers; provides virtual `handle()` method |
| `TypedMessageHandlerFunction<tMessage, tChannel>` | template class | Wraps a free function pointer as a message handler with type parameters |
| `MessageHandlerMethod<tTargetClass, tMessage, tChannel>` | template class | Wraps a class method pointer as a message handler with type parameters |
| `MessageHandlerFunction` | typedef | Convenience alias for `TypedMessageHandlerFunction<Message>` |

## Global / File-Static State
None.

## Key Functions / Methods

### MessageHandler::handle (pure virtual)
- Signature: `virtual void handle(Message* inMessage, CommunicationsChannel* inChannel) = 0`
- Purpose: Abstract entry point for message dispatch
- Inputs: `inMessage` (Message to handle), `inChannel` (originating channel)
- Outputs/Return: void
- Side effects: Implementation-dependent
- Notes: Pure virtual; concrete implementations in derived templates

### TypedMessageHandlerFunction::handle
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)` (override)
- Purpose: Dispatches message to wrapped function pointer with type casting
- Inputs: `inMessage` (cast to `tMessage*`), `inChannel` (cast to `tChannel*`)
- Outputs/Return: void
- Side effects: Invokes wrapped function pointer
- Calls: User-provided function via `mFunction`
- Notes: Checks `mFunction != NULL` before invocation; uses `dynamic_cast` for type safety

### MessageHandlerMethod::handle
- Signature: `void handle(Message* inMessage, CommunicationsChannel* inChannel)` (override)
- Purpose: Dispatches message to wrapped class method with type casting
- Inputs: `inMessage` (cast to `tMessage*`), `inChannel` (cast to `tChannel*`)
- Outputs/Return: void
- Side effects: Invokes method on stored object
- Calls: User-provided method via `(mObject->*(mMethod))`
- Notes: Checks both `mObject != NULL` and `mMethod != NULL` before invocation

### newMessageHandlerMethod (helper factory)
- Signature: `template<...> static inline MessageHandlerMethod<tTargetClass, tMessage, tChannel>* newMessageHandlerMethod(tTargetClass* targetObject, void (tTargetClass::*targetMethod)(...))`
- Purpose: Factory function to create typed method handlers with template deduction
- Inputs: `targetObject` (instance pointer), `targetMethod` (method pointer)
- Outputs/Return: Heap-allocated `MessageHandlerMethod*`
- Side effects: Dynamic memory allocation
- Notes: Caller responsible for deletion

## Control Flow Notes
This file defines the handler registration/dispatch layer of a networking messaging system. At runtime:
1. Messages arrive on a `CommunicationsChannel`
2. A dispatcher (not in this file) calls `handle()` on registered handlers
3. Template specialization provides type-safe upcasting from base `Message`/`CommunicationsChannel` types to concrete types
4. Handler functions/methods execute with properly typed parameters

## External Dependencies
- `<cstdlib>` ΓÇô standard library (included but unused; likely legacy)
- **Forward declarations** (defined elsewhere): `Message`, `CommunicationsChannel`
- No direct dependencies on other project files visible in this header
