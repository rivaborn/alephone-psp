# Source_Files/Misc/VecOps.h

## File Purpose
Template header providing basic 3D vector operations for engine-wide use. Supports vectors as `T*` (3-element arrays) with type-agnostic element conversion, enabling operations across different numeric types (e.g., integer to float vectors).

## Core Responsibilities
- Vector copy with type conversion
- Vector addition and subtraction
- In-place vector operations (add-to, subtract-from)
- Scalar multiplication (in-place and copy-form)
- Dot product (scalar product) computation
- Cross product computation

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### VecCopy
- **Signature:** `template <class T0, class T1> inline void VecCopy(const T0* V0, T1* V1)`
- **Purpose:** Copy one vector to another with element-wise type conversion.
- **Inputs:** Source vector `V0` (read-only pointer to 3 elements of type `T0`).
- **Outputs/Return:** Destination vector `V1` (pointer to 3 elements of type `T1`); modified in place.
- **Side effects:** Writes to `V1[0..2]`.
- **Notes:** Assumes both pointers are valid and point to 3-element arrays.

### VecAdd
- **Signature:** `template <class T0, class T1, class T2> inline void VecAdd(const T0* V0, const T1* V1, T2* V2)`
- **Purpose:** Add two vectors element-wise, storing result in a third with optional type conversion.
- **Inputs:** Vectors `V0`, `V1` (read-only).
- **Outputs/Return:** Result vector `V2`; modified in place.
- **Calls:** Implicit conversion to `T2`.

### VecSub
- **Signature:** `template <class T0, class T1, class T2> inline void VecSub(const T0* V0, const T1* V1, T2* V2)`
- **Purpose:** Subtract `V1` from `V0`, storing result in `V2` with optional type conversion.

### VecAddTo / VecSubFrom
- **Purpose:** In-place addition (`V0 += V1`) and subtraction (`V0 -= V1`).
- **Inputs/Outputs:** First vector modified directly.

### VecScalarMult / VecScalarMultTo
- **Purpose:** Multiply vector by scalar, either storing in new vector or in-place.
- **Inputs:** Vector and scalar value.

### ScalarProd
- **Signature:** `template <class T> inline T ScalarProd(const T* V0, const T* V1)`
- **Purpose:** Compute dot product `V0 ┬╖ V1`.
- **Outputs/Return:** Scalar result of type `T`.
- **Notes:** Both vectors must have same element type.

### VectorProd
- **Signature:** `template <class T0, class T1, class T2> inline void VectorProd(const T0* V0, const T1* V1, T2* V2)`
- **Purpose:** Compute cross product `V0 ├ù V1`, storing in `V2`.
- **Outputs/Return:** Result vector `V2`; modified in place.
- **Notes:** Uses determinant expansion formula for 3D cross product.

## Control Flow Notes
Utility functions called throughout the engine during physics calculations, graphics transformations, and spatial reasoning (no participation in init/frame/update cycles directly).

## External Dependencies
- **Self-contained:** No external includes; uses only inline arithmetic and type casting.
- **Assumes:** Callers provide valid pointers to 3-element arrays.
