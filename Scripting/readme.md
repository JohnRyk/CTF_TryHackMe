https://tryhackme.com/room/scripting

# [Easy] Base64


# [Medium] Gotta Catch em All

You need to write a script that connects to this webserver on the correct port, do an operation on a number and then move onto the next port. Start your original number at 0.

The format is: operation, number, next port.

For example the website might display, add 900 3212 which would be: add 900 and move onto port 3212.

Then if it was minus 212 3499, you'd minus 212 (from the previous number which was 900) and move onto the next port 3499

Do this until you the page response is STOP (or you hit port 9765).

Each port is also only live for 4 seconds. After that it goes to the next port. You might have to wait until port 1337 becomes live again...

Go to: http://<machines_ip>:3010 to start...

General Approach(it's best to do this using the sockets library in Python):

Create a socket in Python using the sockets library
Connect to the port 
Send an operation
View response and continue



```python
import requests
import time
import sys

def solve_webserver_challenge():
    """
    Automatically solve the web server challenge by following port sequences and performing operations.
    Returns the final number rounded to 2 decimal places after all operations.
    """
    base_url = "http://10.201.71.240:"
    current_port = int(sys.argv[1])  # Starting port
    current_number = 0.0  # Initial number (float for decimal operations)
    timeout = 2          # Connection timeout
    max_attempts = 3     # Max retries per port

    try:
        while True:
            print(f"\nCurrent number: {current_number:.2f} | Connecting to port {current_port}")

            # Special handling for port 9765 (flag port)
            if current_port == 9765:
                print("Reached the flag port (9765)...")
                for attempt in range(max_attempts * 2):
                    try:
                        response = requests.get(f"{base_url}{current_port}", timeout=timeout)
                        if response.status_code == 200:
                            flag = response.text.strip()
                            print(f"Flag received: {flag}")
                            print(f"Final number after all operations: {round(current_number, 2)}")
                            return flag, round(current_number, 2)
                        break
                    except requests.exceptions.RequestException as e:
                        print(f"Flag port attempt {attempt + 1} failed: {e}")
                        time.sleep(1)
                print("Failed to get flag from port 9765")
                return None, round(current_number, 2)

            # Normal port handling
            response = None
            for attempt in range(max_attempts):
                try:
                    response = requests.get(f"{base_url}{current_port}", timeout=timeout)
                    if response.status_code == 200:
                        break
                except requests.exceptions.RequestException as e:
                    print(f"Attempt {attempt + 1} failed: {e}")
                    if attempt < max_attempts - 1:
                        time.sleep(1)

            if not response or response.status_code != 200:
                print(f"Could not connect to port {current_port}")
                current_port += 1  # Fallback: try next port
                if current_port > 9765:
                    current_port = 9872  # Wrap around
                continue

            content = response.text.strip()
            print(f"Server response: {content}")

            if content == "STOP":
                print(f"Challenge completed at port {current_port}")
                final_number = round(current_number, 2)
                print(f"Final number: {final_number}")
                return None, final_number

            # Parse instruction (format: "operation number next_port")
            try:
                operation, num_str, next_port_str = content.split()
                num = float(num_str)  # Using float for all operations
                next_port = int(next_port_str)

                # Perform operation while preserving float precision
                prev_number = current_number
                if operation == "add":
                    current_number += num
                elif operation == "minus":
                    current_number -= num
                elif operation == "multiply":
                    current_number *= num
                elif operation == "divide":
                    current_number /= num  # Regular division for decimal precision
                else:
                    raise ValueError(f"Unknown operation: {operation}")

                # Show full precision during calculations, but indicate rounding will happen
                print(f"Operation: {operation} {num} ({prev_number} Â {current_number})")
                print(f"Interim result (will be rounded to 2 decimals at end): {current_number}")
                print(f"Next port: {next_port}")
                
                current_port = next_port
                time.sleep(4)  # Wait for port availability

            except (ValueError, ZeroDivisionError) as e:
                print(f"Error processing instruction: {e}")
                current_port += 1  # Fallback: try next port

    except KeyboardInterrupt:
        print("\nScript interrupted by user")
        return None, round(current_number, 2)

if __name__ == "__main__":
    flag, final_number = solve_webserver_challenge()
    print("\n=== Final Results ===")
    if flag:
        print(f"FLAG: {flag}")
    print(f"Final calculated number (rounded to 2 decimal places): {final_number:.2f}")
```

Start

```shell
python3 exp_ctf_2.py $(curl -s 'http://10.201.6.34:3010/' | egrep -o 'onPort">.*?</a' | egrep -o '[0-9]+')
```

<img width="1920" height="829" alt="image" src="https://github.com/user-attachments/assets/5e85dbb3-1d33-4aa1-9c61-4d67fb218d3d" />

...

<img width="1917" height="814" alt="image" src="https://github.com/user-attachments/assets/a5186c5c-58b0-4314-bd78-52c1d399d4f1" />




## Port-Hopping Arithmetic

> **Challenge:** Connect to a web server, hop from port to port applying arithmetic
> operations to a running total (starting at `0`), and stop when the response is
> `STOP` or you reach port `9765`. Each port is only *live* for ~4 seconds.

---

### 1. The brief

The landing page at `http://10.144.157.58:3010` explains the game:

> The format is: *operation, number, next port.*
> For example the website might display **`add 900 3212`** → add `900` and move onto port `3212`.
> Then if it was **`minus 212 3499`**, you'd minus `212` (from `900`) and move onto port `3499`.
> Do this until the response is `STOP` (or you hit port `9765`).
> Each port is also only live for **4 seconds**. After that it goes to the next port.
> You might have to wait until port `1337` becomes live again…

The suggested approach is the Python `socket` library:

1. Create a socket
2. Connect to the port
3. "Send an operation"
4. View the response and continue

---

### 2. Reconnaissance — figuring out the mechanics

#### 2.1 Are these HTTP servers or raw TCP?

I first tried a plain `socket.connect()` to port `1337`. The connect succeeded (when
the port was live) but `recv()` **timed out** — the server was waiting for *us* to speak
first. So I sent a minimal HTTP request:

```python
s.sendall(b"GET / HTTP/1.1\r\nHost: 10.144.157.58\r\nConnection: close\r\n\r\n")
```

…and got back a proper HTTP response:

```
HTTP/1.0 200 OK
Server: Werkzeug/0.14.1 Python/3.5.2
Content-Length: 13

add 900 23456
```

So each port is a tiny **Werkzeug/Flask** HTTP server whose body is exactly
`<operation> <number> <next_port>`.

#### 2.2 How does the "live window" actually behave?

This is the real twist. I scraped the landing page's
`<a id="onPort">…</a>` element over a couple of minutes and logged which port was
"currently live":

```
0.8s  23456     60.1s 63890    120.4s 4040
5.8s  8888      66.1s 38721    126.6s 5050
13.1s 9823      72.4s 6632     132.9s 9898
19.3s 9887      78.7s 29932    138.8s 3232
23.5s 7823      84.2s 29132    143.9s 10321
30.8s 10456     90.5s 8773     149.9s 7709
35.8s 10457     96.8s 1338     155.9s 9872
42.1s 40000     101.9s 1876    162.2s 32424
48.3s 40200     109.3s 34232   169.5s 65513
54.6s 8743      114.3s 6783    ...
```

Two crucial facts jumped out:

* **The "live" rotation follows the chain exactly.** `1337`'s next port is `23456`,
  and `23456` was the *very first* port observed — immediately followed by `8888`,
  then `9823`, … matching the `next_port` links returned by each server.
* **Only one port is open at a time**, for ~5 seconds each. With ~34 ports in the
  chain the full cycle is **>200 seconds**, which is why a naive "wait for 1337"
  loop with a 2-minute timeout silently failed.

#### 2.3 The winning strategy

Because the live window *walks the chain*, if I can grab `1337` during its window and
then immediately follow each `next_port`, I ride the wave: by the time I finish reading
port *P*, port *P+1* is just opening. No waiting a full cycle — just stay fast and in
sync.

---

### 3. The solver

```python
#!/usr/bin/env python3
"""
Port-Hopping Solver
- Start at port 1337 with total = 0.
- Each port (a tiny Werkzeug HTTP server) replies: "<op> <num> <next_port>".
- Apply op to the running total, hop to next_port.
- Stop on "STOP" or when next_port == 9765.
- Each port is live ~4-5s, so poll patiently and stay in sync with the rotation.
"""
import socket, time, json

HOST       = "10.144.157.58"
START_PORT = 1337
STOP_PORT  = 9765


def fetch(port, timeout=8):
    """Poll until `port` is live, send a GET, return the response body."""
    deadline = time.time() + timeout
    last_err = None
    while time.time() < deadline:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(2)
        try:
            s.connect((HOST, port))
            s.sendall(b"GET / HTTP/1.1\r\nHost: " + HOST.encode() +
                      b"\r\nConnection: close\r\n\r\n")
            data = b""
            try:
                while True:
                    chunk = s.recv(4096)
                    if not chunk:
                        break
                    data += chunk
            except socket.timeout:
                pass
            s.close()
            text = data.decode(errors="replace")
            body = text.split("\r\n\r\n", 1)[1] if "\r\n\r\n" in text else text
            return body.strip()
        except (ConnectionRefusedError, socket.timeout, OSError) as e:
            last_err = e
            s.close()
            time.sleep(0.1)           # port not open yet — retry fast
    raise TimeoutError(f"port {port} never became live ({last_err})")


def apply_op(total, op, num):
    return {
        "add":      total + num,
        "minus":    total - num,
        "multiply": total * num,
        "divide":   total / num,
    }[op.lower()]


def main():
    total, port, steps, seen = 0.0, START_PORT, [], set()

    # Wait a full cycle (~200s+) for 1337 to appear.
    text = fetch(START_PORT, timeout=420)

    step = 0
    while True:
        if text.upper() == "STOP":
            break
        op, num_s, nxt_s = text.split()
        num, nxt = float(num_s), int(nxt_s)

        total = apply_op(total, op, num)
        step += 1
        seen.add(port)
        print(f"[{step:3d}] {port:6d}: {op} {num_s} -> {nxt:6d} | total = {total}")
        steps.append({"step": step, "port": port, "op": op,
                      "num": num_s, "next": nxt, "total": total})

        if nxt == STOP_PORT:
            try:                       # confirm STOP on 9765
                print("   ", STOP_PORT, "->", fetch(STOP_PORT, timeout=10))
            except Exception as e:
                print("   ", e)
            break

        port = nxt
        text = fetch(port, timeout=40)  # next window opens right after this one

    print("\nFinal total :", total)
    print("Rounded (2dp):", round(total, 2))
    json.dump({"total": total, "rounded": round(total, 2), "steps": steps},
              open("path_log.txt", "w"), indent=2)


if __name__ == "__main__":
    main()
```
---

### 4. The run

The script caught `1337`, then rode the rotation cleanly through all 34 ports:

```
[  1]   1337: add 900      ->  23456 | total = 900.0
[  2]  23456: minus 2      ->   8888 | total = 898.0
[  3]   8888: multiply 4   ->   9823 | total = 3592.0
[  4]   9823: divide 2     ->   9887 | total = 1796.0
[  5]   9887: add 456      ->   7823 | total = 2252.0
[  6]   7823: minus 43     ->  10456 | total = 2209.0
[  7]  10456: divide 0.5   ->  10457 | total = 4418.0
[  8]  10457: add 877      ->  40000 | total = 5295.0
[  9]  40000: add 9        ->  40200 | total = 5304.0
[ 10]  40200: multiply 34  ->   8743 | total = 180336.0
[ 11]   8743: minus 5      ->  63890 | total = 180331.0
[ 12]  63890: divide 1     ->  38721 | total = 180331.0
[ 13]  38721: add 43       ->   6632 | total = 180374.0
[ 14]   6632: minus 22     ->  29932 | total = 180352.0
[ 15]  29932: divide 3     ->  29132 | total = 60117.333…
[ 16]  29132: multiply 9   ->   8773 | total = 541056.0
[ 17]   8773: minus 1200   ->   1338 | total = 539856.0
[ 18]   1338: add 12       ->   1876 | total = 539868.0
[ 19]   1876: divide 1     ->  34232 | total = 539868.0
[ 20]  34232: multiply 1   ->   6783 | total = 539868.0
[ 21]   6783: add -100     ->   4040 | total = 539768.0
[ 22]   4040: multiply -2  ->   5050 | total = -1079536.0
[ 23]   5050: add 300      ->   9898 | total = -1079236.0
[ 24]   9898: divide 10    ->   3232 | total = -107923.6
[ 25]   3232: minus 10     ->  10321 | total = -107933.6
[ 26]  10321: add 130      ->   7709 | total = -107803.6
[ 27]   7709: multiply 4   ->   9872 | total = -431214.4
[ 28]   9872: multiply -12 ->  32424 | total = 5174572.8
[ 29]  32424: divide 3     ->  65513 | total = 1724857.6
[ 30]  65513: minus 1000   ->   3459 | total = 1723857.6
[ 31]   3459: add 23       ->   7832 | total = 1723880.6
[ 32]   7832: divide 5     ->   1111 | total = 344776.12
[ 33]   1111: minus 8      ->   2222 | total = 344768.12
[ 34]   2222: add 1        ->   9765 | total = 344769.12
    9765 -> STOP
```

---

### 5. Lessons learned

1. **Read the hint literally.** The challenge *said* use the `socket` library, and the
   ports really are plain TCP endpoints — but they speak HTTP. A first `GET /` request
   was all it took.
2. **Understand the timing model before brute-forcing.** My initial 2-minute wait for
   `1337` failed because the cycle is **>200 seconds**. Logging the live-port sequence
   revealed that the rotation *follows the chain*, which makes the whole thing solvable
   in a single pass.
3. **Ride the wave.** Because each window opens immediately after the previous port's,
   staying fast keeps you perfectly in sync — no need to wait multiple cycles.
4. **Watch out for negative numbers & division.** Several steps use *negative*
   operands (`add -100`, `multiply -12`) and division by fractions (`divide 0.5`), so
   the accumulator must be a `float` and operations must be applied left-to-right in
   order — there's no operator precedence to worry about, just sequential accumulation.



# [Hard] Encrypted Server Chit Chat


## Breaking AES-GCM Over UDP

> **Target:** `10.146.155.58` UDP port `4000`

### Challenge Description

A UDP server is running on port 4000. We are told to:

1. Send a UDP packet with the payload `"hello"` to receive instructions.
2. The server uses **AES-GCM** encryption.
3. Recover the flag using the information the server hands back.

The challenge drops a few breadcrumbs worth keeping in mind:

* Network I/O is done in **bytes**.
* PyCA's `cryptography` library takes its inputs as **bytes**.
* AES-GCM produces a **ciphertext *and* an authentication tag**, and the server
  sends them sequentially: *encrypted plaintext first, then the tag*.

---

### Reconnaissance — Talking to the Server

The protocol is a simple three-step handshake. First we send `hello`:

```text
$ python3 solve.py
[*] hello  -> You've connected to the super secret server,
              send a packet with the payload ready to receive more information
```

Following the instruction, we send `ready` and get the gold:

```text
[*] ready  -> key:thisisaverysecretkeyl337 iv:secureivl337
              to decrypt and find the flag that has a SHA256 checksum of
              5d77f018d2bf777860548655d84d7382dc27d6ce816ede68f65d72621463d9da
              send final in the next payload to receive all the encrypted flags
```

So we now have everything we need:

| Field | Value | Notes |
|-------|-------|-------|
| **Key** | `thisisaverysecretkeyl337` | 24 bytes → **AES-192** |
| **IV / Nonce** | `secureivl337` | 12 bytes (the standard GCM nonce size) |
| **Target SHA256** | `5d77f018...21463d9da` | The checksum of the *real* flag |

Finally we send `final`, and the server blasts back **32 UDP packets**.

---

### Analysis — Spotting the Nonce Reuse

When we print every packet in hex, a pattern jumps out immediately:

```text
Packet  1 | len= 13 | 68093ae9a389abce773a2cde1d
Packet  2 | len= 16 | 53336efe7d78675a3cdbbc2874b6b6c1
Packet  3 | len= 16 | 9f61571ece8099b10e69f49f7e83ca8a
Packet  4 | len= 13 | 68093ae9a99dbad2612d47c21d
Packet  5 | len= 13 | 68093ae9bf94ccd368352bcf1d
Packet  6 | len= 16 | 208ee28eb1335615cca6d2b112c4b201
...
```

Half the packets are **13 bytes** (plus one 28-byte one) and all of them start
with the identical prefix `68093ae9`. The other half are **16 bytes** with
random-looking data.

This is the smoking gun for **AES-GCM nonce reuse**. GCM is built on CTR mode,
so `ciphertext = plaintext XOR keystream`. When the **same key + nonce** encrypt
multiple plaintexts, the *same keystream* is reused, and every plaintext that
shares a prefix produces a ciphertext that shares a prefix.

All of the flags are `THM{...}`, so:

```text
keystream[0:4] = ciphertext[0:4] XOR "THM{"
               = 68 09 3a e9  XOR  54 48 4d 7b
               = 3c 41 77 92
```

That explains why every ciphertext begins with `68093ae9`.

### Splitting ciphertexts from tags

We now know how to classify the 32 packets:

* **Ciphertexts** → start with `68093ae9` (16 of them).
* **Tags** → the remaining 16-byte packets (16 of them).

Because UDP does **not** guarantee ordering, the packets arrived shuffled, so we
can't blindly assume "the next packet is the matching tag". Instead we
brute-force every `(ciphertext, tag)` pairing and let AES-GCM's built-in
authentication tell us which pairs are valid.

---

### The Solve Script

```python
#!/usr/bin/env python3
"""
CTF solver for the AES-GCM nonce-reuse UDP challenge.

Server: 10.146.155.58:4000 (UDP)
Protocol:
    1. send "hello"  -> greeting
    2. send "ready"  -> key, iv (nonce) and the SHA256 checksum of the real flag
    3. send "final"  -> the server streams back many encrypted flags, each as a
                        pair of UDP packets: ciphertext followed by tag.

The server reuses the same key+nonce for every flag, but the real flag is the
one whose decrypted plaintext has the advertised SHA256 checksum. UDP re-ordering
scrambles which ciphertext belongs to which tag, so we brute-force every
(ciphertext, tag) pair, let AES-GCM authenticate each one, and keep the plaintext
whose SHA256 matches.
"""

import hashlib
import re
import socket

from cryptography.hazmat.primitives.ciphers.aead import AESGCM

SERVER_ADDR = ("10.146.155.58", 4000)
TARGET_SHA256 = "5d77f018d2bf777860548655d84d7382dc27d6ce816ede68f65d72621463d9da"


def collect_packets():
    """Run the hello/ready/final handshake and gather every UDP packet."""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.settimeout(15)
    try:
        sock.sendto(b"hello", SERVER_ADDR)
        greeting = sock.recvfrom(65535)[0]
        print("[*] hello  ->", greeting.decode("utf-8", "replace"))

        sock.sendto(b"ready", SERVER_ADDR)
        ready = sock.recvfrom(65535)[0]
        print("[*] ready  ->", ready.decode("utf-8", "replace"))

        sock.sendto(b"final", SERVER_ADDR)

        packets = []
        while True:
            try:
                packets.append(sock.recvfrom(65535)[0])
            except socket.timeout:
                break
        return packets, ready
    finally:
        sock.close()


def main():
    packets, ready = collect_packets()

    # Parse the key and iv advertised by the server.
    text = ready.decode("utf-8", "replace")
    key = re.search(r"key:(\S+)", text).group(1).encode()
    iv = re.search(r"iv:(\S+)", text).group(1).encode()
    print(f"\n[*] key  = {key!r} ({len(key)} bytes -> AES-{len(key)*8})")
    print(f"[*] iv   = {iv!r} ({len(iv)} bytes)")
    print(f"[*] target SHA256 = {TARGET_SHA256}")

    # Ciphertexts all start with the same 4 bytes (they encrypt "THM{" under a
    # reused nonce), so we can split the stream into ciphertexts and tags even
    # though UDP scrambled their order.
    prefix = packets[0][:4]
    ciphertexts = [p for p in packets if p[:4] == prefix]
    tags = [p for p in packets if p[:4] != prefix]
    print(f"\n[*] {len(ciphertexts)} ciphertexts, {len(tags)} candidate tags")

    aesgcm = AESGCM(key)
    candidates = []

    for ct in ciphertexts:
        for tag in tags:
            try:
                pt = aesgcm.decrypt(iv, ct + tag, None)
            except Exception:
                continue  # tag did not authenticate -> wrong pairing
            candidates.append(pt)

    print(f"[*] authenticated {len(candidates)} plaintext(s):\n")
    for pt in candidates:
        digest = hashlib.sha256(pt).hexdigest()
        marker = "  <-- MATCH" if digest == TARGET_SHA256 else ""
        print(f"    {pt.decode('utf-8', 'replace')!r}  sha256={digest}{marker}")

    flag = next(
        pt for pt in candidates if hashlib.sha256(pt).hexdigest() == TARGET_SHA256
    )
    print(f"\n[+] FLAG: {flag.decode()}")


if __name__ == "__main__":
    main()
```

---

### How It Works

1. **UDP handshake** — `socket.SOCK_DGRAM` lets us fire off the `hello` →
   `ready` → `final` payloads and read back the raw bytes the server emits.
2. **Parse key/iv** — a quick regex pulls `thisisaverysecretkeyl337` and
   `secureivl337` straight out of the `ready` reply (everything is `bytes`,
   matching what `AESGCM` expects).
3. **Separate ciphertexts and tags** — thanks to the nonce-reuse signature,
   every ciphertext shares the `68093ae9` prefix; the rest are 16-byte tags.
4. **Authenticate + decrypt** — `AESGCM.decrypt(nonce, ciphertext + tag, data=None)`
   verifies the GHASH tag itself and raises `InvalidTag` on a mismatch, so it
   doubles as an oracle that tells us which ciphertext belongs to which tag —
   no fragile ordering assumptions required.
5. **Pick the winner** — of the 16 successfully decrypted flags, exactly one has
   a SHA256 equal to the advertised checksum.

---

### Key Takeaways

* **Never reuse a nonce in AES-GCM.** GCM is CTR mode + a MAC; reusing the nonce
  leaks the keystream (letting you spot shared plaintext prefixes) and, far more
  seriously, lets an attacker recover the GHASH authentication key and forge tags.
* **UDP is unordered.** The challenge deliberately interleaves ciphertexts and
  tags, so relying on packet order is a trap — rely on the cryptography itself
  (tag verification) to match them up.
* **Work in bytes end-to-end.** The challenge's hints are literal: sockets send
  bytes, `cryptography` wants bytes, and the SHA256 checksum is even delivered
  as raw bytes in the middle of a text payload.





