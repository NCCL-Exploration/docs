# DMTCP Debugging and Troubleshooting

This guide covers debugging techniques, common issues, and troubleshooting strategies for DMTCP development and usage.

## Debugging Tools and Techniques

### 1. Logging System

DMTCP provides a comprehensive logging system through JALIB utilities.

#### Enabling Logging

```bash
# Configure with logging support
./configure --enable-logging
make clean && make

# Run application with logging
dmtcp_launch myapplication

# Logs appear in /tmp/dmtcp-USER@HOST/jassertlog.*
```

#### Log Levels

```cpp
// In DMTCP code
JTRACE("Trace message") (variable1) (variable2);     // Detailed tracing
JNOTE("Informational message") (context);             // Important events
JWARNING("Warning message") (warning_context);        // Non-fatal issues
JERROR("Error message") (error_context);             // Fatal errors (calls _exit)
```

#### Log File Analysis

```bash
# View all logs for a computation
ls /tmp/dmtcp-$USER@$(hostname)/jassertlog.*

# Follow log in real-time
tail -f /tmp/dmtcp-$USER@$(hostname)/jassertlog.PID

# Search for specific events
grep "PRECHECKPOINT" /tmp/dmtcp-$USER@$(hostname)/jassertlog.*
grep "Connection" /tmp/dmtcp-$USER@$(hostname)/jassertlog.*
```

### 2. GDB Debugging

#### Debugging Application Launch

```bash
# Debug dmtcp_launch and application startup
gdb --args dmtcp_launch test/dmtcp1
(gdb) break main
(gdb) break 'dmtcp::DmtcpWorker::DmtcpWorker()'
(gdb) run

# Stop when application enters main()
(gdb) break execvp
(gdb) continue
(gdb) break main
(gdb) continue
```

#### Debugging Checkpoint Process

```bash
# Attach to running application during checkpoint
dmtcp_launch test/dmtcp1 &
APP_PID=$!

# Wait for checkpoint to start
dmtcp_command --checkpoint

# Attach to application process
gdb -p $APP_PID
(gdb) source util/gdb-dmtcp-utils.py
(gdb) break stopthisthread
(gdb) continue
```

#### Debugging Restart Process

```bash
# Use restart pause mechanism
DMTCP_RESTART_PAUSE=3 dmtcp_restart ckpt_*.dmtcp &
RESTART_PID=$!

# Find the restarted process
sleep 2
APP_PID=$(pgrep -n test/dmtcp1)

# Attach GDB
gdb -p $APP_PID
(gdb) source util/gdb-dmtcp-utils.py
(gdb) load-symbols-library <ADDRESS_FROM_LOGS>
(gdb) break main
(gdb) continue
```

#### GDB Utilities

DMTCP provides specialized GDB utilities in `util/gdb-dmtcp-utils.py`:

```python
# Key GDB commands provided
(gdb) load-symbols-library <addr>    # Load libdmtcp.so symbols
(gdb) print-pid-table                 # Show PID/TID mappings
(gdb) print-connections               # Show connection table
(gdb) print-thread-list               # Show thread information
```

### 3. Environment Variables for Debugging

#### Failure Handling

```bash
# Generate core dumps on assertion failures
export DMTCP_ABORT_ON_FAILURE=1

# Sleep infinitely on failure (for GDB attach)
export DMTCP_SLEEP_ON_FAILURE=1

# Custom exit code on failure
export DMTCP_FAIL_RC=42
```

#### Restart Debugging

```bash
# Pause restart at different stages (1-7)
export DMTCP_RESTART_PAUSE=1  # Pause immediately
export DMTCP_RESTART_PAUSE=3  # Pause after memory restore
export DMTCP_RESTART_PAUSE=7  # Pause just before resume

# Alternative command-line option
dmtcp_restart --debug-restart-pause ckpt_*.dmtcp
```

#### Coordinator Debugging

```bash
# Enable coordinator debugging
export DMTCP_COORD_DEBUG=1

# Use alternative port for debugging
dmtcp_coordinator --port 7779 --debug
```

## Common Issues and Solutions

### 1. Checkpoint Failures

#### Thread Suspension Issues

**Symptoms:**
- Checkpoint hangs in SUSPENDED state
- Threads not responding to SIGUSR2
- Coordinator timeout

**Debugging Steps:**

```bash
# Check thread states
cat /proc/$(pgrep myapplication)/status | grep State

# Look for blocked threads
ps -eL | grep myapplication

# Check for signal handlers
cat /proc/$(pgrep myapplication)/status | grep SigPnd
```

**Common Causes:**
- Thread stuck in uninterruptible sleep (D state)
- Signal handler not properly installed
- Thread blocked in kernel syscall

**Solutions:**
```cpp
// Ensure signal handlers are installed early
static void installSignalHandlers() {
  struct sigaction sa;
  sa.sa_handler = stopthisthread;
  sigemptyset(&sa.sa_mask);
  sa.sa_flags = SA_RESTART;
  sigaction(SIGUSR2, &sa, NULL);
}
```

