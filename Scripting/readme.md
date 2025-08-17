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













