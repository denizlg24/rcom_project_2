# FTP Client Implementation - Complete Codebase Explanation

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture and Module Structure](#architecture-and-module-structure)
3. [Module Breakdown](#module-breakdown)
4. [FTP Protocol Implementation](#ftp-protocol-implementation)
5. [Data Flow and Execution](#data-flow-and-execution)
6. [Key Technical Concepts](#key-technical-concepts)

---

## Project Overview

This project implements a command-line FTP (File Transfer Protocol) client in C that downloads files from FTP servers. The client supports:

- URL parsing for FTP addresses with optional authentication
- DNS hostname resolution
- FTP passive mode data transfer
- Real-time download progress with speed monitoring
- Binary file transfers

**Command usage:**
```bash
./build/ftp_client ftp://[user:password@]host/path/to/file
```

---

## Architecture and Module Structure

The codebase is organized into 5 modules, each handling a specific aspect of the FTP client functionality:

### File Structure

```
src/
├── ftp_client.c       # Main program logic and download orchestration
├── ftp_url.c          # URL parsing (extract user, password, host, path)
├── ftp_resolver.c     # DNS hostname resolution
├── ftp_control.c      # FTP control connection and command handling
├── ftp_data.c         # FTP data connection for file transfer
└── includes/
    ├── ftp_url.h
    ├── ftp_resolver.h
    ├── ftp_control.h
    └── ftp_data.h
```

### Module Dependencies

```
ftp_client.c
    ├── ftp_url.c       (URL parsing)
    ├── ftp_resolver.c  (DNS resolution)
    ├── ftp_control.c   (control commands)
    └── ftp_data.c      (data transfer)
```

---

## Module Breakdown

### 1. URL Parsing Module (ftp_url.c)

#### Purpose
Parses FTP URLs into their constituent components for connection and authentication.

#### Data Structure

```c
typedef struct {
  char *user;      // Username for authentication
  char *password;  // Password for authentication
  char *host;      // Server hostname or IP
  char *path;      // File path on server
} ftp_url_t;
```

#### Key Function: `parse_ftp_url()`

**Input:** `ftp://[user:password@]host/path/to/file`

**Parsing Logic:**

1. **Protocol validation:** Checks if URL starts with `ftp://`
2. **Authentication extraction:**
   - Searches for `@` character to identify credentials
   - If found with `:`, splits into user and password
   - If only username provided, password is empty string
   - If no `@`, defaults to `anonymous:anonymous`
3. **Host extraction:**
   - Finds first `/` after credentials
   - Everything before `/` is the hostname
4. **Path extraction:**
   - Everything after first `/` is the file path
   - If no `/`, path is empty string

**Example parsing:**
```
Input:  ftp://user:pass@ftp.example.com/pub/file.txt
Output: user="user", password="pass", host="ftp.example.com", path="pub/file.txt"

Input:  ftp://ftp.example.com/file.txt
Output: user="anonymous", password="anonymous", host="ftp.example.com", path="file.txt"
```

#### Memory Management

The `free_ftp_url()` function deallocates all dynamically allocated strings in the structure. This must be called before program termination to prevent memory leaks.

---

### 2. DNS Resolution Module (ftp_resolver.c)

#### Purpose
Converts hostnames to IP addresses using DNS lookup.

#### Key Function: `resolve_hostname()`

**Process:**

1. Calls `gethostbyname()` system function
   - This queries DNS servers to resolve the hostname
   - Returns a `hostent` structure containing address information
2. Extracts first IPv4 address from results
   - `h_addr_list[0]` contains the primary IP address
3. Converts binary IP to string format using `inet_ntop()`
   - Binary format: 4 bytes representing IP octets
   - String format: "192.168.1.1"

**Why this is needed:**
FTP servers are typically accessed via hostname (e.g., `ftp.example.com`), but TCP sockets require IP addresses for connection. This module bridges that gap.

**Error handling:**
- Returns -1 if DNS lookup fails (hostname doesn't exist, DNS server unreachable, etc.)
- Returns 0 on success

---

### 3. Control Connection Module (ftp_control.c)

This is the core module handling all FTP protocol communication on the control channel.

#### 3.1 Connection Establishment: `ftp_connect()`

**Purpose:** Creates TCP connection to FTP server on port 21 (standard FTP control port).

**Process:**

1. **Socket creation:** `socket(AF_INET, SOCK_STREAM, 0)`
   - `AF_INET`: IPv4 protocol
   - `SOCK_STREAM`: TCP connection (reliable, ordered)
2. **Address structure setup:**
   ```c
   struct sockaddr_in addr;
   addr.sin_family = AF_INET;           // IPv4
   addr.sin_port = htons(port);         // Port 21, converted to network byte order
   inet_pton(AF_INET, ip, &addr.sin_addr); // Convert IP string to binary
   ```
3. **Connection:** `connect()` establishes TCP handshake with server

**Returns:** Socket file descriptor (integer) used for all subsequent control operations, or -1 on error.

#### 3.2 Reading Server Replies: `ftp_read_line()` and `ftp_read_reply()`

**FTP Reply Format:**
```
<code><separator><text>\r\n
```
- Code: 3-digit number (e.g., 220, 331, 230)
- Separator: Either ' ' (space) for single-line or '-' (dash) for multi-line
- Text: Human-readable message
- Line ending: Always CR+LF (`\r\n`)

**Single-line reply example:**
```
220 Welcome to FTP server\r\n
```

**Multi-line reply example:**
```
220-Welcome to FTP server\r\n
220-This is line 2\r\n
220 This is the final line\r\n
```

**`ftp_read_line()` Implementation:**

Reads one character at a time from the socket until finding `\r\n`:
```c
while (reading) {
    recv(sockfd, &c, 1, 0);  // Read 1 byte
    buf[idx++] = c;
    if (buf[idx-2] == '\r' && buf[idx-1] == '\n') {
        buf[idx-2] = 0;  // Replace \r with null terminator
        return strdup(buf);  // Return copy of line
    }
}
```

**`ftp_read_reply()` Implementation:**

Handles both single-line and multi-line responses:

1. Read first line and extract code
2. Check character at position 3:
   - If `' '` (space): single-line, return immediately
   - If `'-'` (dash): multi-line, continue reading
3. For multi-line: keep reading lines until finding `<code> <text>`
   - Example: Keep reading until line starts with "220 "
4. Concatenate all lines into single buffer
5. Return the complete reply and status code

**Dynamic buffer management:**
- Starts with 4KB buffer
- Doubles capacity when needed (`realloc()`)
- Prevents buffer overflow for large replies

#### 3.3 Sending Commands: `ftp_send_cmd()` and `ftp_send_cmd_get_reply()`

**`ftp_send_cmd()`:**
Simple wrapper around `send()` that transmits command string to server.

**`ftp_send_cmd_get_reply()`:**
High-level function combining command sending and reply reading:

1. Format command using `vsnprintf()` (supports printf-style formatting)
2. Append `\r\n` to command (FTP protocol requirement)
3. Send command via `ftp_send_cmd()`
4. Read server reply via `ftp_read_reply()`
5. Print reply to console for user visibility
6. Return status code

**Usage example:**
```c
char *reply;
int code = ftp_send_cmd_get_reply(control, &reply, "USER %s", username);
// Sends: "USER anonymous\r\n"
// Receives and parses reply, returns code (e.g., 331)
```

#### 3.4 PASV Response Parsing: `ftp_parse_pasv()`

**Purpose:** Extract IP address and port number from PASV command response.

**PASV response format:**
```
227 Entering Passive Mode (h1,h2,h3,h4,p1,p2)
```

**Parsing process:**

1. Find opening parenthesis `(`
2. Parse 6 comma-separated integers
3. Reconstruct IP: `h1.h2.h3.h4`
4. Calculate port: `p1 * 256 + p2`

**Example:**
```
Input:  227 Entering Passive Mode (193,137,29,15,198,138)
Output: ip="193.137.29.15", port=50826 (because 198*256+138=50826)
```

**Why this format?**
FTP predates common IP:port notation. The 6-number format allows encoding both IP and port in a way that works with the protocol's comma-separated value system.

#### 3.5 File Size Parsing: `ftp_parse_file_size()`

**Purpose:** Extract file size from RETR command response for progress tracking.

**Response format:**
```
150 Opening BINARY mode data connection for README (1328 bytes)
```

**Parsing:**
1. Find opening parenthesis `(`
2. Use `sscanf()` with pattern `"(%lld bytes)"`
3. Extract size as `long long` (supports files over 2GB)
4. Return -1 if pattern not found (server doesn't provide size)

**Usage:**
If size is available, client can show progress bar. If not, client shows only transferred bytes and speed.

#### 3.6 Disconnection: `ftp_quit()`

Sends QUIT command and expects 221 response code:
```
Client: QUIT\r\n
Server: 221 Goodbye\r\n
```

Returns 0 if successful (code 221), -1 otherwise.

---

### 4. Data Connection Module (ftp_data.c)

#### Purpose
Establishes separate TCP connection for actual file data transfer (FTP uses two connections: control and data).

#### Key Function: `ftp_open_data_connection()`

**Process identical to control connection, but:**
- Uses IP and port from PASV response (not the original server)
- Server listens on this ephemeral port for client connection
- Used exclusively for file data transfer

**Why separate connection?**
- FTP protocol design: control commands and data are isolated
- Allows parallel operations (theoretically)
- Data connection closes after transfer; control persists

---

### 5. Main Program (ftp_client.c)

#### 5.1 Helper Functions

**`ftp_basename()`**

Extracts filename from full path:
```c
Input:  "pub/files/document.pdf"
Output: "document.pdf"
```

Uses `strrchr()` to find last `/` character, returns everything after it. If no `/` found, returns entire string.

**`print_transfer_status()`**

Displays real-time download progress. Two modes:

**Mode 1: File size known (from RETR reply)**
```
[=========>                              ] 25.3% 5.12/20.48 MB - 2.34 MB/s
```
- Progress bar: 40 characters wide
- Percentage: `(bytes_received / total_size) * 100`
- Current/total size in MB
- Current speed in MB/s

**Mode 2: File size unknown**
```
Received: 5.12 MB - Speed: 2.34 MB/s
```

**Speed calculation:**
```c
speed = (bytes_received / 1024 / 1024) / elapsed_time
```
Converts bytes to megabytes, divides by seconds elapsed.

**Progress update frequency:**
Updates approximately every 100KB of data received:
```c
if (total_bytes % (100 * 1024) < sizeof(buf))
```
This prevents excessive terminal updates (performance optimization).

**Terminal handling:**
Uses `\r` (carriage return) to overwrite same line repeatedly, and `fflush(stdout)` to force immediate display.

#### 5.2 Main Program Flow

**Step 1: Command-line argument validation**
```c
if (argc != 2) {
    // Print usage and exit
}
```
Requires exactly one argument: the FTP URL.

**Step 2: URL parsing**
```c
ftp_url_t url;
parse_ftp_url(argv[1], &url);
```
Extracts user, password, host, and path from URL.

**Step 3: DNS resolution**
```c
char ip[64];
resolve_hostname(url.host, ip, sizeof(ip));
```
Converts hostname to IP address (e.g., `ftp.example.com` → `192.0.2.1`).

**Step 4: Control connection establishment**
```c
int control = ftp_connect(ip, 21);
```
Opens TCP socket to server port 21.

**Step 5: Server greeting**
```c
ftp_read_reply(control, &greeting);
```
Server immediately sends welcome message upon connection:
```
220 Welcome to FTP server
```

**Step 6: Authentication**

**USER command:**
```c
int code = ftp_send_cmd_get_reply(control, &reply, "USER %s", url.user);
```
Server responses:
- `331`: Password required (proceed to PASS)
- `230`: Logged in (no password needed)
- Other: Authentication failed

**PASS command (if code was 331):**
```c
if (code == 331) {
    code = ftp_send_cmd_get_reply(control, &reply, "PASS %s", url.password);
}
```
Expected response: `230 Login successful`

**Step 7: Enter passive mode**
```c
ftp_send_cmd_get_reply(control, &reply, "PASV");
```
Server responds with IP and port for data connection:
```
227 Entering Passive Mode (193,137,29,15,198,138)
```

Parse this response:
```c
char ip_data[64];
int data_port;
ftp_parse_pasv(reply, ip_data, sizeof(ip_data), &data_port);
```

**Step 8: Data connection establishment**
```c
int data_sock = ftp_open_data_connection(ip_data, data_port);
```
Client connects to server's data port. Connection is established but no data flows yet.

**Step 9: Set binary transfer mode**
```c
ftp_send_cmd_get_reply(control, &reply, "TYPE I");
```
`TYPE I` = binary mode (as opposed to ASCII mode which converts line endings).
Essential for non-text files (images, executables, archives, etc.).

**Step 10: Request file retrieval**
```c
code = ftp_send_cmd_get_reply(control, &reply, "RETR %s", url.path);
```
Server responses:
- `150`: File transfer starting
- `450`/`550`: File not found or permission denied

Server reply may contain file size:
```
150 Opening BINARY mode data connection for file.bin (1048576 bytes)
```

Extract size:
```c
long long file_size = ftp_parse_file_size(reply);
```

**Step 11: File creation and transfer**

Open local file for writing:
```c
const char *filename = ftp_basename(url.path);
FILE *f = fopen(filename, "wb");
```
- `"wb"`: write mode, binary
- Saves in current working directory with same filename

Initialize timing:
```c
struct timespec start, current;
clock_gettime(CLOCK_MONOTONIC, &start);
```
`CLOCK_MONOTONIC`: monotonic clock (doesn't jump with system time changes), ideal for measuring elapsed time.

Download loop:
```c
char buf[4096];
int n;
long long total_bytes = 0;

while ((n = recv(data_sock, buf, sizeof(buf), 0)) > 0) {
    fwrite(buf, 1, n, f);
    total_bytes += n;
    
    if (total_bytes % (100 * 1024) < (long long)sizeof(buf)) {
        clock_gettime(CLOCK_MONOTONIC, &current);
        double elapsed = (current.tv_sec - start.tv_sec) + 
                         (current.tv_nsec - start.tv_nsec) / 1e9;
        print_transfer_status(total_bytes, file_size, elapsed);
    }
}
```

**Loop mechanics:**
1. `recv()` reads up to 4096 bytes from data socket
   - Returns number of bytes actually read
   - Returns 0 when server closes connection (transfer complete)
   - Returns -1 on error
2. `fwrite()` writes received bytes to file
3. Update total byte counter
4. Every ~100KB, calculate elapsed time and update progress display

**Time calculation:**
```c
double elapsed = (current.tv_sec - start.tv_sec) + 
                 (current.tv_nsec - start.tv_nsec) / 1e9;
```
- `tv_sec`: seconds component
- `tv_nsec`: nanoseconds component (divide by 1 billion to get fractional seconds)

**Step 12: Transfer completion**

Close file:
```c
fclose(f);
```

Calculate final statistics:
```c
struct timespec end;
clock_gettime(CLOCK_MONOTONIC, &end);
double total_time = (end.tv_sec - start.tv_sec) + 
                    (end.tv_nsec - start.tv_nsec) / 1e9;
```

Display summary:
```c
printf("\n\nTransfer complete!\n");
printf("Total size: %.2f MB\n", total_bytes / 1024.0 / 1024.0);
printf("Transfer time: %.2f seconds\n", total_time);
printf("Average speed: %.2f MB/s\n", (total_bytes / 1024.0 / 1024.0) / total_time);
```

**Step 13: Read transfer completion message**
```c
code = ftp_read_reply(control, &reply);
```
Server sends final status after data connection closes:
```
226 Transfer complete
```

**Step 14: Graceful disconnection**
```c
ftp_quit(control);
```
Sends QUIT command, expects 221 response.

**Step 15: Cleanup**
```c
free_ftp_url(&url);  // Free parsed URL strings
close(data_sock);    // Close data connection socket
close(control);      // Close control connection socket
```

---

## FTP Protocol Implementation

### Two-Connection Architecture

FTP uses two separate TCP connections:

**Control Connection (Port 21):**
- Persistent throughout session
- Carries commands and responses
- ASCII-based text protocol
- Examples: USER, PASS, RETR, QUIT

**Data Connection (Ephemeral Port):**
- Created for each transfer
- Carries actual file data
- Binary data (with TYPE I)
- Closed after transfer completes

### Passive Mode (PASV)

**Why passive mode?**
In active mode, server initiates data connection to client. This fails with firewalls and NAT. Passive mode solves this by having client initiate both connections.

**Process:**
1. Client sends PASV command
2. Server opens random high port (e.g., 50826) and listens
3. Server sends IP:port to client via control connection
4. Client connects to that IP:port for data transfer

### Command Sequence

Complete typical session:
```
C: [connects to server:21]
S: 220 Welcome

C: USER anonymous\r\n
S: 331 Please specify the password\r\n

C: PASS anonymous\r\n
S: 230 Login successful\r\n

C: PASV\r\n
S: 227 Entering Passive Mode (192,168,1,1,195,149)\r\n

C: [connects to 192.168.1.1:50069]

C: TYPE I\r\n
S: 200 Switching to Binary mode\r\n

C: RETR file.bin\r\n
S: 150 Opening BINARY mode data connection for file.bin (1024 bytes)\r\n
S: [sends data on data connection]
S: 226 Transfer complete\r\n

C: QUIT\r\n
S: 221 Goodbye\r\n
```

---

## Data Flow and Execution

### Visual Flow Diagram

```
┌─────────────────┐
│ Command Line    │
│ Parse URL       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ DNS Resolution  │
│ host → IP       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Control Socket  │
│ Connect to :21  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Authentication  │
│ USER + PASS     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ PASV Command    │
│ Get data IP:port│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Data Socket     │
│ Connect to port │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ RETR Command    │
│ Request file    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Download Loop   │
│ recv → write    │
│ Update progress │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Cleanup         │
│ QUIT, close     │
└─────────────────┘
```

### Memory Management

**Heap allocations:**
- URL components (user, password, host, path) via `strdup()`
- Server reply buffers via `malloc()`/`realloc()`
- File buffer (4096 bytes) on stack

**Deallocation points:**
- Each `reply` after printing with `free(reply)`
- URL structure at program end with `free_ftp_url()`
- File closed with `fclose()`
- Sockets closed with `close()`

**Memory leak prevention:**
Every `malloc()`/`strdup()` has corresponding `free()` in same function or at cleanup.

### Error Handling Strategy

**Connection errors:**
- DNS resolution failure: exit immediately
- Socket creation failure: exit immediately
- Connection refused: exit immediately

**Protocol errors:**
- Wrong status code: print error, close connections, exit
- Read failure: print error, close connections, exit
- File creation failure: close sockets, exit

**Philosophy:**
Since this is a simple command-line tool for single file downloads, any error is fatal. No retry logic or recovery mechanisms.

---

## Key Technical Concepts

### Socket Programming

**Socket:** Endpoint for network communication, represented as file descriptor.

**TCP (SOCK_STREAM):**
- Connection-oriented
- Reliable delivery
- Ordered packets
- Error checking

**Socket operations used:**
1. `socket()`: Create socket
2. `connect()`: Establish connection
3. `send()`: Send data
4. `recv()`: Receive data
5. `close()`: Terminate connection

### Byte Order Conversion

**Network byte order:** Big-endian (most significant byte first)
**Host byte order:** Platform-dependent (x86 is little-endian)

**Functions used:**
- `htons()`: Host to network short (16-bit, for port)
- `inet_pton()`: Presentation to network (string IP to binary)
- `inet_ntop()`: Network to presentation (binary to string IP)

**Why necessary?**
Different CPU architectures store multi-byte values differently. Network protocols standardize on big-endian.

### String Handling

**Dynamic allocation:**
- `strdup()`: Allocate and copy string
- `strndup()`: Copy n characters
- `malloc()`/`realloc()`: Manual allocation

**Concatenation:**
- `strcat()`: Append string
- `strncat()`: Append with length limit

**Searching:**
- `strchr()`: Find character
- `strrchr()`: Find last occurrence
- `strncmp()`: Compare n characters

**Safety considerations:**
- Always null-terminate strings
- Use `strncat()` with proper size limits
- Check allocation success

### Time Measurement

**`clock_gettime()` with `CLOCK_MONOTONIC`:**
- High resolution (nanosecond precision)
- Monotonic: never jumps backward
- Unaffected by system time changes (NTP, DST)
- Ideal for measuring durations

**`struct timespec`:**
```c
struct timespec {
    time_t tv_sec;   // Seconds
    long tv_nsec;    // Nanoseconds (0-999,999,999)
};
```

### File I/O

**Binary mode (`"wb"`):**
- No newline translation
- Writes bytes exactly as received
- Essential for non-text files

**Buffered I/O:**
- `fwrite()` uses internal buffer
- Writes to disk in chunks (performance optimization)
- `fclose()` flushes buffer

### Terminal Output

**Carriage return (`\r`):**
Returns cursor to line start without advancing line.
Enables overwriting same line for progress display.

**`fflush(stdout)`:**
Forces immediate display of buffered output.
Necessary because stdout is line-buffered (normally waits for `\n`).

---

## Build and Execution

### Makefile

Compilation:
```bash
make
```

Compiles all `.c` files in `src/` to `.o` files in `build/`, then links into `build/ftp_client`.

Execution:
```bash
make run
```

Downloads test file: `ftp://anonymous:anonymous@ftp.bit.nl/speedtest/100mb.bin`

### Compilation Flags

- `-Wall`: Enable all warnings
- `-Wextra`: Enable extra warnings
- `-O2`: Optimization level 2 (good balance of speed and compilation time)

---

## Limitations and Considerations

**What this implementation does NOT support:**
- Active mode (only passive mode)
- IPv6
- TLS/SSL (FTPS)
- Resume capability
- Multiple file transfers
- Directory operations
- Symbolic links

**Security considerations:**
- Passwords transmitted in cleartext
- No certificate validation
- Anonymous FTP is unsecured by design

**Platform:**
- POSIX-compliant systems (Linux, macOS, BSD)
- Uses POSIX socket API
- `clock_gettime()` requires `_POSIX_C_SOURCE >= 199309L`

---

## Summary

This FTP client demonstrates:
1. Network socket programming in C
2. FTP protocol implementation (RFC 959)
3. DNS resolution
4. String parsing and manipulation
5. Binary file I/O
6. Real-time progress monitoring
7. Modular code organization

The implementation follows the FTP standard for passive mode file downloads, handling the complete lifecycle from URL parsing to file storage with comprehensive error handling and user feedback.