#### Socket Draining Timeouts

**Symptoms:**
- Checkpoint hangs in DRAINED state
- Network connection timeouts
- "Drain timeout" error messages

**Debugging Steps:**

```bash
# Check network connections
netstat -an | grep ESTABLISHED

# Monitor socket activity
strace -e trace=network -p $(pgrep myapplication)

# Check for blocked reads
lsof -p $(pgrep myapplication) | grep TCP
```

**Solutions:**
- Increase drain timeout: `export DMTCP_DRAIN_TIMEOUT=30`
- Check network connectivity
- Verify peer process is responsive

#### Disk Space Issues

**Symptoms:**
- Checkpoint fails with "No space left on device"
- Partial checkpoint files
- "Disk full" error messages

**Debugging Steps:**

```bash
# Check available space
df -h /tmp

# Monitor checkpoint file sizes
ls -lh /tmp/ckpt_*.dmtcp

# Check disk usage during checkpoint
watch -n 1 'df -h /tmp; ls -lh /tmp/ckpt_*.dmtcp'
```

**Solutions:**
- Free disk space in `/tmp`
- Use alternative checkpoint directory: `export DMTCP_TMPDIR=/path/to/space`
- Enable compression: `export DMTCP_COMPRESS=1`

### 2. Restart Failures

#### Memory Restoration Errors

**Symptoms:**
- Restart fails during memory restoration
- "Cannot mmap" errors
- Segmentation faults on restart

**Debugging Steps:**

```bash
# Check memory availability
free -h

# Verify checkpoint file integrity
file ckpt_*.dmtcp

# Check architecture compatibility
uname -m
file ckpt_*.dmtcp | grep -i architecture
```

**Common Causes:**
- Insufficient memory on restart host
- Architecture mismatch (32-bit vs 64-bit)
- Corrupted checkpoint files

**Solutions:**
- Ensure sufficient memory available
- Match architecture between checkpoint and restart
- Verify checkpoint file integrity

#### Socket Reconnection Failures

**Symptoms:**
- Restart fails to reconnect sockets
- "Connection refused" errors
- Processes cannot communicate after restart

**Debugging Steps:**

```bash
# Check coordinator status
dmtcp_command --status

# Verify network connectivity
telnet peer_host peer_port

# Check firewall settings
iptables -L
```

**Solutions:**
- Ensure coordinator is running
- Check network connectivity between hosts
- Verify firewall allows coordinator port (default 7779)

#### PID/TID Conflicts

**Symptoms:**
- Restart fails with "PID conflict"
- Processes terminate unexpectedly
- "TID conflict detected" messages

**Debugging Steps:**

```bash
# Check running processes
ps aux | grep myapplication

# Look for conflicting PIDs
cat /tmp/dmtcp-$USER@$(hostname)/jassertlog.* | grep -i conflict
```

**Solutions:**
- Kill conflicting processes
- Use different user account
- Restart coordinator to clear state

### 3. Performance Issues

#### Slow Checkpoints

**Symptoms:**
- Checkpoints take excessive time
- Application appears frozen during checkpoint
- Poor checkpoint frequency

**Debugging Steps:**

```bash
# Time checkpoint operation
time dmtcp_command --checkpoint

# Monitor I/O during checkpoint
iotop -p $(pgrep dmtcp_coordinator)

# Check memory usage
cat /proc/$(pgrep myapplication)/status | grep Vm
```

**Optimization Strategies:**
- Enable compression: `export DMTCP_COMPRESS=1`
- Use faster storage for checkpoint directory
- Reduce checkpoint frequency if possible

#### High Memory Usage

**Symptoms:**
- DMTCP uses excessive memory
- Out-of-memory errors
- System swapping during checkpoint

**Debugging Steps:**

```bash
# Monitor memory usage
watch -n 1 'ps aux | grep -E "(myapplication|dmtcp)"'

# Check memory maps
cat /proc/$(pgrep myapplication)/smaps

# Profile memory allocation
valgrind --tool=massif dmtcp_launch myapplication
```

**Solutions:**
- Reduce application memory usage
- Increase system memory
- Use 64-bit system for larger address space

## Advanced Debugging Techniques

### 1. Memory Debugging

#### Valgrind Integration

```bash
# Debug memory leaks in DMTCP
valgrind --leak-check=full --show-leak-kinds=all \
         dmtcp_launch test/dmtcp1

# Debug memory errors during checkpoint
valgrind --tool=memcheck --track-origins=yes \
         dmtcp_launch --interval 10 test/dmtcp1
```

#### AddressSanitizer

```bash
# Compile with AddressSanitizer
export CFLAGS="-fsanitize=address -g"
export CXXFLAGS="-fsanitize=address -g"
export LDFLAGS="-fsanitize=address"

./configure
make clean && make

# Run with ASan
ASAN_OPTIONS=detect_leaks=1 dmtcp_launch test/dmtcp1
```

