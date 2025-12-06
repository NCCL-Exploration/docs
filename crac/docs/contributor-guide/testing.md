# Testing Guide

This guide covers the CRAC test suite, how to run tests, and guidelines for creating new test cases.

## Test Suite Overview

The CRAC test suite is located in the `test/` directory and includes:

### Test Programs
- **`counter.cu`**: Simple CUDA counter test
- **`counter_mpi.cu`**: MPI + CUDA counter test  
- **`mpi_cuda.cu`**: Complex MPI + CUDA vector addition test

### Test Infrastructure
- **`test/Makefile`**: Build configuration for test programs
- **Main Makefile**: Integration with test suite via `make check` target

## Running Tests

### Prerequisites
```bash
# Ensure CRAC is built
make DMTCP_ROOT=$DMTCP_ROOT

# Ensure tests are built
make tests

# Verify DMTCP is available
$DMTCP_ROOT/bin/dmtcp_launch --version
```

### Quick Test Run
```bash
# Run the built-in check target
make check DMTCP_ROOT=$DMTCP_ROOT
```

This will:
1. Kill any old coordinator on port 7779
2. Launch `./test/counter` with CRAC plugin
3. Take automatic checkpoints every 5 seconds
4. Verify basic functionality

### Manual Test Execution

#### Basic CUDA Test
```bash
# Start coordinator
$DMTCP_ROOT/bin/dmtcp_coordinator --port 7779 &

# Launch test program
$DMTCP_ROOT/bin/dmtcp_launch --coord-port 7779 --with-plugin ./libdmtcp_crac.so ./test/counter &

# Take checkpoint
$DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779

# Kill and restart
kill %1  # Kill the test program
$DMTCP_ROOT/bin/dmtcp_restart --coord-port 7779 ckpt_*.dmtcp
```

#### MPI + CUDA Test
```bash
# Requires MPI to be installed
mpirun -np 2 $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/counter_mpi
```

#### Complex Vector Addition Test
```bash
# More comprehensive test
mpirun -np 4 $DMTCP_ROOT/bin/dmtcp_launch --with-plugin ./libdmtcp_crac.so ./test/mpi_cuda
```

## Test Descriptions

### counter.cu
**Purpose**: Basic CUDA functionality test

**Features**:
- Device variable increment in kernel
- Host-device memory copy
- Continuous loop with sleep
- Minimal resource usage

**What it Tests**:
- Basic CUDA kernel execution
- Device memory persistence
- Simple checkpoint/restart scenario

**Expected Behavior**:
```
0...
1...
2...
3...
[checkpoint]
4...
5...
6...
```

### counter_mpi.cu
**Purpose**: MPI + CUDA integration test

**Features**:
- MPI initialization and finalization
- Same CUDA counter logic as basic test
- Multi-process coordination

**What it Tests**:
- MPI + CUDA compatibility
- Checkpoint coordination across processes
- MPI state preservation

**Expected Behavior**:
Each process prints its counter value independently.

### mpi_cuda.cu
**Purpose**: Complex distributed CUDA computation test

**Features**:
- Distributed vector addition
- MPI scatter/gather operations
- CUDA kernel execution
- Result verification

**What it Tests**:
- Complex CUDA memory management
- MPI data distribution
- Large-scale computation checkpoint
- Result correctness after restart

**Expected Behavior**:
```
Iteration 1
Total Vector Size N = 1048576, running with P = 4 processes.
Each process handles a chunk of size 262144.
Verification check...
Successfully verified the distributed vector addition!
Iteration 2
...
```

## Automated Testing

### Test Script Framework
Create comprehensive test automation:

