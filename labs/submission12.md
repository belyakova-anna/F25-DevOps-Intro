# Lab 12 — WebAssembly Containers vs Traditional Containers

## Task 1 — Create the Moscow Time Application

### 1. Confirmation of Working Directory

```bash
belyak_anya@zephyrus:~$ cd /home/belyak_anya/F25-DevOps-Intro/labs/lab12
belyak_anya@zephyrus:~/F25-DevOps-Intro/labs/lab12$ ls
Dockerfile  Dockerfile.wasm  main.go  spin.toml
```

### 2. CLI Mode Test (MODE=once)

Screenshot of CLI mode output (MODE=once)

![MODE=once](https://github.com/user-attachments/assets/17d73c00-9c51-432c-abd2-c6f604343306)

### 3. Server Mode Test

Screenshot of server mode output

![server mode](https://github.com/user-attachments/assets/0ac70f92-a71f-4c5f-97f6-b1be82ce6669)

Screenshot of server mode running in browser

![browser server mode](https://github.com/user-attachments/assets/584919f9-143f-4128-a085-b5907f54c57a)

### 4. Explanation of How main.go Works in Three Different Execution Contexts

### 4.1 How `isWagi()` detects the execution context

The function:

```go
func isWagi() bool {
    return os.Getenv("REQUEST_METHOD") != ""
}
```

detects Spin’s WAGI environment by checking whether the environment variable `REQUEST_METHOD` is set.
WAGI (WebAssembly Gateway Interface) always provides CGI-style variables such as `REQUEST_METHOD`, `PATH_INFO`, `QUERY_STRING`, etc. No other environment in this lab sets these variables. Therefore:

* If `REQUEST_METHOD` is present → running under Spin WAGI executor
* If not → fallback to either CLI or traditional HTTP server mode

This allows the program to reliably identify Spin WAGI execution without additional dependencies.

---

### 4.2 How `runWagiOnce()` handles CGI-style requests

The function:

```go
func runWagiOnce()
```

implements a minimal CGI/WAGI response by writing HTTP headers and the response body directly to **stdout**.

The logic:

* Reads the request path from `PATH_INFO`
* Depending on the path:

  * `/` → prints `Content-Type: text/html` + serves the HTML page
  * `/api/time` → prints `Content-Type: application/json` + JSON output
  * anything else → prints `Status: 404 Not Found`
* Always outputs a blank line after headers (required by CGI/WAGI)
* Exits after handling exactly one request (as expected in WAGI)

This makes the WASM binary compatible with Spin without using any Spin SDK.

---

### 4.3 How the same code works in all three modes (CLI, Docker HTTP, Spin WAGI)

The program chooses its execution mode based on environment variables:

- **CLI mode** (`MODE=once`)

   * If `MODE=once` is set, the program prints a JSON object with Moscow time and exits.
   * This mode works identically for the native Go binary and the WASM binary.

- **WAGI mode** (Spin)

   * If `REQUEST_METHOD` is detected, the program assumes Spin WAGI.
   * Calls `runWagiOnce()` and generates CGI-style output.
   * No networking or HTTP server is used; Spin handles HTTP externally.

- **Traditional server mode** (Docker)

   * If neither of the above conditions is met, the program starts a normal Go HTTP server on port 8080.
   * Uses `net/http` handlers for `/` and `/api/time`.

Because the mode is selected purely via environment variables, the **same exact `main.go` file** can run:

* as a native Linux HTTP server,
* as a lightweight WASM CLI application under containerd,
* and as a full HTTP endpoint under Spin Cloud.

## Task 2 — Build Traditional Docker Container

### 1. Container build

![container build](https://github.com/user-attachments/assets/bad36b7b-70b4-446a-b76d-753aa465bc6e)

### 2. Test CLI Mode

![cli mode](https://github.com/user-attachments/assets/53109167-f887-4344-a9c2-fb4d2d4720dc)

### 3. Test Server Mode

![server mode](https://github.com/user-attachments/assets/d0d90f86-8305-48fe-b1f1-564f79e46f26)

### 4. Binary size

![binary size](https://github.com/user-attachments/assets/7bfacb00-d9e3-4627-b9c1-483f6237ae1d)

**Binary size (traditional container):** 4.5 MB

### 5. Image size

![image size](https://github.com/user-attachments/assets/6af2b9c6-309b-47d0-91df-db83cb60b8ed)

**Image size (docker images):** 4.7 MB

**Image size (docker image inspect):** 4.48 MB

### 6. Startup time benchmark

![startup time](https://github.com/user-attachments/assets/0cc65a23-e1b2-4aa0-9386-e7fc897c0e60)

**Average startup time (5 runs):** 0.326 seconds

### 7. Memory usage

![memory usage](https://github.com/user-attachments/assets/45048e78-f2cc-4b72-ba65-bc16753083d1)

**Memory usage (traditional container):** 1.461 MiB

## Task 3 — Build WASM Container (ctr-based)

### 1. TinyGo version

![tiny go](https://github.com/user-attachments/assets/10a1909e-fbc9-475f-bc80-aa55d197bdf6)

**TinyGo version used:** tinygo version 0.39.0 linux/amd64

### 2. Build WASM Binary Using TinyGo

![wasm binary](https://github.com/user-attachments/assets/e8a66c07-8710-411f-a91c-575cc790f927)

**WASM binary size (main.wasm):** 2.4 MB

### 3. Build WASI OCI Image and Import into containerd

Using Docker Buildx, I built an OCI image for the WASI platform:

![oci image](https://github.com/user-attachments/assets/14b3c52a-5416-4583-b870-7402e3893fca)

Then I imported this OCI archive into containerd:

![oci archive](https://github.com/user-attachments/assets/83f11545-25cb-4be3-b2bc-eba257cc1355)

I confirmed that the image was successfully registered in containerd using `ctr images ls`:

![image](https://github.com/user-attachments/assets/597abe4b-e888-4834-aaa7-a33817f5d968)

**WASI image size (ctr images ls):** 819.9 KiB

### 4. Run the WASM Container (CLI Mode)

![cli mode](https://github.com/user-attachments/assets/17000574-5f97-4291-adaf-084b4158565a)

### 5. Startup Time Benchmark

![time](https://github.com/user-attachments/assets/4693f32b-24fb-4894-8635-105f585f6016)

**Average startup time (5 runs):** 0.342 seconds

### 6. Why Server Mode Does Not Work Under `ctr`

Server mode cannot run under `ctr` because this WASM module targets **WASI Preview1**, which does **not** provide TCP socket support.

The Go `net/http` server inside `main.go` attempts to bind to port `8080`, but WASI runtimes (including Wasmtime) do not expose any networking APIs. Therefore, only CLI mode works when running the WASM module through containerd.

### 7. Server Mode Works Under Spin

Although server mode does not work in containerd, the **same `main.wasm`** runs correctly under Spin in WAGI mode.

Spin provides an HTTP interface and forwards incoming requests to the WASM module using CGI-style environment variables. The existing `isWagi()` and `runWagiOnce()` functions make the same binary Spin-compatible without modifications.

### 8. Memory Usage (N/A)

Memory metrics for WASM containers are **not available** using `ctr`.  
Unlike traditional Linux containers, WASM workloads executed via the Wasmtime shim are not exposed through cgroup metrics, so containerd cannot report memory usage.

**Memory usage:** N/A — containerd does not expose memory metrics for WASM shims.

### 9. Same Source Code as Traditional Build

The WASM version of the application was compiled from the **exact same `main.go` file** used in Task 2 for the traditional Docker container.

### 10. Confirmation of Using `ctr` for WASM Execution

I executed the WASM image using the **containerd CLI (`ctr`)**, as required, including:

- `ctr images import` to load the OCI image
- `ctr images ls` to verify it
- `ctr run --rm --runtime io.containerd.wasmtime.v1 --platform wasi/wasm ...` to execute the WASM module in MODE=once

## Task 4 — Performance Comparison & Analysis

### 1. Comprehensive Comparison Table

| Metric                 | Traditional Container | WASM Container                       | Improvement                                    | Notes                                         |
| ---------------------- | --------------------- | ------------------------------------ | ---------------------------------------------- | --------------------------------------------- |
| **Binary Size**        | **4.5 MB**            | **2.4 MB**                           | **46.7% smaller**                              | From `ls -lh`                                 |
| **Image Size**         | **4.7 MB**            | **0.82 MB**                          | **82.5% smaller**                              | From `docker image inspect` / `ctr images ls` |
| **Startup Time (CLI)** | **326 ms**            | **342 ms**                           | **0.95× faster (traditional slightly faster)** | Average of 5 runs                             |
| **Memory Usage**       | **1.461 MB**          | **N/A**                              | **N/A**                                        | From `docker stats`; WASM memory unavailable  |
| **Base Image**         | scratch               | scratch                              | Same                                           | Both minimal                                  |
| **Source Code**        | main.go               | main.go                              | Identical                                      | ✅ Same file!                                  |
| **Server Mode**        | ✅ Works (net/http)    | ❌ Not via ctr <br> ✅ Via Spin (WAGI) | N/A                                            | WASI Preview1 has no sockets                  |

---

### Calculated improvement percentages

- **Binary Size Reduction**

```
(4.5 − 2.4) / 4.5 × 100 = 46.7%
```

- **Image Size Reduction**

```
(4.7 − 0.82) / 4.7 × 100 = 82.5%
```

- **Startup Time Speed Ratio**

```
Traditional / WASM = 326 / 342 = 0.95×
```

Traditional container is ≈5% faster.

- **Memory Reduction**

```
N/A — WASM memory metrics not supported by ctr
```

---

### 2. Analysis Questions

### 2.1 Binary Size Comparison

**Why is the WASM binary so much smaller than the traditional Go binary?**

Because TinyGo produces extremely compact binaries by eliminating most of the Go runtime and standard library. It only keeps the code that is directly used and provides a minimal WASI-compatible runtime.

**What did TinyGo optimize away?**

- Full Go garbage collector
- Scheduler and goroutine runtime
- Most of the standard library
- OS abstractions and syscalls
- Debug symbols and metadata

This makes the WASM binary about ~2× smaller.

### 2.2 Startup Performance

**Why does WASM start faster?**

WASM runtimes initialize very quickly: they load a small module, allocate linear memory, and start execution inside a sandbox. No OS process needs to be created.

**What initialization overhead exists in traditional containers?**

Traditional containers must:

- start a full Linux process
- initialize the standard Go runtime (GC, scheduler, stacks)
- load ELF binary headers
- perform libc and kernel interactions

This adds extra startup latency compared to WASM.

### 2.3 Use Case Decision Matrix

**When would you choose WASM over traditional containers?**

- For serverless / FaaS functions with cold start requirements
- For edge and embedded environments with small storage
- For high-security sandboxed execution
- For cross-platform portability (browser, ARM, Linux, etc.)
- When deploying many tiny microservices

**When would you stick with traditional containers?**

- For applications needing networking (`net/http`, TCP sockets)
- For long-running services
- For workloads using the full Go runtime
- For apps needing OS features (filesystem, processes, cgroups)
- When accurate resource metrics are required
