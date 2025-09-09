# Hubris Resource Management for PRoT Integration

**Author**: GitHub Copilot  
**Date**: September 8, 2025  
**Focus**: Resource ownership and management patterns for Platform Root of Trust (PRoT) integration in Hubris

## Table of Contents

1. [Introduction](#introduction)
2. [PRoT Resource Requirements](#prot-resource-requirements)
3. [Core Resource Management Philosophy](#core-resource-management-philosophy)
4. [Direct Hardware Ownership Model](#direct-hardware-ownership-model)
5. [Server-Based Resource Sharing Model](#server-based-resource-sharing-model)
6. [Hybrid Resource Management for PRoT](#hybrid-resource-management-for-prot)
7. [PRoT Communication Patterns](#prot-communication-patterns)
8. [Performance and Security Trade-offs](#performance-and-security-trade-offs)
9. [PRoT Integration Examples](#prot-integration-examples)
10. [Best Practices for PRoT Systems](#best-practices-for-prot-systems)
11. [Conclusion](#conclusion)

---

## Introduction

Platform Root of Trust (PRoT) systems require **secure, isolated, and predictable** resource management to maintain security boundaries and meet real-time attestation requirements. Hubris provides unique advantages for PRoT implementations through its **compile-time resource allocation** and **exclusive ownership** patterns.

This document explores Hubris resource management specifically in the context of **PRoT integration**, focusing on:

- **MCTP communication** for secure management protocols
- **Cryptographic hardware** isolation and access patterns  
- **Attestation services** and their resource requirements
- **Security boundaries** enforcement through resource ownership
- **Real-time guarantees** for critical security operations

---

## PRoT Resource Requirements

### **Critical PRoT Components**

A typical PRoT system in Hubris manages these key resources:

```toml
# PRoT-specific resource allocation
[tasks.mctp_controller]
uses = ["i2c2", "i2c3"]              # Dedicated MCTP buses for secure comms
priority = 1                         # Highest priority for security protocols

[tasks.crypto_engine]
uses = ["aes", "sha256", "rng"]       # Dedicated crypto hardware
priority = 1                         # Critical for attestation

[tasks.secure_storage]
uses = ["qspi", "sram_bank_2"]        # Dedicated secure storage
priority = 2                         # Protected key/cert storage

[tasks.attestation_service]
# Uses crypto_engine via secure IPC    # Software-only, high isolation
priority = 1
```

### **Security-Critical Resources**

| Resource Type | PRoT Usage | Ownership Model | Rationale |
|---------------|------------|-----------------|-----------|
| **I2C Controllers** | MCTP/SPDM communication | Direct | Real-time security protocols |
| **Crypto Hardware** | Key operations, attestation | Direct | Isolation from other tasks |
| **Secure Storage** | Root keys, certificates | Direct | Security boundary enforcement |
| **RNG/Entropy** | Cryptographic operations | Direct | Non-shareable security resource |
| **Debug Interfaces** | Manufacturing/test only | Server/Disabled | Controlled access in production |

---

## Core Resource Management Philosophy

### **Exclusive Ownership Principle**

Hubris's resource management is built on the principle that **hardware resources should be owned by exactly one task at a time**, eliminating:

- **Race conditions** - No concurrent access to shared state
- **Resource contention** - No competition for hardware resources
- **Memory corruption** - Tasks cannot interfere with each other's hardware state
- **Deadlocks** - No complex locking mechanisms needed

### **Compile-Time Allocation**

All resource allocation decisions are made at **build time** rather than runtime:

```toml
# app.toml - Resource allocation is declarative and static
[tasks.sensor_manager]
uses = ["i2c1", "adc1", "timer2"]    # Exclusive hardware ownership

[tasks.network_stack] 
uses = ["ethernet", "uart1"]         # Different hardware, no conflicts

[tasks.power_controller]
uses = ["i2c2", "gpio_bank_a"]       # Isolated from sensor_manager
```

**Benefits:**
- ✅ **No runtime allocation failures** - All resources pre-allocated
- ✅ **Predictable resource usage** - No dynamic memory management
- ✅ **Compile-time conflict detection** - Cannot accidentally share resources
- ✅ **Zero-overhead abstraction** - No runtime resource management cost

---

## Direct Hardware Ownership Model

### **Concept**

In the direct ownership model, tasks have **exclusive, unmediated access** to hardware peripherals. The task directly manipulates hardware registers and handles interrupts.

### **Architecture Pattern**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Task A        │    │   Task B        │    │   Task C        │
│   (GPIO Owner)  │    │   (I2C Owner)   │    │   (SPI Owner)   │
└─────────┬───────┘    └─────────┬───────┘    └─────────┬───────┘
          │                      │                      │
          │ Direct Access        │ Direct Access        │ Direct Access
          ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   GPIO Hardware │    │   I2C Hardware  │    │   SPI Hardware  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### **Implementation Example**

```rust
// Task configuration
[tasks.gpio_controller]
name = "gpio-controller-task"
uses = ["gpio_bank_a", "gpio_bank_b"]     // Direct hardware ownership
interrupts = {
    "exti0" = "gpio-pin-0-irq",
    "exti1" = "gpio-pin-1-irq"
}

// Task implementation
use drv_stm32xx_sys_api::{Sys, Port, Pin, Mode, OutputType};

#[no_mangle]
fn main() -> ! {
    let sys = Sys::from(SYS.get_task_id());
    
    // Direct hardware configuration - no intermediary
    sys.gpio_configure(
        Port::A,
        Pin::PIN_5,
        Mode::Output,
        OutputType::PushPull,
    );
    
    loop {
        // Direct hardware manipulation
        sys.gpio_set(Port::A, Pin::PIN_5);
        userlib::hl::sleep_for(1000);
        sys.gpio_reset(Port::A, Pin::PIN_5);
        userlib::hl::sleep_for(1000);
    }
}
```

### **Direct Ownership Advantages**

| Aspect | Benefit | Impact |
|--------|---------|--------|
| **Performance** | No IPC overhead | 1-10μs typical operation* |
| **Latency** | Predictable timing | Real-time guarantees |
| **Isolation** | Hardware faults contained | No cascade failures |
| **Simplicity** | No shared state | Easier debugging |
| **Memory** | No intermediate buffers | Lower RAM usage |

### **Direct Ownership Disadvantages**

| Aspect | Cost | Impact |
|--------|------|--------|
| **Hardware Requirements** | More peripherals needed | Higher BOM cost |
| **Code Duplication** | Multiple driver instances | Larger flash usage |
| **Resource Waste** | Underutilized peripherals | Efficiency loss |
| **Complexity** | More tasks to coordinate | System complexity |

---

## Server-Based Resource Sharing Model

### **Concept**

In the server model, a **single task owns the hardware** and provides services to other tasks through well-defined IPC interfaces. This enables resource sharing while maintaining Hubris's safety properties.

### **Architecture Pattern**

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Client A   │  │  Client B   │  │  Client C   │
│  (Sensors)  │  │  (Power)    │  │  (MCTP)     │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       │    IPC         │    IPC         │    IPC
       │   Messages     │   Messages     │   Messages
       └────────────────┼────────────────┘
                        │
              ┌─────────▼─────────┐
              │   I2C Server      │ ← Single hardware owner
              │   (Shared Access) │
              └─────────┬─────────┘
                        │ Direct Access
              ┌─────────▼─────────┐
              │   I2C Hardware    │
              └───────────────────┘
```

### **Implementation Example**

```rust
// Server task configuration
[tasks.i2c_server]
name = "drv-stm32xx-i2c-server"
uses = ["i2c1", "i2c2", "i2c3"]           // Server owns all I2C controllers
interrupts = {
    "i2c1.event" = "i2c1-event-irq",
    "i2c2.event" = "i2c2-event-irq",
    "i2c3.event" = "i2c3-event-irq"
}

// Client task configuration
[tasks.sensor_manager]
name = "sensor-manager-task"
# No direct hardware access - uses IPC to i2c_server

// Server implementation
use drv_i2c_api::*;
use userlib::hl;

fn main() -> ! {
    let mut controllers = initialize_i2c_controllers();
    let mut buffer = [0u8; 1024];
    
    loop {
        // Block until IPC message from client arrives
        hl::recv_without_notification(&mut buffer, |op, msg| {
            match op {
                Op::WriteRead => handle_write_read(&mut controllers, msg),
                Op::WriteReadBlock => handle_block_operation(&mut controllers, msg),
                Op::Reset => handle_reset(&mut controllers, msg),
                _ => ResponseCode::BadArg,
            }
            // Return value automatically sent back to client
        });
    }
}

// Client implementation
use drv_i2c_api::{I2c, I2cDevice, Controller, Port};

struct SensorManager {
    i2c: I2c,
}

impl SensorManager {
    fn read_temperature(&self) -> Result<f32, ResponseCode> {
        let device = I2cDevice::new(Controller::I2C1, Port::P0, None, 0x48);
        let mut temp_data = [0u8; 2];
        
        // IPC call to I2C server
        self.i2c.write_read(
            device,
            &[0x00],        // Temperature register
            &mut temp_data,
        )?;
        
        Ok(parse_temperature(&temp_data))
    }
}
```

### **Server Model Advantages**

| Aspect | Benefit | Impact |
|--------|---------|--------|
| **Resource Efficiency** | Shared hardware utilization | Lower hardware requirements |
| **Code Reuse** | Single driver implementation | Smaller flash footprint |
| **Coordination** | Centralized resource management | Easier bus arbitration |
| **Flexibility** | Multiple protocols on same bus | Versatile configurations |

### **Server Model Disadvantages**

| Aspect | Cost | Impact |
|--------|------|--------|
| **Performance** | IPC overhead | 20-150μs typical operation* |
| **Latency** | Synchronous IPC blocking | Task scheduling dependencies |
| **Coupling** | Clients depend on server | Failure propagation risk |
| **Complexity** | IPC protocol design | More complex interfaces |

---

## Hybrid Resource Management for PRoT

PRoT systems benefit from **hybrid approaches** that apply the optimal ownership model based on security and performance requirements:

### **PRoT Decision Matrix**

| PRoT Component | Ownership Model | Security Rationale |
|----------------|-----------------|-------------------|
| **MCTP Transport** | Direct | Real-time security protocols, isolation |
| **Crypto Engines** | Direct | Security boundary enforcement |
| **Attestation Data** | Direct | Non-shareable security context |
| **Manufacturing Test** | Server | Controlled access, debug features |
| **Status/Monitoring** | Server | Non-security-critical telemetry |

### **PRoT Hybrid Configuration Example**

```toml
# Security-critical: Direct ownership
[tasks.mctp_controller]
name = "mctp-controller-task"
uses = ["i2c2", "i2c3"]              # Dedicated secure communication buses
priority = 1                         # Highest priority for security
interrupts = {
    "i2c2.event" = "mctp-secure-event",
    "i2c3.event" = "mctp-backup-event"
}

[tasks.crypto_engine]
name = "crypto-engine-task"  
uses = ["aes", "sha256", "rng", "ecc"] # Dedicated crypto hardware
priority = 1                         # Critical for attestation operations

# Non-security-critical: Server model
[tasks.telemetry_server]
name = "telemetry-server"
uses = ["uart1"]                     # Shared diagnostic interface
priority = 3                         # Lower priority for monitoring

# Clients use telemetry server for non-critical data
[tasks.health_monitor]
name = "health-monitor"
# Uses telemetry_server via IPC for status reporting
```

### **Communication Patterns**

```rust
// Direct ownership for MCTP (performance-critical)
impl MctpController {
    fn handle_urgent_power_command(&mut self) {
        // Direct hardware access - minimal latency
        self.i2c2.operate_as_target(/* ... */);
    }
}

// Server-based for sensors (less critical)
impl SensorManager {
    fn read_ambient_temperature(&self) -> Result<f32, Error> {
        // IPC to shared I2C server - acceptable latency
        self.i2c_client.write_read(/* ... */)
    }
}
```

---

## Configuration and Declaration

### **Static Resource Declaration**

Resources are declared statically in the application configuration:

```toml
[tasks.task_name]
name = "task-binary-name"
features = ["feature1", "feature2"]
priority = 3                         # Task scheduling priority
uses = ["resource1", "resource2"]    # Hardware resources owned
interrupts = {                       # Interrupt ownership
    "interrupt_name" = "handler_name"
}
task-slots = ["slot1", "slot2"]      # IPC endpoints
stacksize = 2048                     # Stack allocation
```

### **Resource Types**

#### **Hardware Peripherals**
```toml
uses = [
    "i2c1", "i2c2",                 # I2C controllers
    "spi1", "spi3",                 # SPI controllers  
    "uart1", "uart2",               # UART controllers
    "adc1",                         # ADC controllers
    "timer1", "timer3",             # Timer peripherals
    "gpio_bank_a", "gpio_bank_b",   # GPIO banks
    "dma1_ch1", "dma1_ch2",         # DMA channels
]
```

#### **Interrupts**
```toml
interrupts = {
    "i2c1.event" = "i2c1-event-handler",
    "i2c1.error" = "i2c1-error-handler",
    "timer1.update" = "timer-update-handler",
    "exti5" = "gpio-pin-5-handler",
}
```

#### **Memory Regions**
```toml
extern-regions = ["flash_bank_2", "sram_bank_3"]
```

### **Compile-Time Validation**

Hubris validates resource allocation at build time:

```bash
# Build-time conflict detection
$ cargo xtask build app.toml

Error: Resource conflict detected
  Task 'sensor_manager' requests 'i2c1'
  Task 'power_controller' requests 'i2c1'
  
  Resources must be owned by exactly one task.
  
  Suggestion: 
  - Use different I2C controllers, or
  - Create an I2C server task for shared access
```

---

## PRoT Communication Patterns

### **1. Secure MCTP Communication**

PRoT tasks with direct I2C ownership handle secure MCTP protocols:

```rust
// PRoT MCTP controller - direct hardware ownership
impl ProtMctpController {
    fn handle_spdm_request(&mut self, request: &SpdmMessage) -> SpdmResponse {
        // Direct hardware access for real-time security protocols
        match request.request_type {
            SpdmRequestType::GetVersion => self.handle_get_version(),
            SpdmRequestType::GetCapabilities => self.handle_get_capabilities(),
            SpdmRequestType::Challenge => {
                // Time-critical attestation - direct crypto access
                self.crypto_engine.generate_attestation(request.nonce)
            }
        }
    }
    
    fn handle_mctp_interrupt(&mut self) {
        // Direct I2C interrupt handling for secure transport
        let message = self.i2c_secure.read_mctp_packet();
        self.process_secure_message(message);
    }
}
```

### **2. Secure IPC for PRoT Services**

PRoT services use **authenticated IPC** for secure inter-task communication:

```rust
// PRoT Attestation Service - secure IPC with crypto engine
impl ProtAttestationService {
    fn generate_platform_certificate(&self, nonce: &[u8]) -> Result<Certificate, ProtError> {
        // Secure IPC call to dedicated crypto engine
        let signature_request = AttestationRequest {
            measurement_data: self.collect_platform_measurements(),
            nonce: nonce.to_vec(),
            key_id: ROOT_ATTESTATION_KEY,
        };
        
        // Authenticated IPC - only PRoT tasks can access crypto engine
        let signature = self.crypto_client.sign_attestation(signature_request)?;
        
        self.build_certificate(signature)
    }
}

// PRoT Crypto Engine - validates caller identity
fn handle_crypto_request(&mut self, caller_id: TaskId, op: CryptoOp, msg: &Message) -> ResponseCode {
    // Verify caller is authorized PRoT service
    if !self.is_authorized_prot_caller(caller_id) {
        return ResponseCode::AccessDenied;
    }
    
    match op {
        CryptoOp::SignAttestation => self.perform_attestation_signature(msg),
        CryptoOp::DeriveKey => self.perform_key_derivation(msg),
        _ => ResponseCode::BadArg,
    }
}
```

#### **Understanding `recv_without_notification`**

This is Hubris's primary server-side IPC pattern:

```rust
pub fn recv_without_notification<F, R>(
    buffer: &mut [u8], 
    f: F
) -> R 
where
    F: FnOnce(u32, userlib::RecvMessage) -> R
```

**What it does:**
1. **Blocks the task** until an IPC message arrives from a client
2. **Receives the message** directly into the provided buffer (zero-copy)
3. **Calls the closure** with `(operation_code, message)`
4. **Automatically sends response** back to the client
5. **Returns to blocking** for the next message

**The "without_notification" name** indicates it only handles IPC messages and ignores **non-hardware notifications** (custom task notifications, timers, etc.) while blocked. **Hardware interrupts are still processed** - they wake the task and must be handled separately.

#### **Correct I2C Server Pattern**

I2C servers **must** handle hardware interrupts to know when operations complete:

```rust
fn main() -> ! {
    let mut server = I2cServer::new();
    let mut buffer = [0u8; 1024];
    
    loop {
        // Wait for any notification (IPC or hardware interrupt)
        let notification = sys_recv_notification();
        
        match notification {
            // CRITICAL: Handle I2C hardware interrupts
            I2C1_EVENT_IRQ => server.handle_i2c1_event_interrupt(),
            I2C1_ERROR_IRQ => server.handle_i2c1_error_interrupt(),
            I2C2_EVENT_IRQ => server.handle_i2c2_event_interrupt(),
            I2C2_ERROR_IRQ => server.handle_i2c2_error_interrupt(),
            
            // Handle client IPC requests
            _ => {
                recv_without_notification(&mut buffer, |op, msg| {
                    server.handle_client_request(op, msg)
                });
            }
        }
    }
}
```

#### **Why I2C Interrupts Are Critical**

I2C operations are **asynchronous** at the hardware level:

```rust
impl I2cServer {
    fn handle_write_read(&mut self, msg: &Message) -> ResponseCode {
        let controller = &mut self.controllers[device.controller];
        
        // Start I2C transaction (hardware begins operation)
        controller.start_write_read(device.address, write_data, read_len);
        
        // Hardware interrupt will fire when:
        // - Transaction completes successfully
        // - Error occurs (NACK, bus error, timeout)
        // - DMA transfer completes
        
        ResponseCode::Success // Return immediately, interrupt handles completion
    }
    
    fn handle_i2c_event_interrupt(&mut self) {
        // Hardware signaled operation complete
        let controller = &mut self.controllers[interrupt_source];
        
        if controller.transaction_complete() {
            // Wake any blocked clients, update transaction state
            controller.complete_pending_transaction();
        }
    }
    
    fn handle_i2c_error_interrupt(&mut self) {
        // Hardware signaled error condition
        let controller = &mut self.controllers[interrupt_source];
        
        let error = controller.get_error_status();
        controller.handle_error_recovery(error);
    }
}
```

#### **Simple IPC-Only Server Example**

For servers that **don't** use hardware interrupts (rare):

```rust
fn main() -> ! {
    let mut simple_server = SimpleServer::new();
    let mut buffer = [0u8; 1024];
    
    loop {
        // Block until client IPC message arrives
        recv_without_notification(&mut buffer, |op, msg| {
            // op: u32 - operation code from client
            // msg: RecvMessage - contains request data and response buffers
            
            match op {
                Op::GetStatus => {
                    // Synchronous operation, no hardware interrupts needed
                    let status = simple_server.get_current_status();
                    msg.write_response_data(&status).unwrap();
                    ResponseCode::Success
                }
                Op::SetConfig => {
                    // Pure software operation
                    let config = msg.read_request_data().unwrap();
                    simple_server.update_config(config);
                    ResponseCode::Success
                }
                _ => ResponseCode::BadArg,
            }
            // Whatever is returned here gets sent to client automatically
        });
        // Loop back to wait for next message
    }
}
```



#### **Key IPC Characteristics**

- **Synchronous** - Client blocks until server completes operation
- **Zero-copy** - Data passed by reference, not copied between tasks
- **Rendezvous-based** - Direct communication, no message queues
- **Type-safe operations** - Operation codes usually cast from enums
- **Automatic response** - Server's return value sent back to client
- **Single-threaded** - One message processed at a time per server

### **3. Notification System**

For event-driven communication and hardware interrupt handling:

```rust
// Hardware interrupt triggers notification
// (Configured in app.toml interrupt mapping)

// Task waits for notifications
let notification = userlib::sys_recv_notification();
match notification {
    TIMER_INTERRUPT => process_timer_event(),
    I2C_INTERRUPT => handle_i2c_completion(),
    GPIO_INTERRUPT => process_gpio_change(),
    IPC_MESSAGE_AVAILABLE => {
        // Handle pending IPC message
        recv_without_notification(&mut buffer, |op, msg| {
            handle_client_request(op, msg)
        });
    }
    _ => handle_unexpected_event(),
}

// Tasks can also send notifications to each other
userlib::sys_post(target_task_id, CUSTOM_EVENT_CODE);
```
```

### **4. Shared Memory Regions**

For high-performance data sharing:

```rust
// Shared memory configuration
[tasks.producer]
extern-regions = ["shared_buffer"]

[tasks.consumer] 
extern-regions = ["shared_buffer"]

// Producer writes data
let shared_buf = unsafe { SHARED_BUFFER.as_mut() };
shared_buf.copy_from_slice(&sensor_data);
sys_post(consumer_task, NOTIFICATION_NEW_DATA);

// Consumer reads data
let shared_buf = unsafe { SHARED_BUFFER.as_ref() };
process_sensor_data(shared_buf);
```

---

## Performance and Safety Trade-offs

### **Performance Comparison**

| Operation | Direct Ownership | Server Model | Hybrid |
|-----------|------------------|--------------|--------|
| **Hardware Access** | 5-10μs* | 50-100μs* | 5-100μs* |
| **Memory Usage** | Lower | Higher | Mixed |
| **Flash Usage** | Higher | Lower | Mixed |
| **Predictability** | High | Medium | High |

***Latency estimates based on typical ARM Cortex-M systems. Actual performance depends on:**
- **CPU frequency** (100-400MHz typical)
- **Hardware peripheral speeds** (I2C: 100kHz-3.4MHz, SPI: 1-50MHz)
- **Message complexity** and data payload size
- **System load** and task priorities
- **Memory architecture** (TCM vs SRAM vs external)

**Direct access components:**
- Register operations: 1-3μs
- Simple peripheral transactions: 3-7μs

**IPC overhead components:**
- Context switches: 10-20μs each (2x per round-trip)
- Message handling: 10-30μs
- Synchronous rendezvous: 5-15μs

### **Measuring Actual Performance**

To get accurate latency measurements for your specific Hubris system:

```rust
// Direct hardware timing measurement
use cortex_m_semihosting::hprintln;
use cortex_m::peripheral::DWT;

fn measure_direct_i2c_operation() {
    let start = DWT::cycle_count();
    
    // Direct I2C operation
    i2c_controller.write_read(address, &write_data, &mut read_buffer);
    
    let end = DWT::cycle_count();
    let cycles = end.wrapping_sub(start);
    let microseconds = cycles * 1_000_000 / system_clock_hz;
    
    hprintln!("Direct I2C operation: {}μs", microseconds);
}

// IPC timing measurement  
fn measure_ipc_i2c_operation() {
    let start = DWT::cycle_count();
    
    // IPC call to I2C server
    let result = i2c_client.write_read(device, &write_data, &mut read_buffer);
    
    let end = DWT::cycle_count();
    let cycles = end.wrapping_sub(start);
    let microseconds = cycles * 1_000_000 / system_clock_hz;
    
    hprintln!("IPC I2C operation: {}μs", microseconds);
}
```

**Benchmark Configuration:**
```toml
# Enable cycle counter for precise timing
[dependencies]
cortex-m = { version = "0.7", features = ["critical-section-single-core"] }

# Build with timing optimizations
[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

### **Safety Comparison**

| Aspect | Direct Ownership | Server Model | Hybrid |
|--------|------------------|--------------|--------|
| **Isolation** | Maximum | Good | Maximum |
| **Fault Containment** | Excellent | Good | Excellent |
| **Resource Conflicts** | Impossible | Managed | Minimized |
| **Attack Surface** | Minimal | Larger | Minimal |

### **Development Complexity**

| Phase | Direct Ownership | Server Model | Hybrid |
|-------|------------------|--------------|--------|
| **Design** | Simple | Complex | Medium |
| **Implementation** | Medium | Complex | Complex |
| **Testing** | Simple | Complex | Complex |
| **Debugging** | Easy | Hard | Medium |
| **Maintenance** | Easy | Medium | Medium |

---

## PRoT Integration Examples

### **Example 1: Server Management Controller with PRoT**

```toml
# BMC with integrated Platform Root of Trust

# Core PRoT Components - Maximum Security Isolation
[tasks.prot_mctp_controller]
name = "prot-mctp-controller"
uses = ["i2c2", "i2c3"]              # Dedicated secure MCTP buses
priority = 1                         # Highest priority
interrupts = {
    "i2c2.event" = "prot-mctp-primary",
    "i2c3.event" = "prot-mctp-backup"
}

[tasks.prot_crypto_engine]
name = "prot-crypto-engine"
uses = ["aes", "sha256", "rng", "ecc"] # Dedicated crypto hardware
priority = 1                         # Critical for attestation
interrupts = {
    "aes.complete" = "crypto-operation-complete"
}

[tasks.prot_attestation_service]
name = "prot-attestation-service"
# Pure software task - uses crypto_engine via secure IPC
priority = 1                         # Real-time attestation responses

[tasks.prot_secure_storage]
name = "prot-secure-storage"
uses = ["qspi_bank_1"]               # Dedicated secure flash
priority = 2                         # Root keys and certificates

# BMC System Components - Shared Resources
[tasks.bmc_sensor_server]
name = "bmc-sensor-server"
uses = ["i2c1", "i2c4", "adc1"]      # Standard BMC sensor buses
priority = 3                         # Lower priority than PRoT

[tasks.bmc_network_stack]
name = "bmc-network-stack"
uses = ["ethernet"]                  # Management network
priority = 3

[tasks.bmc_fan_controller]
name = "bmc-fan-controller"
# Uses bmc_sensor_server for temperature readings
```

### **Example 2: Standalone PRoT Module**

```toml
# Dedicated Platform Root of Trust implementation

[tasks.prot_mctp_primary]
name = "prot-mctp-primary"
uses = ["i2c1"]                      # Primary secure communication
priority = 1
interrupts = {
    "i2c1.event" = "mctp-primary-event",
    "i2c1.error" = "mctp-primary-error"
}

[tasks.prot_spdm_engine]
name = "prot-spdm-engine" 
uses = ["aes", "ecdsa", "sha384"]     # SPDM-specific crypto
priority = 1
interrupts = {
    "ecdsa.complete" = "signature-complete"
}

[tasks.prot_dice_engine]
name = "prot-dice-engine"
uses = ["rng", "otp_fuses"]          # Device Identity composition
priority = 1

[tasks.prot_measurement_service]
name = "prot-measurement-service"
# Software-only task for platform measurements
priority = 1

# Manufacturing/Debug (disabled in production)
[tasks.manufacturing_test]
name = "manufacturing-test"
uses = ["uart_debug"]               # Debug interface
priority = 4                        # Lowest priority
features = ["debug-build-only"]     # Conditionally compiled
```

### **Example 3: PRoT-Enhanced Network Interface Card**

```toml
# SmartNIC with integrated Platform Root of Trust

# PRoT Security Components
[tasks.nic_prot_controller]
name = "nic-prot-controller"
uses = ["i2c_secure"]               # Dedicated secure bus
priority = 1
interrupts = {
    "i2c_secure.event" = "prot-secure-event"
}

[tasks.nic_attestation_engine]
name = "nic-attestation-engine"
uses = ["crypto_accel", "secure_ram"] # Hardware crypto + secure storage
priority = 1

# NIC Primary Functions
[tasks.network_data_plane]
name = "network-data-plane"
uses = ["ethernet_mac", "dma_engine"] # High-performance networking
priority = 2                         # Real-time packet processing

[tasks.network_control_plane]
name = "network-control-plane"
uses = ["pcie_config"]              # PCIe configuration space
priority = 3

# Management (shared resources)
[tasks.nic_management_server]
name = "nic-management-server"
uses = ["i2c_mgmt"]                 # Shared management bus
priority = 4
```

---

## Implementation Patterns

### **Pattern 1: Hardware Abstraction Layer (HAL)**

For direct ownership tasks:

```rust
// hal/src/gpio.rs
pub struct GpioController {
    port_a: stm32h7xx_hal::gpio::gpioa::Parts,
    port_b: stm32h7xx_hal::gpio::gpiob::Parts,
}

impl GpioController {
    pub fn new() -> Self {
        let dp = stm32h7xx_hal::pac::Peripherals::take().unwrap();
        let gpioa = dp.GPIOA.split();
        let gpiob = dp.GPIOB.split();
        
        Self {
            port_a: gpioa,
            port_b: gpiob,
        }
    }
    
    pub fn configure_output(&mut self, pin: GpioPin) -> OutputPin {
        match pin {
            GpioPin::PA5 => self.port_a.pa5.into_push_pull_output(),
            GpioPin::PB3 => self.port_b.pb3.into_push_pull_output(),
            // ... other pins
        }
    }
}
```

### **Pattern 2: Server Task Implementation**

For server-based resource sharing:

```rust
// server/src/main.rs
use drv_i2c_api::*;

struct I2cServer {
    controllers: [I2cController; MAX_CONTROLLERS],
    mux_state: MuxState,
    statistics: ServerStatistics,
}

impl I2cServer {
    fn handle_client_request(&mut self, op: Op, msg: &Message) -> ResponseCode {
        match op {
            Op::WriteRead => self.handle_write_read(msg),
            Op::WriteReadBlock => self.handle_block_operation(msg),
            Op::Reset => self.handle_reset(msg),
        }
    }
    
    fn handle_write_read(&mut self, msg: &Message) -> ResponseCode {
        let (device, write_len, read_len) = self.parse_request(msg)?;
        let controller = self.get_controller(device.controller)?;
        
        controller.write_read(
            device.address,
            write_len,
            |pos| msg.read_write_data(pos),
            ReadLength::Fixed(read_len),
            |pos, byte| msg.write_read_data(pos, byte),
        )
    }
}

fn main() -> ! {
    let mut server = I2cServer::new();
    let mut buffer = [0u8; 1024];
    
    loop {
        // Block until IPC message arrives from client
        hl::recv_without_notification(&mut buffer, |op, msg| {
            // Process client request synchronously
            server.handle_client_request(op, msg)
            // Client unblocks when this returns
        });
    }
}
```

### **Pattern 3: Client API Implementation**

For tasks using server-based resources:

```rust
// client-api/src/lib.rs
use userlib::*;

pub struct I2cClient {
    server_task_id: TaskId,
}

impl I2cClient {
    pub fn new() -> Self {
        Self {
            server_task_id: I2C_SERVER.get_task_id(),
        }
    }
    
    pub fn write_read(
        &self,
        device: I2cDevice,
        write_data: &[u8],
        read_buffer: &mut [u8],
    ) -> Result<(), ResponseCode> {
        let mut request = WriteReadRequest {
            device,
            write_len: write_data.len(),
            read_len: read_buffer.len(),
        };
        
        let rc = hl::send_leased_rw(
            self.server_task_id,
            Op::WriteRead as u32,
            &request,
            &[write_data],
            &mut [read_buffer],
        )?;
        
        ResponseCode::from_u32(rc).ok_or(ResponseCode::BadResponse)
    }
}
```

### **Pattern 4: Hybrid Resource Manager**

For complex systems with mixed ownership models:

```rust
// hybrid-manager/src/lib.rs
pub enum ResourceAccess {
    Direct(DirectResource),
    Server(ServerResource),
}

pub struct HybridResourceManager {
    direct_resources: HashMap<ResourceId, DirectResource>,
    server_clients: HashMap<ServerId, ServerClient>,
}

impl HybridResourceManager {
    pub fn access_resource(&self, id: ResourceId) -> ResourceAccess {
        if let Some(direct) = self.direct_resources.get(&id) {
            ResourceAccess::Direct(direct.clone())
        } else if let Some(server) = self.find_server_for_resource(id) {
            ResourceAccess::Server(server.clone())
        } else {
            panic!("Resource {} not available", id);
        }
    }
}
```

---

## Best Practices for PRoT Systems

### **Choosing the Right Model for PRoT**

#### **Use Direct Ownership When:**
- ✅ **Security isolation** is paramount (crypto engines, secure storage)
- ✅ **Real-time attestation** responses required
- ✅ **MCTP/SPDM protocols** need deterministic timing
- ✅ **Root keys and certificates** must be isolated
- ✅ **Hardware RNG/entropy** sources for cryptographic operations

#### **Use Server Model When:**
- ✅ **Manufacturing/debug interfaces** need controlled access
- ✅ **Status monitoring** for non-security-critical telemetry
- ✅ **Shared diagnostic resources** during development
- ✅ **Non-critical I2C devices** (fans, LEDs, general sensors)

#### **PRoT Security Guidelines**

```toml
# ✅ Good: Clear security boundaries
[tasks.prot_crypto_engine]
uses = ["aes", "ecdsa", "rng"]       # Dedicated crypto hardware
priority = 1                        # Highest priority for security

[tasks.prot_secure_storage]
uses = ["qspi_secure"]               # Dedicated secure flash
priority = 1                        # Isolated key storage

# ✅ Good: Non-security resources can be shared
[tasks.diagnostic_server]
name = "diagnostic-server"
uses = ["uart_debug"]               # Shared debug interface
priority = 4                        # Lowest priority
features = ["debug-build-only"]     # Production disabled

# ❌ Bad: Security-critical resources shared
[tasks.crypto_server]
uses = ["aes", "rng"]               # Don't share crypto hardware!
# Multiple tasks accessing crypto = security boundary violation
```

### **PRoT Task Priority Guidelines**

```toml
# Priority 1: Security-critical, real-time PRoT services
[tasks.prot_mctp_controller]
priority = 1                        # Real-time security protocols

[tasks.prot_attestation_service] 
priority = 1                        # Time-sensitive attestation

# Priority 2: Important but not real-time security services
[tasks.prot_measurement_service]
priority = 2                        # Platform measurement collection

# Priority 3+: Non-security-critical system functions
[tasks.health_monitor]
priority = 3                        # System health telemetry

[tasks.debug_interface]
priority = 4                        # Debug/manufacturing only
```

---

## Conclusion

Hubris's resource management models provide **critical advantages for Platform Root of Trust integration**, enabling secure, isolated, and predictable execution of security-critical operations:

### **Key PRoT Benefits**

1. **Security Isolation** - Direct ownership ensures crypto hardware and secure storage cannot be accessed by unauthorized tasks, enforcing security boundaries at the hardware level.

2. **Real-Time Attestation** - Direct hardware access enables deterministic timing for MCTP/SPDM protocol responses, meeting strict timing requirements for attestation protocols.

3. **Compile-Time Security Verification** - Static resource allocation prevents runtime security violations and ensures security-critical resources remain isolated.

4. **Hybrid Security Model** - Critical security functions use direct ownership while non-security functions can use efficient server models.

### **PRoT Integration Recommendations**

| PRoT Component | Recommended Model | Key Benefit |
|----------------|-------------------|-------------|
| **MCTP Transport** | Direct Ownership | Real-time security protocol handling |
| **Crypto Engines** | Direct Ownership | Hardware security boundary enforcement |
| **Secure Storage** | Direct Ownership | Root key and certificate isolation |
| **Debug Interfaces** | Server-Based | Controlled access, production disabling |
| **Status Monitoring** | Server-Based | Resource efficiency for non-critical telemetry |

### **Future PRoT Considerations**

As PRoT requirements evolve, Hubris's flexible resource management enables:

- **Security model hardening** through additional hardware isolation
- **Performance optimization** for emerging attestation protocols  
- **Compliance adaptation** for new security standards and certifications
- **Threat model evolution** to address new attack vectors

The combination of **compile-time verification**, **hardware-enforced isolation**, and **deterministic execution** makes Hubris particularly well-suited for implementing Platform Root of Trust functionality in critical systems.

---

**This document provides the foundation for PRoT integration in Hubris. For specific MCTP implementation guidance, refer to the companion MCTP server extension documentation.**