**run_tests.sh**:
```bash
#!/bin/bash

set -e

DMTCP_ROOT=${DMTCP_ROOT:-/usr/local}
PLUGIN_PATH="./libdmtcp_crac.so"
TEST_LOG="test_results.log"
FAILED_TESTS=0
TOTAL_TESTS=0

# Function to run single test
run_test() {
    local test_name=$1
    local test_command=$2
    local expected_duration=$3
    
    echo "Running test: $test_name" | tee -a $TEST_LOG
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    
    # Start coordinator
    $DMTCP_ROOT/bin/dmtcp_coordinator --port 7779 >/dev/null 2>&1 &
    COORD_PID=$!
    sleep 2
    
    # Launch test
    timeout $expected_duration $test_command >/dev/null 2>&1 &
    TEST_PID=$!
    sleep 3
    
    # Take checkpoint
    if $DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779 >/dev/null 2>&1; then
        echo "  Checkpoint: SUCCESS" | tee -a $TEST_LOG
    else
        echo "  Checkpoint: FAILED" | tee -a $TEST_LOG
        FAILED_TESTS=$((FAILED_TESTS + 1))
    fi
    
    sleep 2
    
    # Restart test
    if $DMTCP_ROOT/bin/dmtcp_restart --coord-port 7779 ckpt_*.dmtcp >/dev/null 2>&1 & then
        echo "  Restart: SUCCESS" | tee -a $TEST_LOG
        sleep 2
        kill $! 2>/dev/null || true
    else
        echo "  Restart: FAILED" | tee -a $TEST_LOG
        FAILED_TESTS=$((FAILED_TESTS + 1))
    fi
    
    # Cleanup
    kill $TEST_PID 2>/dev/null || true
    kill $COORD_PID 2>/dev/null || true
    wait 2>/dev/null || true
    rm -f ckpt_*.dmtcp dmtcp_restart_script*.sh
    
    echo "  Test completed" | tee -a $TEST_LOG
    echo "" | tee -a $TEST_LOG
}

# Test suite
echo "CRAC Test Suite Results" > $TEST_LOG
echo "======================" >> $TEST_LOG
echo "" >> $TEST_LOG

# Basic functionality tests
run_test "Basic CUDA Counter" \
    "$DMTCP_ROOT/bin/dmtcp_launch --coord-port 7779 --with-plugin $PLUGIN_PATH ./test/counter" \
    30

run_test "MPI CUDA Counter" \
    "mpirun -np 2 $DMTCP_ROOT/bin/dmtcp_launch --with-plugin $PLUGIN_PATH ./test/counter_mpi" \
    30

run_test "MPI CUDA Vector Add" \
    "mpirun -np 4 $DMTCP_ROOT/bin/dmtcp_launch --with-plugin $PLUGIN_PATH ./test/mpi_cuda" \
    60

# Results
echo "Test Results Summary" | tee -a $TEST_LOG
echo "==================" >> $TEST_LOG
echo "Total tests: $TOTAL_TESTS" | tee -a $TEST_LOG
echo "Failed tests: $FAILED_TESTS" | tee -a $TEST_LOG
echo "Success rate: $(( ($TOTAL_TESTS - $FAILED_TESTS) * 100 / $TOTAL_TESTS ))%" | tee -a $TEST_LOG

if [ $FAILED_TESTS -eq 0 ]; then
    echo "All tests PASSED!" | tee -a $TEST_LOG
    exit 0
else
    echo "Some tests FAILED!" | tee -a $TEST_LOG
    exit 1
fi
```

### Performance Testing
**benchmark_tests.sh**:
```bash
#!/bin/bash

DMTCP_ROOT=${DMTCP_ROOT:-/usr/local}
PLUGIN_PATH="./libdmtcp_crac.so"
BENCHMARK_LOG="benchmark_results.log"

# Function to measure checkpoint time
measure_checkpoint_time() {
    local test_program=$1
    local iterations=$2
    
    echo "Benchmarking $test_program..." | tee -a $BENCHMARK_LOG
    echo "Iterations: $iterations" | tee -a $BENCHMARK_LOG
    
    total_time=0
    
    for i in $(seq 1 $iterations); do
        # Start coordinator
        $DMTCP_ROOT/bin/dmtcp_coordinator --port 7779 >/dev/null 2>&1 &
        COORD_PID=$!
        sleep 1
        
        # Launch test
        $DMTCP_ROOT/bin/dmtcp_launch --coord-port 7779 --with-plugin $PLUGIN_PATH ./$test_program >/dev/null 2>&1 &
        TEST_PID=$!
        sleep 2
        
        # Measure checkpoint time
        start_time=$(date +%s%N)
        if $DMTCP_ROOT/bin/dmtcp_command --checkpoint --coord-port 7779 >/dev/null 2>&1; then
            end_time=$(date +%s%N)
            checkpoint_time=$((($end_time - $start_time) / 1000000)) # milliseconds
            total_time=$(($total_time + $checkpoint_time))
            echo "  Iteration $i: ${checkpoint_time}ms" | tee -a $BENCHMARK_LOG
        else
            echo "  Iteration $i: FAILED" | tee -a $BENCHMARK_LOG
        fi
        
        # Cleanup
        kill $TEST_PID $COORD_PID 2>/dev/null || true
        wait 2>/dev/null || true
        rm -f ckpt_*.dmtcp
    done
    
    avg_time=$(($total_time / $iterations))
    echo "  Average checkpoint time: ${avg_time}ms" | tee -a $BENCHMARK_LOG
    echo "" | tee -a $BENCHMARK_LOG
}

# Run benchmarks
echo "CRAC Performance Benchmarks" > $BENCHMARK_LOG
echo "=========================" >> $BENCHMARK_LOG
echo "" >> $BENCHMARK_LOG

measure_checkpoint_time "./test/counter" 5
measure_checkpoint_time "./test/counter_mpi" 3
measure_checkpoint_time "./test/mpi_cuda" 3

echo "Benchmarking completed. Results saved to $BENCHMARK_LOG"
```

## Creating New Test Cases

### Test Case Guidelines

#### 1. Test Naming
- Use descriptive names: `feature_scenario.cu`
- Keep names under 32 characters
- Use snake_case

#### 2. Test Structure
```cuda
#include <stdio.h>
#include <unistd.h>

// Test-specific CUDA code
__global__ void test_kernel() {
    // Kernel implementation
}

int main(int argc, char **argv) {
    // Test setup
    // CUDA operations
    // Verification
    // Loop for checkpoint testing
    return 0;
}
```

