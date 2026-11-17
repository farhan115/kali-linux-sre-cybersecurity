# Day 09 — Linux Network Troubleshooting

**Author:** Farhan Muntasir  
**GitHub:** [farhan115](https://github.com/farhan115)  
**Environment:** Kali Linux VM in VirtualBox  
**Duration:** Approximately 30 minutes  
**Folder:** `day-09-linux-network-troubleshooting`

## Project overview

This lab investigates Linux connectivity at four levels: interface addressing, routing, name resolution, and application access. A local Python web server provides a controlled application for testing, simulating an outage, and verifying recovery.

## Evidence and current results

The supplied screenshots verify the following results:

| Check | Observed result | Interpretation |
|---|---|---|
| Interface | `eth0` UP, `10.0.2.15/24` | VM has an IPv4 address |
| Default route | `default via 10.0.2.2 dev eth0` | Gateway configured |
| Selected route | `1.1.1.1 via 10.0.2.2 dev eth0 src 10.0.2.15` | External destination uses the expected interface |
| ICMP | 4 transmitted, 4 received, 0% loss | Connectivity to `1.1.1.1` confirmed during the test |
| Ping latency | Average `18.155 ms` | Measured round-trip time for this test |
| Name resolution | `example.com` returned `172.66.147.243` and `104.20.23.154` | System name resolution succeeded; addresses can change |
| Resolver configuration | Includes `75.75.75.75`, `75.75.76.76`, and an IPv6 resolver | Resolver settings inspected |
| HTTPS | `HTTP/2 200` | External HTTPS request succeeded |
| Local application | Expected page text returned; server logged HTTP 200 requests | Local HTTP application worked |
| Active listener | `127.0.0.1:8000` present in `ss -ltn` | TCP listener confirmed while running |
| Stopped application | `curl: (7) Failed to connect` after Ctrl+C | Controlled outage reproduced |
| Listener after shutdown | Port `8000` absent from `ss -ltn` | Application stopped listening |

**Recovery verification remains pending.** The screenshots show successful requests while the server was running and a failure after shutdown. They do not establish a successful request after a restart.

### Incident analysis

The controlled local outage occurred after the Python server was stopped with Ctrl+C. A subsequent request to `127.0.0.1:8000` failed, and socket inspection showed no listener on that port. This identifies the stopped application as the cause of this test failure. Restarting the server and repeating the same request is the remaining recovery check.

The `docker0` interface was DOWN, but the demonstrated external traffic used `eth0` and succeeded. Other listening ports shown in the output are separate from the port 8000 exercise.

## Objectives

1. Inspect interface addresses and the default route.
2. Distinguish name resolution from IP connectivity.
3. Test HTTPS application access.
4. Inspect a listening TCP port.
5. Simulate and recover a local application outage.

## Prerequisites

Use a Linux VM with `ip`, `ping`, `getent`, `ss`, `curl`, and Python 3 available. Internet access is required for the external checks. The local web server exercise works without internet access. No administrator privileges are needed for these lab commands.

## Procedure and command explanations

### 1. Create the workspace

```bash
mkdir -p ~/linux-labs/day-09-linux-network-troubleshooting
cd ~/linux-labs/day-09-linux-network-troubleshooting
```

- **Line 1:** `mkdir` creates directories; `-p` creates missing parents and permits an existing directory. `~` means your home directory.
- **Line 2:** `cd` changes into the lab directory.
- **Why:** Keep lab files organized and separate from other projects.

### 2. Inspect the interface addresses

```bash
ip -br address
```

- `ip` displays network information.
- `-br` produces brief output.
- `address` lists interface addresses.
- **Why:** Confirm that the VM has an address on an interface other than loopback. An enabled interface alone does not prove internet access.

`127.0.0.1` is the local machine's IPv4 loopback address. Other interface names and addresses vary by VM configuration.

### 3. Inspect routing

```bash
ip route
ip route get 1.1.1.1
```

- **Line 1:** Displays the IPv4 routing table. A default route is used when no more specific route matches.
- **Line 2:** Shows the route, interface, and source address Linux would select for this destination.
- **Why:** Missing or incorrect routing can prevent access to remote networks. The second command selects a route without testing actual connectivity.

### 4. Test connectivity without name resolution

```bash
ping -c 4 1.1.1.1
```

- `ping` sends ICMP Echo Requests.
- `-c 4` stops after four requests.
- A numeric destination avoids needing hostname resolution.
- **Why:** Helps isolate connectivity from name resolution. Replies confirm ICMP connectivity to that destination; failure alone is inconclusive because ICMP may be filtered.

### 5. Inspect name resolution

```bash
getent ahostsv4 example.com
cat /etc/resolv.conf
```

- **Line 1:** Queries IPv4 address information using the system's configured name service. This may use DNS, `/etc/hosts`, and other configured sources.
- **Line 2:** Prints resolver configuration.
- **Why:** Applications need to resolve hostnames before connecting. Inspect configuration without changing it.

A resolver address such as `127.0.0.53` can refer to a local resolver and is not automatically a fault.

### 6. Test HTTPS access

```bash
curl -I --connect-timeout 5 --max-time 15 https://example.com
```

| Part | Explanation | Purpose |
|---|---|---|
| `curl` | Makes a network request | Tests application access |
| `-I` | Uses an HTTP HEAD request | Retrieves headers without the page body |
| `--connect-timeout 5` | Limits connection setup to 5 seconds | Bounds connection waiting |
| `--max-time 15` | Limits the whole operation to 15 seconds | Bounds total test time |
| URL | Specifies an HTTPS destination | Tests access to that endpoint |

**Observed:** `HTTP/2 200` in the supplied screenshot. This confirms that the HTTPS endpoint returned a successful HTTP response for that request.

### 7. Create and start the local website

In Terminal 1, from the lab directory:

```bash
mkdir -p web
printf '%s\n' 'Linux Day 09: local web server is working.' > web/index.html
python3 -m http.server 8000 --bind 127.0.0.1 --directory web
```

- **Line 1:** Creates the website directory.
- **Line 2:** `printf` writes the message followed by a newline. `>` creates or overwrites `web/index.html`.
- **Line 3:** `python3 -m http.server` runs Python's HTTP server module. `8000` selects the port, `--bind 127.0.0.1` restricts connections to this machine, and `--directory web` selects the content directory.
- **Why:** A known local application lets you practice diagnosis without depending on an external server.

**Observed:** The screenshot shows `Serving HTTP on 127.0.0.1 port 8000`.

Leave Terminal 1 running. Open Terminal 2 with **Ctrl+Alt+T**, or open another terminal tab with **Ctrl+Shift+T**.

### 8. Verify the socket and application

In Terminal 2:

```bash
ss -ltn
curl --max-time 5 http://127.0.0.1:8000
```

- **Line 1:** `ss` inspects sockets; `-l` shows listeners, `-t` selects TCP, and `-n` uses numeric addresses and ports. Look for `127.0.0.1:8000`.
- **Line 2:** Requests the local web page with a five-second total limit.
- **Why:** The listener confirms a socket is available; the expected page confirms the application serves the intended content.

Observed response in the supplied screenshots:

```text
Linux Day 09: local web server is working.
```

### 9. Simulate an outage

Press **Ctrl+C in Terminal 1** to stop the Python server. In Terminal 2, repeat:

```bash
curl --max-time 5 http://127.0.0.1:8000
ss -ltn
```

- **Line 1:** Tests access after stopping the application. A connection failure is expected if no other service is listening on port 8000.
- **Line 2:** Confirms the port 8000 listener is absent.
- **Why:** Demonstrates an application outage while the machine's network configuration remains available.

**Observed outage:** After Ctrl+C, `curl` reported failure to connect to port 8000. The subsequent socket listing contained no port 8000 listener.

### 10. Recover and verify (pending)

In Terminal 1, from the lab directory:

```bash
python3 -m http.server 8000 --bind 127.0.0.1 --directory web
```

This restarts the same local service. In Terminal 2:

```bash
curl --max-time 5 http://127.0.0.1:8000
```

Receiving the expected page verifies recovery. Stop the server with **Ctrl+C** when finished.

## Validation checklist

- [x] Record the VM's interface address.
- [x] Record the default route and selected path to an external IP.
- [x] Record the ICMP result, including any filtering limitation.
- [x] Record the hostname resolution result.
- [x] Receive an external HTTPS response (`HTTP/2 200` observed).
- [x] Start the local Python HTTP server (startup observed).
- [x] Confirm the local TCP listener and page content.
- [x] Stop the server and capture the failed request.
- [ ] Restart the server and capture the successful request.

## Troubleshooting reference

| Symptom | Possible explanation | Next check |
|---|---|---|
| No non-loopback IPv4 address | VM adapter disconnected or address assignment failed | Inspect VM adapter settings and interface state |
| No default route | Gateway configuration missing | Inspect network configuration |
| Numeric IP responds but hostname lookup fails | Name resolution problem | Check `getent` output and resolver configuration |
| Ping fails but HTTPS works | ICMP may be filtered | Use application tests to assess the required service |
| `curl: command not found` | Tool is unavailable | Check prerequisites; this does not prove a network fault |
| Local connection refused | No listener or an active rejection | Inspect `ss -ltn` and server terminal |
| Address already in use | Port is occupied | Inspect listeners; use another test port consistently if needed |
| HTTP 404 | Requested path does not exist | Check the served directory and filename |
| Unexpected directory listing | Default page missing from served directory | Confirm `web/index.html` exists |
| HTTPS certificate error | Clock, trust, certificate, or interception problem | Read the error and check system time and trust configuration |

## Interview questions and answers

**1. What is the difference between DNS resolution and connectivity?**  
Name resolution maps a hostname to an address. Connectivity concerns reaching a destination. Successful resolution does not guarantee a working connection.

**2. What is a default gateway?**  
A router used to reach destinations when no more specific route matches.

**3. Why use a numeric IP for a connectivity test?**  
It avoids requiring hostname resolution for that test.

**4. Why can ping fail while a website works?**  
ICMP can be filtered independently of HTTPS traffic.

**5. What does a listening TCP port prove?**  
A socket is waiting for connections at that address and port. It does not guarantee a correct application response.

**6. Why use both `ss` and `curl`?**  
`ss` inspects listening sockets. `curl` verifies the application's response.

**7. Why use two terminals?**  
One holds the foreground server process while the other runs client tests.

**8. What does binding to `127.0.0.1` mean?**  
The server accepts connections through the machine's loopback interface. Other machines cannot reach that listener directly.

**9. What is the difference between HTTP 404 and connection refused?**  
A 404 is an HTTP response from a reachable server about a missing resource. Connection refused happens before an HTTP response is received.

**10. How do you confirm recovery?**  
Repeat the same client request after restarting the service and verify the expected content.

## Portfolio wording

**Supported by current evidence:** Validated Linux interface addressing, routing, name resolution, and HTTPS access; confirmed 0% packet loss across 4 ICMP requests and diagnosed 1 simulated local HTTP service outage using client requests and TCP socket inspection.

**Use after completing all checks:** Investigated Linux addressing, routing, name resolution, and HTTP connectivity; simulated and recovered 1 local web service outage using socket inspection and client response verification.

## GitHub description

Hands-on Linux networking lab covering IP addresses, routing, DNS resolution, HTTPS checks, TCP listeners, and local web server outage recovery with explanations and troubleshooting Q&A.
