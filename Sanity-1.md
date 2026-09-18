# Sanity-1

| Field | Detail |
| :--- | :--- |
| **Challenge** | Sanity-1 |
| **Category** | OSINT / Social Media Reconnaissance |
| **Platform** | Null Origin CTF |
| **Flag** | `Null0rigin{Y0u_L00ked_Cl0s3r}` |

---

## 1. Challenge Overview

The challenge provided an initial clue hinting at social media reconnaissance:
> *"Are you on IG? Maybe we should meet there 👀"*

This directly signaled investigating the event organizer's Instagram account (**CyberHX** / **Null Origin CTF**).

---

## 2. Reconnaissance & Discovery

1. Navigated to the official Instagram profile for CyberHX / Null Origin CTF.
2. Browsed the published posts and examined the comments section.
3. Located an anomalous comment string divided across two distinct comments:
   * First comment: `AhyyOevtva{LOh_YOOxrq`
   * Second comment: `_PyOf3e}`
4. Reconstructed the full candidate string by concatenating both parts:
   ```text
   AhyyOevtva{LOh_YOOxrq_PyOf3e}
   ```

---

## 3. Cryptographic & Cipher Analysis

1. **Prefix Identification (ROT13):**
   * The prefix `AhyyOevtva` follows standard ROT13 rotation:
     $$\text{ROT13}(\texttt{AhyyOevtva}) = \texttt{NullOrigin}$$
2. **Payload Decoding:**
   * Direct ROT13 on the inner text `LOh_YOOxrq_PyOf3e` gave intermediate letters that mapped to a leetspeak phrase with preserved capitalization and numeric substitutions:
     $$\texttt{LOh\_YOOxrq\_PyOf3e} \longrightarrow \texttt{Y0u\_L00ked\_Cl0s3r}$$
3. **Format Normalization:**
   * Applying the platform's standard flag format (`Null0rigin{...}` with `0` replacing `O` in the prefix) yielded the final accepted flag:
     ```text
     Null0rigin{Y0u_L00ked_Cl0s3r}
     ```

---

## 4. Flag

```text
Null0rigin{Y0u_L00ked_Cl0s3r}
```
