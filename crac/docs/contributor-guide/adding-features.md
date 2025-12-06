# Adding Features

This guide explains how to extend CRAC with new functionality, including adding DMTCP event handlers, extending CUDA checkpoint capabilities, and following the project's coding conventions.

## Understanding the Extension Points

CRAC has several well-defined extension points where new functionality can be added:

### 1. DMTCP Event Handlers
Add new event handling in the `cuda_event_hook` function.

### 2. CUDA API Integration
Extend CUDA checkpoint/restart functionality with additional CUDA APIs.

### 3. Resource Tracking
Add tracking for additional system resources that need preservation.

### 4. Trampoline Wrappers
Add wrappers for system calls that need special handling.

## Adding New DMTCP Event Handlers

### Step 1: Identify the Event
Review the DMTCP event types in `dmtcp.h` to find the appropriate event for your functionality.

Common events:
- `DMTCP_EVENT_INIT`: Plugin initialization
- `DMTCP_EVENT_EXIT`: Process exit
- `DMTCP_EVENT_PREEXEC`: Before exec() call
- `DMTCP_EVENT_POSTEXEC`: After exec() call
- `DMTCP_EVENT_ATFORK_PREPARE`: Before fork()
- `DMTCP_EVENT_ATFORK_PARENT`: After fork() in parent
- `DMTCP_EVENT_ATFORK_CHILD`: After fork() in child

### Step 2: Add Event Handler
Edit the `cuda_event_hook` function in `crac.cpp`:

```cpp
static void cuda_event_hook(DmtcpEvent_t event, DmtcpEventData_t *data) {
  switch (event) {
    // Existing handlers...
    
    case DMTCP_EVENT_EXIT:
      // Your new exit handling code
      handle_process_exit();
      break;
      
    case DMTCP_EVENT_PREEXEC:
      // Your new pre-exec handling code
      prepare_for_exec();
      break;
      
    default:
      break;
  }
}
```

### Step 3: Implement Handler Function
Add your handler function implementation:

```cpp
static void handle_process_exit() {
  // Cleanup resources before process exit
  // Example: Close temporary files, free memory
  if (cuda_initialized) {
    // Cleanup CUDA resources if needed
    cuda_initialized = 0;
  }
}

static void prepare_for_exec() {
  // Prepare process for exec() call
  // Example: Save state that needs to survive exec
}
```

### Step 4: Update Data Structures
If your new functionality requires additional state, add global variables:

```cpp
// Add with other global variables in crac.cpp
static int my_feature_enabled = 0;
static char *saved_state = NULL;
```

## Extending CUDA Checkpoint Capabilities

### Adding New CUDA API Calls

#### Step 1: Include Required Headers
```cpp
#include <cuda.h>           // Already included
#include <cuda_runtime_api.h> // Already included
// Add any additional CUDA headers needed
```

#### Step 2: Extend Checkpoint Function
Modify `checkpoint_gpu()` to include new CUDA operations:

```cpp
static void checkpoint_gpu() {
  // Existing code...
  
  // Add your new CUDA checkpoint logic
  if (my_feature_enabled) {
    CUresult ret = your_cuda_checkpoint_function();
    JASSERT(ret == CUDA_SUCCESS)(ret);
  }
  
  // Continue with existing code...
}
```

#### Step 3: Extend Restore Function
Modify `restore_gpu()` to include new CUDA operations:

```cpp
static void restore_gpu() {
  // Existing code...
  
  // Add your new CUDA restore logic
  if (my_feature_enabled) {
    CUresult ret = your_cuda_restore_function();
    JASSERT(ret == CUDA_SUCCESS)(ret);
  }
  
  // Continue with existing code...
}
```

### Example: Adding CUDA Stream Tracking

```cpp
// Global variables for stream tracking
static std::vector<CUstream> tracked_streams;
static int stream_tracking_enabled = 0;

// In checkpoint_gpu():
if (stream_tracking_enabled) {
  for (CUstream stream : tracked_streams) {
    CUresult ret = cuStreamSynchronize(stream);
    JASSERT(ret == CUDA_SUCCESS)(ret);
  }
}

// In restore_gpu():
if (stream_tracking_enabled) {
  // Recreate streams if needed
  for (size_t i = 0; i < tracked_streams.size(); i++) {
    CUresult ret = cuStreamCreate(&tracked_streams[i], CU_STREAM_DEFAULT);
    JASSERT(ret == CUDA_SUCCESS)(ret);
  }
}
```

## Adding Resource Tracking

### Example: Tracking Shared Memory

#### Step 1: Define Data Structures
```cpp
typedef struct {
    void *addr;
    size_t size;
    int shmid;
} shmem_info_t;

#define MAX_SHMEM_SEGMENTS 64
static shmem_info_t shmem_list[MAX_SHMEM_SEGMENTS];
static int num_shmem_segments = 0;
```

#### Step 2: Add Inspection Function
```cpp
static int inspect_shmem_segments() {
  // Scan /proc/self/maps for shared memory segments
  dmtcp::ProcSelfMaps proc_maps;
  Area area;
  int count = 0;
  
  while (proc_maps.getNextArea(&area) && count < MAX_SHMEM_SEGMENTS) {
    if (strstr(area.name, "SYSV") || strstr(area.name, "/dev/shm")) {
      shmem_list[count].addr = area.addr;
      shmem_list[count].size = area.size;
      // Extract shmid if needed
      count++;
    }
  }
  
  num_shmem_segments = count;
  return count;
}
```