#### 3. Essential Components
Every test should include:
- **CUDA Operations**: Kernels, memory allocation, data transfer
- **Verification**: Check results are correct
- **Loop**: Continuous execution for checkpoint testing
- **Output**: Progress indicators for manual verification

### Example: Memory Allocation Test
**test/memory_test.cu**:
```cuda
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__global__ void memory_test_kernel(float *data, int size) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < size) {
        data[idx] = idx * 2.0f;
    }
}

int main(int argc, char **argv) {
    const int SIZE = 1024 * 1024;
    const int BLOCK_SIZE = 256;
    
    float *d_data;
    float *h_data = (float*)malloc(SIZE * sizeof(float));
    
    // Allocate device memory
    cudaMalloc(&d_data, SIZE * sizeof(float));
    
    int iteration = 0;
    while (1) {
        iteration++;
        printf("Iteration %d: Testing memory operations\n", iteration);
        
        // Launch kernel
        int num_blocks = (SIZE + BLOCK_SIZE - 1) / BLOCK_SIZE;
        memory_test_kernel<<<num_blocks, BLOCK_SIZE>>>(d_data, SIZE);
        
        // Copy back and verify
        cudaMemcpy(h_data, d_data, SIZE * sizeof(float), cudaMemcpyDeviceToHost);
        
        // Verify results
        int errors = 0;
        for (int i = 0; i < SIZE; i++) {
            if (h_data[i] != i * 2.0f) {
                errors++;
                if (errors < 10) {
                    printf("  Error at index %d: expected %f, got %f\n", 
                           i, i * 2.0f, h_data[i]);
                }
            }
        }
        
        if (errors == 0) {
            printf("  Verification: PASSED\n");
        } else {
            printf("  Verification: FAILED (%d errors)\n", errors);
        }
        
        sleep(1);
    }
    
    // Cleanup
    cudaFree(d_data);
    free(h_data);
    return 0;
}
```

### Test Categories

#### 1. Basic Functionality
- Simple CUDA operations
- Memory allocation/deallocation
- Kernel execution
- Basic checkpoint/restart

#### 2. Resource Management
- Multiple memory allocations
- Stream operations
- Context management
- Resource cleanup

#### 3. MPI Integration
- Multi-process scenarios
- MPI + CUDA coordination
- Distributed memory
- Inter-process communication

#### 4. Stress Testing
- Large memory allocations
- Many small allocations
- Long-running computations
- Resource exhaustion

#### 5. Edge Cases
- Error conditions
- Boundary conditions
- Empty operations
- Maximum limits

### Adding Tests to Build System

#### 1. Update test/Makefile
```makefile
# Add to FILES list
FILES=counter counter_mpi mpi_cuda memory_test stream_test

# Add build rule
memory_test: memory_test.cu
	nvcc -g -O0 -o $@ $^ -cudart=shared

stream_test: stream_test.cu
	nvcc -g -O0 -o $@ $^ -cudart=shared
```

#### 2. Update Main Makefile
```makefile
# Add to check target if needed
check: ${LIBNAME}.so tests
	# Your test can be added here
```

### Test Verification

#### 1. Correctness Verification
- Check kernel results are mathematically correct
- Verify memory contents after restart
- Ensure no data corruption

#### 2. Performance Verification
- Monitor checkpoint times
- Check memory usage
- Verify no resource leaks

#### 3. Robustness Verification
- Test with different input sizes
- Test with multiple checkpoints
- Test error conditions

## Continuous Integration

### GitHub Actions Example
**.github/workflows/test.yml**:
```yaml
name: CRAC Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v2
    
    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y build-essential cuda-libraries-dev-12-4
        
    - name: Build DMTCP
      run: |
        git clone https://github.com/dmtcp/dmtcp.git
        cd dmtcp
        ./configure
        make
        echo "DMTCP_ROOT=$PWD" >> $GITHUB_ENV
        
    - name: Build CRAC
      run: |
        make DMTCP_ROOT=$DMTCP_ROOT
        
    - name: Run tests
      run: |
        chmod +x run_tests.sh
        ./run_tests.sh
        
    - name: Upload test results
      uses: actions/upload-artifact@v2
      with:
        name: test-results
        path: test_results.log
```

## Test Result Analysis

### Success Criteria
1. **Checkpoint Success**: Checkpoint files created without errors
2. **Restart Success**: Application resumes from checkpoint point
3. **Data Integrity**: All computations produce correct results
4. **Resource Cleanup**: No memory leaks or resource exhaustion

### Failure Analysis
Common failure modes and their causes:

1. **Checkpoint Failures**:
   - CUDA API errors
   - Insufficient memory
   - Resource conflicts

2. **Restart Failures**:
   - Missing checkpoint files
   - Incompatible CUDA versions
   - Resource recreation failures

3. **Data Corruption**:
   - Incorrect memory handling
   - Race conditions
   - Incomplete state restoration

## See Also

- [Development Setup](development-setup.md) - Setting up test environment
- [Adding Features](adding-features.md) - Creating testable features
- [Troubleshooting](../troubleshooting.md) - Debugging test failures
- [Architecture Overview](../architecture/overview.md) - Understanding system behavior