# Code Review: setWaitingInterval.ts

## 1. **Destructive Array Mutation**
**Severity**: High  
**Location**: Lines 5-17   
**Problem**: The `getLastUntilOneLeft()` function uses `arr.pop()`, which mutates the original `timeouts` array. This means:
- After the first call, the array permanently loses elements
- Subsequent intervals will have incorrect timing
- Multiple intervals sharing the same array will interfere with each other

**Example of the bug**:
```javascript
const delays = [16, 8, 4, 2];
setWaitingInterval(callback, delays);
// delays is now [16, 8, 4], the 2 is gone!
setWaitingInterval(callback, delays); // This will use wrong timings
```

**Recommendation**: Either clone the array or use indexing instead of mutation (will require callCount):
```javascript
function getTimeout(arr: number[], callCount: number): number {
    return arr[Math.max(0, arr.length - 1 - callCount)];
}
```

---

## 2. **Incorrect Argument Passing**
**Severity**: High  
**Location**: Line 31   
**Problem**: `handler(argsInternal)` passes arguments as a single array instead of spreading them. If the handler expects multiple arguments, it will receive an array as the first argument.

**Expected**: `handler(arg1, arg2, arg3)`  
**Actual**: `handler([arg1, arg2, arg3])`

**Recommendation**:
```javascript
handler(...argsInternal); // or handler(...args)
```

---

## 3. **Global State with Module-Level Variables**
**Severity**: High  
**Location**: Lines 1, 3  
**Problem**:
- `map` and `waitingIntervalId` are shared across all instances
- Not safe for concurrent use or testing
- Can cause memory leaks (cleared intervals remain in map)
- ID collision risk if counter overflows

**Recommendation**: Encapsulate state properly:
```javascript
class WaitingIntervalManager {
    private map = new Map<number, number>();
    private nextId = 0;
    
    // methods here
}
```

---

## 4. **Memory Leak in clearWaitingInterval**
**Severity**: Medium  
**Location**: Lines 46-52  
**Problem**: After calling `clearTimeout()`, the entry is never removed from the `map`, causing memory leaks.

**Recommendation**:
```javascript
export function clearWaitingInterval(intervalId: number): void {
    const realTimeoutId = map.get(intervalId);
    
    if (typeof realTimeoutId === 'number') {
        clearTimeout(realTimeoutId);
        map.delete(intervalId); // Clean up
    }
}
```

---

## 5. **ID Counter Overflow Risk**
**Severity**: Low  
**Location**: Line 28  
**Problem**: `waitingIntervalId` will eventually overflow after 2^53 calls, potentially causing ID collisions.

**Recommendation**: Reset to 1 when it exceeds `Number.MAX_SAFE_INTEGER`, or use a better ID generation strategy.

---


## 6. **Confusing Array Order**
**Severity**: Medium  
**Location**: Documentation comment  
**Problem**: The comment says `[16, 8, 4, 2]` produces delays `2, 4, 8, 16, 16...` which is counterintuitive. Most developers would expect the array to be in the order delays are applied: `[2, 4, 8, 16]`.

**Recommendation**: Either reverse the expected input or update the function logic to read from index 0 onwards.

---

## 7. **Unused Type Check**
**Severity**: Low  
**Location**: Lines 9  
**Problem**: The type check `typeof item !== 'number'` is redundant since TypeScript guarantees `arr: number[]`. This can only happen if someone passes a wrong type at runtime, but then the whole array would be wrong.

**Recommendation**: Remove the check and reduce indentation:
```javascript
if (arr.length === 0) {
    throw new Error('Timeouts array cannot be empty');
}
```

---

## 8. **Weak typing and implicit return**
**Severity**: Medium  
**Location**: Line 27  
**Problem:**`handler: Function` and `...args: any[]` disable TypeScript’s type safety.  
The return type is implicit.

**Recommendation**:
```javascript
export function setWaitingInterval<T extends any[]>(
    handler: (...args: T) => void,
    timeouts: number[],
    ...args: T
): number
```

---

## 9. **Missing Input Validation**
**Severity**: Medium  
**Location**: Line 27  
**Problem**: No validation for:
- Empty `timeouts` array
- Negative timeout values
- Non-array input

**Recommendation**: Add validation at the start:
```javascript
if (!timeouts.length) {
    throw new Error('Timeouts array cannot be empty');
}
if (timeouts.some(t => t < 0)) {
    throw new Error('Timeouts must be non-negative');
}
```
