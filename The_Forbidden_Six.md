# The Forbidden Six

| Field | Detail |
| :--- | :--- |
| **Challenge** | The Forbidden Six |
| **Category** | Web Exploitation |
| **Points** | 100 |
| **Author** | CYBxM0nk |
| **Flag** | `NullOrigin{tH3_c1ient_siD3_spe4ks_th3_TrUth}` |

---

## 1. Challenge Overview

The challenge presented a web-based Ludo game interface:
* Three tokens were already safely home.
* Token #4 was positioned exactly 6 squares away from the winning finish spot.
* The dice roll was hardcoded / fixed at `6`, meaning any valid forward move of token #4 would immediately trigger a win condition.

---

## 2. Initial Reconnaissance & Source Code Analysis

Inspecting the client-side JavaScript logic revealed a validation function named `validateMove()`:
* The client-side script simulated the move locally and purposefully blocked outgoing HTTP requests if the move resulted in a win.
* While this initially appeared to be a simple client-side bypass where one could send a direct request to `/api/move`, an inline code comment in the source provided crucial context:
  > *"check the session-handling logic around /api/reset instead — repeated resets without a client id have been flagged internally as a possible race condition."*
* The comment explained that an earlier patch had added server-side verification to `/api/move`, which explicitly checked the win condition and rejected winning moves with `"win validation failed"`.

---

## 3. Vulnerability Analysis & Exploitation

### Root Cause
The backend server suffered from a race condition in its session and state management logic triggered by unauthenticated / missing-client-ID calls to the `/api/reset` endpoint.

### Attack Steps
1. **Bypassing Frontend Validation:**
   * Bypassed the browser's client-side `validateMove()` function completely by targeting the underlying REST endpoints directly via automated requests.

2. **Triggering the Race Condition:**
   * Sent concurrent asynchronous requests to the `/api/reset` endpoint to disrupt the server's session state and win-validation lock mechanism.

3. **Submitting the Winning Move:**
   * Simultaneously dispatched the winning payload directly to `/api/move`:
     ```json
     {
       "token": 4,
       "dice": 6
     }
     ```

4. **Result:**
   * With the session state compromised via the concurrent reset interaction, the server-side win validator failed to enforce the block, accepting the move.
   * The server returned the winning state response containing the flag.

---

## 4. Flag

```text
NullOrigin{tH3_c1ient_siD3_spe4ks_th3_TrUth}
```
