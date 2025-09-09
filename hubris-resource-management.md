# Hubris Resource Management Models

## Table of Contents

1. [Introduction](#introduction)
2. [Core Resource Management Philosophy](#core-resource-management-philosophy)
3. [Direct Hardware Ownership Model](#direct-hardware-ownership-model)
4. [Server-Based Resource Sharing Model](#server-based-resource-sharing-model)
5. [Hybrid Resource Management](#hybrid-resource-management)
6. [Configuration and Declaration](#configuration-and-declaration)
7. [Inter-Task Communication Patterns](#inter-task-communication-patterns)
8. [Performance and Safety Trade-offs](#performance-and-safety-trade-offs)
9. [Real-World Examples](#real-world-examples)
10. [Implementation Patterns](#implementation-patterns)
11. [Best Practices](#best-practices)
12. [Conclusion](#conclusion)

---

## Introduction

Hubris takes a fundamentally different approach to resource management compared to traditional operating systems. Instead of kernel-mediated shared access to hardware resources, Hubris implements **compile-time resource allocation** with **exclusive ownership** patterns that eliminate entire classes of runtime errors and provide strong isolation guarantees.

This document explores the various resource management models available in Hubris, their trade-offs, and practical implementation patterns.

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
| **Performance** | No IPC overhead | ~5-10μs vs ~50-100μs |
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
        // Process IPC messages from clients
        hl::recv_without_notification(&mut buffer, |op, msg| {
            match op {
                Op::WriteRead => handle_write_read(&controllers, msg),
                Op::WriteReadBlock => handle_block_operation(&controllers, msg),
                // Handle various I2C operations
            }
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
| **Performance** | IPC overhead | ~50-100μs vs ~5-10μs |
| **Latency** | Message queuing delays | Less predictable timing |
| **Coupling** | Clients depend on server | Failure propagation risk |
| **Complexity** | IPC protocol design | More complex interfaces |

---

## Hybrid Resource Management

### **Concept**

Hybrid models combine direct ownership for performance-critical resources with server-based sharing for less critical or naturally shared resources.

### **Decision Matrix**

| Resource Type | Ownership Model | Rationale |
|---------------|-----------------|-----------|
| **Time-critical I/O** | Direct | Real-time requirements |
| **High-frequency operations** | Direct | Performance optimization |
| **Security-sensitive** | Direct | Isolation requirements |
| **Shared peripherals** | Server | Resource efficiency |
| **Protocol stacks** | Server | Code reuse benefits |
| **Debug interfaces** | Server | Development convenience |

### **Implementation Example**

```toml
# Hybrid resource allocation
[tasks.mctp_controller]
name = "mctp-controller-task"
uses = ["i2c2", "i2c3"]              # Direct ownership for MCTP buses
priority = 2                         # High priority for real-time

[tasks.i2c_server]
name = "drv-stm32xx-i2c-server"
uses = ["i2c1", "i2c4"]              # Server model for shared buses
priority = 3                         # Standard priority

[tasks.sensor_manager]
name = "sensor-manager-task"
# Uses i2c_server via IPC for non-critical sensors

[tasks.power_controller]
name = "power-controller-task"
# Uses mctp_controller via IPC for critical power management
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

## Inter-Task Communication Patterns

### **1. Direct Hardware Communication**

Tasks with direct ownership communicate through hardware mechanisms:

```rust
// Task A writes to shared memory region
task_a.write_shared_flag(SharedFlag::DataReady);

// Task B polls hardware register
while !task_b.read_shared_flag(SharedFlag::DataReady) {
    // Wait for hardware signal
}
```

### **2. IPC Message Passing**

Server-client communication uses Hubris's IPC system:

```rust
// Client sends message to server
let response: ResponseCode = client.send_message(
    ServerOp::I2cWrite,
    &request_data,
    &mut response_buffer,
)?;

// Server receives and processes message
hl::recv_without_notification(&mut buffer, |op, msg| {
    match op {
        ServerOp::I2cWrite => handle_i2c_write(msg),
        ServerOp::I2cRead => handle_i2c_read(msg),
    }
});
```

### **3. Notification System**

For event-driven communication:

```rust
// Producer task sends notification
userlib::sys_post(consumer_task_id, NOTIFICATION_DATA_READY);

// Consumer task waits for notification
let notification = userlib::sys_recv_notification();
match notification {
    NOTIFICATION_DATA_READY => process_new_data(),
    _ => handle_other_events(),
}
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
| **Hardware Access** | 5-10μs | 50-100μs | 5-100μs |
| **Memory Usage** | Lower | Higher | Mixed |
| **Flash Usage** | Higher | Lower | Mixed |
| **Predictability** | High | Medium | High |

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

## Real-World Examples

### **Example 1: Server Rack Management Controller**

```toml
# High-performance BMC configuration using hybrid model

[tasks.mctp_controller]
name = "mctp-controller"
uses = ["i2c2", "i2c3"]              # Direct ownership for MCTP
priority = 1                         # Highest priority
interrupts = {
    "i2c2.event" = "mctp-i2c2-event",
    "i2c3.event" = "mctp-i2c3-event"
}

[tasks.sensor_server]  
name = "sensor-server"
uses = ["i2c1", "i2c4", "adc1"]      # Direct ownership for time-critical sensors
priority = 2
interrupts = {
    "i2c1.event" = "sensor-i2c-event",
    "adc1.eoc" = "adc-conversion-complete"
}

[tasks.i2c_shared_server]
name = "drv-stm32xx-i2c-server"
uses = ["i2c5", "i2c6"]              # Server model for non-critical devices
priority = 3

[tasks.network_stack]
name = "network-stack"
uses = ["ethernet"]                  # Direct ownership for network
priority = 2

[tasks.storage_manager]
name = "storage-manager"
uses = ["spi1", "spi2"]              # Direct ownership for flash/eMMC
priority = 3

[tasks.power_monitor]
name = "power-monitor"
# Uses sensor_server via IPC for power readings

[tasks.fan_controller]
name = "fan-controller"
# Uses i2c_shared_server via IPC for fan control

[tasks.led_controller]
name = "led-controller"
# Uses i2c_shared_server via IPC for status LEDs
```

### **Example 2: Industrial IoT Gateway**

```toml
# Industrial gateway with mixed connectivity

[tasks.modbus_controller]
name = "modbus-controller"
uses = ["uart1", "uart2", "uart3"]   # Direct ownership for RS485/Modbus
priority = 1

[tasks.ethernet_stack]
name = "ethernet-stack"  
uses = ["ethernet"]                  # Direct ownership for network
priority = 1

[tasks.wireless_controller]
name = "wireless-controller"
uses = ["spi3"]                      # Direct ownership for WiFi/cellular
priority = 2

[tasks.sensor_aggregator]
name = "sensor-aggregator"
uses = ["i2c1"]                      # Direct ownership for local sensors
priority = 2

[tasks.display_controller]
name = "display-controller"
uses = ["spi1", "spi2"]              # Direct ownership for LCD/OLED
priority = 3

[tasks.config_manager]
name = "config-manager"
uses = ["qspi"]                      # Direct ownership for config storage
priority = 3

[tasks.debug_server]
name = "debug-server"
uses = ["uart4"]                     # Direct ownership for debug console
priority = 4
```

### **Example 3: Automotive ECU**

```toml
# Automotive Electronic Control Unit

[tasks.can_controller]
name = "can-controller"
uses = ["can1", "can2", "can3"]      # Direct ownership for CAN bus
priority = 1                         # Real-time critical

[tasks.sensor_fusion]
name = "sensor-fusion"
uses = ["spi1", "spi2", "i2c1"]      # Direct ownership for IMU/sensors
priority = 1                         # Real-time critical

[tasks.actuator_controller]
name = "actuator-controller"
uses = ["timer1", "timer2", "pwm1"]  # Direct ownership for motor control
priority = 1                         # Real-time critical

[tasks.diagnostic_server]
name = "diagnostic-server"
uses = ["uart1"]                     # Direct ownership for OBD-II
priority = 2

[tasks.flash_manager]
name = "flash-manager"
uses = ["qspi"]                      # Direct ownership for data logging
priority = 3

[tasks.shared_i2c_server]
name = "shared-i2c-server"
uses = ["i2c2"]                      # Server model for non-critical I2C
priority = 3

[tasks.climate_control]
name = "climate-control"
# Uses shared_i2c_server for temperature sensors

[tasks.infotainment_bridge]
name = "infotainment-bridge"  
# Uses shared_i2c_server for audio/display interfaces
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
        hl::recv_without_notification(&mut buffer, |op, msg| {
            server.handle_client_request(op, msg)
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

## Best Practices

### **Choosing the Right Model**

#### **Use Direct Ownership When:**
- ✅ **Real-time constraints** are critical
- ✅ **Performance** is paramount
- ✅ **Isolation** is required for security
- ✅ **Hardware is abundant** and dedicated use makes sense
- ✅ **Simple protocols** that don't benefit from sharing

#### **Use Server Model When:**
- ✅ **Resource efficiency** is important
- ✅ **Complex protocols** benefit from centralization
- ✅ **Multiple clients** need the same resource
- ✅ **Hardware is limited** and must be shared
- ✅ **Non-critical timing** requirements

#### **Use Hybrid Model When:**
- ✅ **Mixed requirements** exist in the same system
- ✅ **Critical and non-critical** operations coexist
- ✅ **Gradual migration** from one model to another
- ✅ **Different subsystems** have different constraints

### **Design Guidelines**

#### **For Direct Ownership:**
```rust
// ✅ Good: Clear resource boundaries
[tasks.mctp_controller]
uses = ["i2c2", "i2c3"]              # Dedicated MCTP buses

[tasks.sensor_manager]
uses = ["i2c1", "adc1"]              # Dedicated sensor buses

// ❌ Bad: Resource conflicts
[tasks.task_a]
uses = ["i2c1"]

[tasks.task_b] 
uses = ["i2c1"]                      # Conflict! Build will fail
```

#### **For Server Design:**
```rust
// ✅ Good: Well-defined operations
pub enum I2cOp {
    WriteRead,
    WriteReadBlock,
    Reset,
    GetStatistics,
}

// ✅ Good: Clear error handling
pub enum I2cResponseCode {
    Success,
    NoDevice,
    BusError,
    Timeout,
}

// ❌ Bad: Leaky abstractions
pub enum I2cOp {
    WriteRegister,      // Too specific
    ManipulateHardware, // Too generic
}
```

#### **For Error Handling:**
```rust
// ✅ Good: Comprehensive error types
#[derive(Debug, Copy, Clone)]
pub enum ResourceError {
    NotAvailable,
    AccessDenied,
    HardwareFault,
    ProtocolError,
    Timeout,
}

// ✅ Good: Graceful degradation
impl ResourceManager {
    fn try_operation(&self) -> Result<(), ResourceError> {
        match self.attempt_access() {
            Ok(result) => Ok(result),
            Err(ResourceError::Timeout) => {
                // Retry with longer timeout
                self.attempt_access_with_timeout(extended_timeout)
            }
            Err(other) => Err(other),
        }
    }
}
```

### **Configuration Best Practices**

```toml
# ✅ Good: Clear naming and organization
[tasks.primary_i2c_server]
name = "drv-stm32xx-i2c-server"
features = ["h753", "i2c-mux-support"]
uses = ["i2c1", "i2c4", "i2c5"]      # Group related resources
priority = 3

[tasks.critical_sensor_controller]
name = "critical-sensor-controller"
features = ["h753", "high-precision-adc"]
uses = ["i2c2", "adc1", "timer1"]    # Dedicated critical resources
priority = 1                         # Higher priority for critical tasks

# ❌ Bad: Unclear naming and priorities
[tasks.task1]
name = "some-task"
uses = ["i2c1", "spi1", "gpio_bank_a", "timer2", "dma1_ch1"]  # Too many resources
priority = 2

[tasks.task2]
name = "other-task"
uses = ["i2c2", "spi2", "gpio_bank_b", "timer3", "dma1_ch2"]  # Resource sprawl
priority = 2
```

---

## Conclusion

Hubris's resource management models provide a spectrum of approaches from direct hardware ownership to server-based sharing, each with distinct advantages and trade-offs:

### **Key Takeaways**

1. **Direct Ownership** provides maximum performance and isolation but requires more hardware resources and can lead to code duplication.

2. **Server-Based Sharing** enables resource efficiency and code reuse but introduces IPC overhead and potential failure propagation.

3. **Hybrid Models** offer the best of both approaches by applying the right model to each resource based on its specific requirements.

4. **Compile-Time Allocation** eliminates entire classes of runtime errors and provides strong safety guarantees regardless of the chosen model.

5. **Configuration-Driven Design** makes resource allocation decisions explicit and auditable.

### **Selection Criteria Summary**

| Requirement | Recommended Model |
|-------------|-------------------|
| **Real-time performance** | Direct Ownership |
| **Resource efficiency** | Server-Based |
| **Maximum isolation** | Direct Ownership |
| **Code reuse** | Server-Based |
| **Mixed requirements** | Hybrid |
| **Simple protocols** | Direct Ownership |
| **Complex protocols** | Server-Based |

### **Future Considerations**

As systems evolve, Hubris's resource management models can be adapted:

- **Migration strategies** from server-based to direct ownership as hardware becomes available
- **Dynamic reconfiguration** for development vs. production builds
- **Security model evolution** to address new threat models
- **Performance optimization** through profile-guided resource allocation

The flexibility of Hubris's approach allows systems to evolve their resource management strategy while maintaining the core safety and performance guarantees that make Hubris suitable for critical embedded applications.

---

**This document provides the foundation for understanding Hubris resource management. For specific implementation guidance, refer to the Hubris documentation and examine real-world application configurations in the Hubris repository.**
