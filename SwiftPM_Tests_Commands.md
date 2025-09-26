# Swift Package Manager Tests - Commands Cheat Sheet

## Basic Commands

```bash
# Run all tests
swift test

# Run tests with verbose output
swift test --verbose

# Run tests with very verbose output (debug level)
swift test --very-verbose

# Run tests quietly (errors only)
swift test --quiet
```

## Test Discovery & Listing

```bash
# List all available tests
swift test list

# List tests in specifier format
swift test -l
```

## Filtering & Selection

```bash
# Run specific test by name
swift test --filter "testSuccessfulInitialization"

# Run tests matching pattern
swift test --filter "PreloadViewModelTests"

# Skip tests matching pattern
swift test --skip "PerformanceTests"

# Run tests with specifier
swift test -s "AppTests.PreloadViewModelTests/testSuccessfulInitialization"

# Run all tests in specific test class
swift test --filter "PreloadViewModelTests"

# Run all tests in specific test target and class
swift test --filter "AppTests.PreloadViewModelTests"

# Run specific test method with full path
swift test --filter "AppTests.PreloadViewModelTests/testSuccessfulInitialization"

# Run all tests containing pattern
swift test --filter "Preload"

# Run all tests starting with pattern
swift test --filter "testInitialization"

# Run all error handling tests
swift test --filter "test.*Error"

# Run performance tests only
swift test --filter "test.*Performance"
```

## Parallel Execution

```bash
# Run tests in parallel (default: disabled)
swift test --parallel

# Run tests serially
swift test --no-parallel

# Specify number of parallel workers
swift test --parallel --num-workers 4
```

## Code Coverage

```bash
# Enable code coverage
swift test --enable-code-coverage

# Disable code coverage
swift test --disable-code-coverage

# Show coverage file path
swift test --show-codecov-path
```

## Build Configuration

```bash
# Run tests in debug mode (default)
swift test --configuration debug

# Run tests in release mode
swift test --configuration release

# Skip building test target
swift test --skip-build
```

## Output & Reporting

```bash
# Generate xUnit XML report
swift test --xunit-output ./test-results.xml

# Enable testable imports (default: enabled)
swift test --enable-testable-imports

# Disable testable imports
swift test --disable-testable-imports
```

## Framework Selection

```bash
# Enable XCTest support (default: enabled)
swift test --enable-xctest

# Disable XCTest support
swift test --disable-xctest

# Enable Swift Testing support
swift test --enable-swift-testing

# Disable Swift Testing support
swift test --disable-swift-testing
```

## Package & Path Options

```bash
# Specify package path
swift test --package-path /path/to/package

# Use specific cache path
swift test --cache-path /path/to/cache

# Use specific configuration path
swift test --config-path /path/to/config
```

## Performance & Debugging

```bash
# Set number of build jobs
swift test -j 4

# Enable address sanitizer
swift test --sanitize address

# Enable thread sanitizer
swift test --sanitize thread

# Enable undefined behavior sanitizer
swift test --sanitize undefined
```

## Common Workflows

```bash
# Quick test run
swift test

# Full test run with coverage
swift test --enable-code-coverage --parallel

# Debug specific test
swift test --filter "testName" --verbose

# Performance testing
swift test --filter "Performance" --configuration release

# CI/CD friendly output
swift test --xunit-output results.xml --quiet
```

## Environment Variables

```bash
# Set Swift compiler flags
SWIFT_FLAGS="-Xswiftc -O" swift test

# Set build configuration
SWIFT_BUILD_CONFIGURATION=release swift test

# Enable parallel execution
SWIFT_TEST_PARALLEL=1 swift test
``` 