### 2. Thread Debugging

#### Thread Sanitizer

```bash
# Compile with ThreadSanitizer
export CFLAGS="-fsanitize=thread -g"
export CXXFLAGS="-fsanitize=thread -g"
export LDFLAGS="-fsanitize=thread"

./configure
make clean && make

# Run with TSan
dmtcp_launch test/dmtcp1
```

#### Thread State Analysis

```bash
# Examine thread states
cat /proc/$(pgrep myapplication)/status | grep -A 20 Threads

# Monitor thread creation
strace -f -e trace=clone,fork -p $(pgrep dmtcp_coordinator)

# GDB thread debugging
gdb -p $(pgrep myapplication)
(gdb) info threads
(gdb) thread apply all bt
```

### 3. Network Debugging

#### Packet Capture

```bash
# Capture coordinator communication
tcpdump -i lo port 7779 -w coordinator.pcap

# Capture application network traffic
tcpdump -i any -w app_traffic.pcap host $(hostname)

# Analyze with Wireshark
wireshark coordinator.pcap
```

#### Socket Debugging

```bash
# List all sockets
ss -tulpn | grep myapplication

# Monitor socket activity
strace -e trace=send,recv,sendto,recvfrom -p $(pgrep myapplication)

# Debug socket options
cat /proc/$(pgrep myapplication)/fdinfo/* | grep socket
```

## Plugin-Specific Debugging

### Plugin Loading Issues

```bash
# Check plugin loading
LD_DEBUG=libs dmtcp_launch --with-plugin ./libmyplugin.so myapplication 2>&1 | grep myplugin

# Verify plugin symbols
nm -D libmyplugin.so | grep dmtcp

# Check plugin dependencies
ldd libmyplugin.so
```

### Plugin Event Debugging

```cpp
// Add debugging to plugin events
static void myplugin_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data) {
  JTRACE("Plugin event") (event) (getpid());
  
  switch (event) {
    case DMTCP_EVENT_PRECHECKPOINT:
      JNOTE("Pre-checkpoint in plugin");
      break;
    // ... other events
  }
}
```

### Plugin Wrapper Debugging

```bash
# Trace wrapper calls
LD_DEBUG=bindings dmtcp_launch --with-plugin ./libmyplugin.so myapplication 2>&1 | grep open

# GDB wrapper debugging
gdb --args dmtcp_launch --with-plugin ./libmyplugin.so myapplication
(gdb) break myplugin_open
(gdb) run
```

## Testing and Validation

### Unit Testing

```bash
# Run DMTCP test suite
make check

# Run specific test
make check-dmtcp1

# Run with debugging
make check-dmtcp1 DMTCP_DEBUG=1
```

### Integration Testing

```bash
# Test multi-process applications
dmtcp_launch test/client-server &
dmtcp_command --checkpoint
dmtcp_restart ckpt_*_*.dmtcp

# Test plugin integration
dmtcp_launch --with-plugin ./libmyplugin.so test/dmtcp1
dmtcp_command --checkpoint
dmtcp_restart ckpt_dmtcp1_*.dmtcp
```

### Stress Testing

```bash
# High-frequency checkpointing
dmtcp_launch --interval 1 test/dmtcp1

# Large memory applications
dmtcp_launch test/alloc-large-memory

# Many processes
for i in {1..100}; do
  dmtcp_launch test/dmtcp1 &
done
dmtcp_command --checkpoint
```

## Getting Help

### Community Resources

- **GitHub Issues**: https://github.com/dmtcp/dmtcp/issues
- **Mailing List**: dmtcp-forum@lists.sourceforge.net
- **Documentation**: `doc/` directory in source

### Bug Reports

When filing bug reports, include:

1. **System Information**:
   ```bash
   uname -a
   lsb_release -a
   gcc --version
   ```

2. **DMTCP Version**:
   ```bash
   dmtcp_launch --version
   ```

3. **Configuration**:
   ```bash
   ./config.status
   ```

4. **Logs**:
   ```bash
   tar czf dmtcp-logs.tar.gz /tmp/dmtcp-$USER@$(hostname)/
   ```

5. **Reproduction Steps**:
   - Exact command line used
   - Application source (if available)
   - Checkpoint files (if relevant)

### Debug Information Collection

```bash
# Collect comprehensive debug information
#!/bin/bash
echo "=== System Information ==="
uname -a
lsb_release -a

echo "=== DMTCP Version ==="
dmtcp_launch --version

echo "=== Configuration ==="
cat config.status

echo "=== Environment ==="
env | grep DMTCP

echo "=== Process Information ==="
ps aux | grep -E "(dmtcp|myapplication)"

echo "=== Network Information ==="
netstat -an | grep -E "(7779|LISTEN)"

echo "=== Logs ==="
tar czf debug-logs.tar.gz /tmp/dmtcp-$USER@$(hostname)/
```

---

This debugging guide should help resolve most common DMTCP issues. For complex problems, consider reaching out to the community through the mailing list or GitHub issues.