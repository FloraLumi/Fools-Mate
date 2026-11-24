# TryHackMe: Fools Mate — Walkthrough

References
https://dhruvrajsinh8.github.io/WRITEUP-CTF/foolsmate/

**Platform:** TryHackMe  
**Room URL:** https://tryhackme.com/room/

**Difficulty:** Easy

---

## 1. Target Information

- **Room Name:** Fools Mate
- **Target IP:** `10.48.158.161`
- **Application:** EndgameTrainer (Node.js)
- **Objective:** Identify why the web application prevents playing the winning checkmate move, bypass the client-side restriction, and retrieve the flag.

**Room Overview**  
<img width="940" height="445" alt="Room Overview" src="https://github.com/user-attachments/assets/b9ebf84a-f9c5-4731-8247-cdfc7c6004b7" />

---

## 2. Methodology

1. **Reconnaissance:** Identifying open ports and running services using `nmap`.
2. **Web Enumeration:** Inspecting the `EndgameTrainer` web application interface and testing the puzzle solution (`Ra8#`).
4. **Client-Side Source Code Analysis:** Reviewing frontend JavaScript files via Firefox Developer Tools (`Debugger`) to locate the move validation logic.
5. **Exploitation (Client-Side Bypass):** Using a conditional breakpoint in the Firefox Debugger to neutralize the client-side checkmate restriction, execute `Ra8#`, and reveal the flag.

---

## 3. Step-by-Step Walkthrough

### Step 1: Reconnaissance
Running an Nmap service-version scan against the target IP (`10.48.158.161`):

```bash
nmap -sV -vv 10.48.158.161
```
#### Open Ports & Services

| Port | State | Service | Version |
| :--- | :--- | :--- | :--- |
| `22/tcp` | open | ssh | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0) |
| `80/tcp` | open | http | Node.js Express framework |

<!-- SCREENSHOT 2: Terminal showing the nmap -sV -vv 10.48.158.161 scan output -->
[Nmap Scan]
<img width="940" height="740" alt="image" src="https://github.com/user-attachments/assets/228706d2-8bc4-4bec-a581-da4e802d40e0" />


---

### Step 2: Enumeration
Navigating to `http://10.48.158.161` in Firefox loads **EndgameTrainer**, a chess puzzle interface displaying a **"Mate-in-one · White to move"** scenario:
* **White:** Rook on `a1`, King on `g1`, Pawns on `f2`, `g2`, `h2`
* **Black:** King on `g8`, Pawns on `f7`, `g7`, `h7`

<!-- SCREENSHOT 3: Initial EndgameTrainer chessboard at http://10.48.158.161 -->
Initial Board
<img width="940" height="624" alt="image" src="https://github.com/user-attachments/assets/604fd1d6-c727-4d3d-86f4-18e1c26d2158" />

The winning move is a classic back-rank checkmate: **Ra8#** (sliding the White Rook from `a1` to `a8`). However, dragging the rook to `a8` triggers a fake system warning dialog (`"I'll shut down your PC if you play that."`) and prevents the move from registering.

<!-- SCREENSHOT 4: Fake error dialog "I'll shut down your PC if you play that." -->
Blocked Move Notice

<img width="711" height="356" alt="image" src="https://github.com/user-attachments/assets/6ecaaa0d-68a0-4712-80d4-b2466eb37c8f" />


---

### Step 3: Source Code Analysis
Opening **Firefox Developer Tools (`F12`) → Debugger** and inspecting `js/app.js` around **lines 257–260** reveals the checkmate-blocking condition inside `preMoveCheck()`:

```javascript
if (result && probe.isCheckmate()) {
  showSystemNotice("I'll shut down your PC if you play that.");
  return false;
}
```

<!-- SCREENSHOT 5: Firefox Debugger showing lines 257-260 in js/app.js -->
[Source Code Analysis]
<img width="914" height="97" alt="image" src="https://github.com/user-attachments/assets/2c7bb0ae-7a64-4479-88f3-47e5183d5468" />



#### Vulnerability Analysis
* The application simulates the attempted move on a temporary board (`probe`) using the current FEN string.
* When `probe.isCheckmate()` evaluates to `true`, it displays the fake system notice and returns `false` on line **259**, dropping the move.
* Because `preMoveCheck()` is scoped inside a closure rather than exposed globally on `window`, and all move validation occurs client-side in browser JavaScript, the check can be bypassed directly in the Debugger.

---

### Step 4: Exploitation
To bypass the checkmate restriction in **Firefox Debugger**:
1. Open `js/app.js` and locate line **257** (`if (result && probe.isCheckmate()) {`).
2. Right-click line **257** in the gutter and select **Add condition** (`Ctrl+Shift+B`).
3. Enter the following expression and press **Enter**:
   ```javascript
   result = null
   ```
4. Drag the White Rook on the board from **`a1` to `a8`**.

<!-- SCREENSHOT 6: Conditional breakpoint (>> result = null) set on line 257 -->
[Debugger Conditional Breakpoint]


<img width="550" height="97" alt="image" src="https://github.com/user-attachments/assets/a2f45b63-68df-4b84-b7d6-b773b9302415" />


Setting `result = null` inside the conditional breakpoint forces the `if` statement on line **257** to evaluate to `false` without pausing execution, skipping `showSystemNotice()` and `return false;` and proceeding directly to `return true;`.

---

### Step 5: Checkmate & Flag Retrieval
With the client-side check bypassed, `Ra8#` registers on the board, the status panel updates to **"Checkmate — White wins"**, and the flag is revealed.

<!-- SCREENSHOT 7: "Checkmate — White wins" panel displaying THM{cl13nt_s1d3_ch3ckm4t3} -->
[Checkmate and Flag]

<img width="400" height="125" alt="image" src="https://github.com/user-attachments/assets/f5d0386c-dc8b-4550-9bd4-581d224bea88" />


---

## 4. Flag

```text
THM{cl13nt_s1d3_ch3ckm4t3}
```

---

## 5. Remediation

* **Enforce Server-Side Validation:** Move legality and win-condition checks must be verified on the backend Node.js/Express server rather than relying solely on client-side JavaScript.
* **Protect Sensitive Rewards:** Flags or completion states should only be issued by a server-side endpoint after independently validating the final game state.