#### Step 3: Add to Event Handlers
```cpp
// In DMTCP_EVENT_PRECHECKPOINT:
case DMTCP_EVENT_PRECHECKPOINT:
  // Existing code...
  num_shmem_segments = inspect_shmem_segments();
  break;

// In DMTCP_EVENT_RESTART:
case DMTCP_EVENT_RESTART:
  // Existing code...
  restore_shmem_segments();
  break;
```

## Adding System Call Wrappers

### Step 1: Define Trampoline Structure
```cpp
static trampoline_info_t my_syscall_trampoline_info;
```

### Step 2: Create Wrapper Function
```cpp
static int my_syscall_wrapper(int arg1, int arg2) {
  return real_my_syscall(arg1, arg2);
}
```

### Step 3: Create Trampoline Function
```cpp
static int my_syscall_trampoline(int arg1, int arg2) {
  UNINSTALL_TRAMPOLINE(my_syscall_trampoline_info);
  int retval = my_syscall_wrapper(arg1, arg2);
  INSTALL_TRAMPOLINE(my_syscall_trampoline_info);
  return retval;
}
```

### Step 4: Initialize in DMTCP_EVENT_INIT
```cpp
case DMTCP_EVENT_INIT:
  // Existing initialization...
  dmtcp_setup_trampoline("my_syscall", (void *)&my_syscall_trampoline,
                         &my_syscall_trampoline_info);
  break;
```

## Code Style and Conventions

### Naming Conventions
- **Functions**: `snake_case` (e.g., `inspect_pipes`, `restore_gpu`)
- **Variables**: `snake_case` (e.g., `cuda_state`, `num_fds_found`)
- **Constants**: `UPPER_CASE` (e.g., `MAX_PIPE_FDS`, `PATH_MAX`)
- **Types**: `snake_case` with `_t` suffix (e.g., `pipe_info_t`, `shmem_info_t`)

### Error Handling
Always use `JASSERT` for error checking:

```cpp
CUresult ret = cuSomeFunction();
JASSERT(ret == CUDA_SUCCESS)(ret);

int result = some_syscall();
JASSERT(result != -1)(errno);

// For optional operations
if (optional_feature_enabled) {
  ret = optional_function();
  if (ret != CUDA_SUCCESS) {
    // Handle gracefully, maybe disable feature
    optional_feature_enabled = 0;
  }
}
```

### Memory Management
- Use RAII patterns with STL containers
- Free allocated memory in cleanup functions
- Use `std::vector` instead of raw arrays when possible

```cpp
// Good: Use STL containers
std::vector<my_struct_t> my_list;

// Avoid: Raw arrays with manual management
my_struct_t *my_list = malloc(sizeof(my_struct_t) * count);
// ... remember to free later
```

### Documentation
Add comments for complex functions:

```cpp
/**
 * Brief description of what the function does
 * 
 * @param param1 Description of first parameter
 * @param param2 Description of second parameter
 * @return Description of return value
 * 
 * Important notes about side effects or usage
 */
static int my_function(int param1, char *param2) {
  // Implementation...
}
```

## Testing New Features

### 1. Unit Testing
Create simple test programs to verify your functionality:

```cpp
// test/my_feature_test.cu
#include <stdio.h>

int main() {
  // Test your new feature
  printf("Testing new feature...\n");
  
  // Call functions that trigger your new code paths
  
  printf("Test completed\n");
  return 0;
}
```

### 2. Integration Testing
Test with existing CRAC functionality:

```bash
# Build with your changes
make clean && make DMTCP_ROOT=$DMTCP_ROOT

# Test with existing programs
$DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter

# Verify checkpoint/restart works
$DMTCP_ROOT/bin/dmtcp_command --checkpoint
# ... restart and verify
```

### 3. Edge Case Testing
Test error conditions and edge cases:

```cpp
// Test with maximum limits
// Test with zero values
// Test with invalid inputs
// Test resource exhaustion scenarios
```

## Performance Considerations

### 1. Minimize Checkpoint Time
- Avoid expensive operations in checkpoint path
- Cache expensive computations
- Use lazy evaluation when possible

### 2. Memory Usage
- Be mindful of memory overhead
- Use efficient data structures
- Clean up resources promptly

### 3. Scalability
- Consider impact on large-scale applications
- Test with multiple processes
- Verify behavior with many resources

## Submitting Changes

### 1. Create Feature Branch
```bash
git checkout -b feature/my-new-feature
```

### 2. Make Changes
- Implement your feature
- Add tests
- Update documentation

### 3. Test Thoroughly
```bash
# Run existing tests
make check

# Add your test to the test suite
# Verify all tests pass
```

### 4. Commit Changes
```bash
git add .
git commit -m "Add feature: brief description of changes

- Detailed description of what was added
- How it works
- Testing performed
- Any limitations"
```

### 5. Submit Pull Request
- Create pull request against main branch
- Include clear description of changes
- Reference any related issues

## Common Pitfalls to Avoid

### 1. Race Conditions
- Be careful with shared state
- Use proper synchronization
- Consider DMTCP's threading model

### 2. Memory Leaks
- Always free allocated memory
- Use valgrind to check for leaks
- Pay attention to error paths

### 3. Incorrect Event Ordering
- Understand DMTCP event sequence
- Don't assume event order
- Handle all relevant events

### 4. CUDA API Errors
- Always check return values
- Handle CUDA errors appropriately
- Consider CUDA context state

## See Also

- [Development Setup](development-setup.md) - Setting up development environment
- [Code Walkthrough](code-walkthrough.md) - Understanding existing code
- [Testing Guide](testing.md) - Testing your changes
- [DMTCP Plugin API](../reference/dmtcp-plugin-api.md) - DMTCP event reference
- [CUDA Checkpoint API](../reference/cuda-checkpoint-api.md) - CUDA API reference