# STM32F767ZI Basic Subtractive Synthesizer - Development Specification

## 1. Project Overview

### 1.1 Project Scope
This document specifies the development of a basic subtractive synthesizer using the STM32F767ZI Nucleo-144 development board with a PCM5102 I2S DAC breakout board. The synthesizer will implement fundamental synthesis components including oscillators, filters, envelopes, and real-time parameter control via UART interface and physical potentiometers.

### 1.2 Target Hardware
- **Primary MCU**: STM32F767ZI (Cortex-M7 @216MHz, 512kB SRAM, 2MB Flash)
- **Audio DAC**: PCM5102 I2S Breakout Board (24-bit, up to 384kHz sample rate)
- **Development Board**: Nucleo-144 F767ZI
- **Interface**: USB for programming/MIDI, UART for parameter control, ADC for potentiometers

### 1.3 Synthesis Architecture
The synthesizer will implement a classic subtractive synthesis signal chain:
- **Oscillator** → **Filter** → **Envelope** → **Amplifier** → **Output**

## 2. System Architecture

### 2.1 Signal Flow Architecture
The system follows a modular design with clear separation between input processing, synthesis engine, and audio output:

1. **Input Processing**
   - MIDI input via USB/UART
   - Control parameter input via ADC (potentiometers)
   - Button input via GPIO

2. **Synthesis Engine**
   - Oscillator with multiple waveforms (sine, sawtooth, square, triangle)
   - Low-pass filter with cutoff frequency and resonance control
   - ADSR envelope generator
   - Voice management (monophonic initially)

3. **Audio Output**
   - I2S interface to PCM5102 DAC
   - Stereo line-out capability
   - Sample rate: 48kHz, 16-bit initially (expandable to 24-bit)

### 2.2 Software Architecture
The software follows a layered architecture pattern with clear abstraction levels:

**Application Layer**
- Main synthesis loop
- Parameter management
- User interface handling

**Synthesis Engine Layer**
- Oscillator modules
- Filter modules
- Envelope generators
- Voice management

**Hardware Abstraction Layer**
- I2S audio driver
- MIDI interface
- ADC interface for controls
- GPIO interface
- DMA buffer management

**HAL/Low-Level Layer**
- STM32 HAL libraries
- Direct hardware access

## 3. Hardware Connections

### 3.1 PCM5102 I2S DAC Connection

| PCM5102 Pin | STM32F767ZI Pin | Function | Notes |
|-------------|----------------|----------|-------|
| VCC/VIN | 3.3V | Power supply | 3.3V from Nucleo board |
| GND | GND | Ground | Common ground |
| BCK | PC10 (I2S3_CK) | Bit Clock | I2S serial clock |
| DIN | PC12 (I2S3_SD) | Data Input | I2S serial data |
| LCK/WSEL | PA15 (I2S3_WS) | Word Select | Left/Right channel select |
| SCK | GND | System Clock | Not connected (internal PLL) |
| FMT | GND | Format Select | I2S format (not left-justified) |
| XMT | 3.3V | Soft Mute | Unmuted operation |

### 3.2 Control Interface Connections

| Component | STM32F767ZI Pin | Function | Notes |
|-----------|----------------|----------|-------|
| Portamento Pot | PA0 (ADC1_CH0) | Analog input | 10kΩ linear potentiometer |
| Button 1 | PC13 | Digital input | Note trigger/menu |
| USB UART | PA9/PA10 | USART1 | Parameter control interface |

### 3.3 Power and Debug
- USB power via ST-LINK
- SWD debugging interface available
- Serial debug output via UART

## 4. Software Design Patterns

### 4.1 Real-Time Audio Processing Patterns

**Double Buffering Pattern**
- Primary pattern for DMA-based I2S audio output
- Ensures continuous audio stream without interruption
- Two audio buffers alternate between filling and playback

**Circular Buffer Pattern**
- Used for MIDI input buffering
- Handles variable-rate MIDI data efficiently
- Prevents data loss during high MIDI traffic

**State Machine Pattern**
- Voice state management (NOTE_OFF, ATTACK, DECAY, SUSTAIN, RELEASE)
- System mode management (PLAY, MENU, PARAM_EDIT)
- Envelope generator state control

### 4.2 Memory Management Patterns

**Static Allocation Pattern**
- All audio buffers pre-allocated at compile time
- No dynamic memory allocation in real-time context
- Predictable memory usage and real-time performance

**Fixed-Size Pool Pattern**
- Voice allocation from fixed pool
- Efficient voice stealing algorithm for polyphony expansion

## 5. Synthesis Engine Specification

### 5.1 Oscillator Module

**Waveform Generation**
- Sine wave: Using lookup table or ARM CMSIS DSP sine function
- Sawtooth wave: Linear ramp with reset
- Square wave: Bipolar square wave with 50% duty cycle
- Triangle wave: Integrated square wave

**Parameters**
- Frequency: 20Hz to 20kHz (MIDI note-based)
- Amplitude: 0.0 to 1.0 (float)
- Waveform selection: 4 basic waveforms

**Technical Implementation**
- Phase accumulator approach for frequency generation
- 32-bit phase accumulator for frequency precision
- Lookup table size: 1024 samples for sine wave

### 5.2 Filter Module

**Filter Type**
- Single-pole IIR low-pass filter (initially)
- Expandable to multi-pole designs

**Parameters**
- Cutoff Frequency: 20Hz to 20kHz
- Resonance: 0.0 to 0.95 (Q factor control)

**Technical Implementation**
```c
// Simple IIR low-pass filter equation:
// y[n] = (1-α) * y[n-1] + α * x[n]
// where α = cutoff frequency coefficient
```

### 5.3 Envelope Module

**ADSR Envelope**
- Attack: 0ms to 5000ms
- Decay: 0ms to 5000ms  
- Sustain: 0.0 to 1.0 level
- Release: 0ms to 5000ms

**Technical Implementation**
- State machine with 4 states (ATTACK, DECAY, SUSTAIN, RELEASE)
- Linear or exponential curve options
- Sample-accurate timing

### 5.4 Audio Buffer Management

**Buffer Configuration**
- Buffer size: 128 samples per channel (expandable to 64 or 256)
- Dual buffers for ping-pong operation
- 16-bit signed PCM format initially

**DMA Configuration**
- Circular mode DMA for continuous audio output
- Half-complete and full-complete interrupts for buffer swapping
- I2S peripheral in master transmit mode

## 6. Project Structure

### 6.1 Recommended Folder Structure

```
stm32-synthesizer/
├── Core/
│   ├── Inc/                    # Generated headers
│   └── Src/                    # Generated source files
│       └── main.c              # Main application entry
├── Synth/                      # Synthesizer modules
│   ├── Inc/
│   │   ├── synth_oscillator.h
│   │   ├── synth_filter.h
│   │   ├── synth_envelope.h
│   │   └── synth_voice.h
│   └── Src/
│       ├── synth_oscillator.c
│       ├── synth_filter.c
│       ├── synth_envelope.c
│       └── synth_voice.c
├── Hardware/                   # Hardware abstraction
│   ├── Inc/
│   │   ├── audio_driver.h
│   │   ├── midi_interface.h
│   │   └── control_interface.h
│   └── Src/
│       ├── audio_driver.c
│       ├── midi_interface.c
│       └── control_interface.c
├── Utils/                      # Utility functions
│   ├── Inc/
│   │   ├── circular_buffer.h
│   │   └── math_utils.h
│   └── Src/
│       ├── circular_buffer.c
│       └── math_utils.c
└── Drivers/                    # STM32 HAL drivers
```

### 6.2 Main Application Structure

```c
// main.c structure
int main(void) {
    // System initialization
    HAL_Init();
    SystemClock_Config();
    
    // Peripheral initialization
    MX_GPIO_Init();
    MX_DMA_Init();
    MX_I2S3_Init();
    MX_ADC1_Init();
    MX_UART4_Init();
    
    // Synthesizer initialization
    Synth_Init();
    Audio_Driver_Init();
    MIDI_Interface_Init();
    
    // Start audio processing
    Audio_Driver_Start();
    
    // Main application loop
    while (1) {
        // Non-real-time tasks
        MIDI_Process_Messages();
        Control_Update_Parameters();
        UI_Update();
        
        // Brief sleep/idle
        HAL_Delay(1);
    }
}

// Audio callback (called from DMA interrupt)
void HAL_I2S_TxHalfCpltCallback(I2S_HandleTypeDef *hi2s) {
    Synth_Process_Audio_Block(audio_buffer_1, BUFFER_SIZE);
}

void HAL_I2S_TxCpltCallback(I2S_HandleTypeDef *hi2s) {
    Synth_Process_Audio_Block(audio_buffer_2, BUFFER_SIZE);
}
```

## 7. Development Phases

### 7.1 Phase 1: Basic Framework (Week 1-2)
- STM32CubeIDE project setup
- PCM5102 I2S connection and basic audio output
- Simple sine wave generation test
- Basic project structure implementation

### 7.2 Phase 2: Core Synthesis (Week 3-4)
- Oscillator module with multiple waveforms
- Basic low-pass filter implementation
- ADSR envelope generator
- Monophonic voice management

### 7.3 Phase 3: Control Interface (Week 5-6)
- UART parameter control implementation
- ADC-based potentiometer reading
- MIDI note input processing
- Basic UI/menu system

### 7.4 Phase 4: Enhancement & Polish (Week 7-8)
- Audio quality optimization
- Performance tuning
- Additional synthesis features
- Documentation and testing

## 8. Technical Specifications

### 8.1 Audio Specifications
- **Sample Rate**: 48kHz (primary), expandable to 96kHz
- **Bit Depth**: 16-bit (primary), expandable to 24-bit
- **Channels**: Stereo output (mono synthesis duplicated)
- **Latency**: <10ms (target <5ms)
- **Frequency Response**: 20Hz - 20kHz
- **THD+N**: <0.1% (target performance)

### 8.2 Performance Requirements
- **CPU Usage**: <70% average, <90% peak
- **Memory Usage**: <256kB SRAM, <1MB Flash
- **Real-time Constraints**: Audio processing must complete within buffer period
- **Polyphony**: Monophonic initially, designed for easy expansion

### 8.3 Control Interface Specifications
- **MIDI Input**: USB MIDI class compliant or UART at 31.25kbaud
- **Parameter Control**: UART at 115200 baud
- **Analog Controls**: 12-bit ADC resolution
- **Response Time**: <50ms for parameter changes

## 9. Build and Development Environment

### 9.1 Required Tools
- **IDE**: STM32CubeIDE (recommended) or CLion with STM32 plugin
- **Toolchain**: ARM GCC (included with STM32CubeIDE)
- **Debugger**: ST-LINK (integrated on Nucleo board)
- **Version Control**: Git recommended

### 9.2 Key Dependencies
- STM32Cube HAL F7 firmware package
- ARM CMSIS DSP library (optional, for optimized math functions)
- Custom synthesizer modules (developed in-house)

### 9.3 Configuration Management
- STM32CubeMX for peripheral configuration (.ioc file)
- Makefile or CMake build system
- Separate configuration header for synthesis parameters

## 10. Testing and Validation

### 10.1 Unit Testing
- Individual module testing (oscillator, filter, envelope)
- Audio buffer management testing
- Control interface validation

### 10.2 Integration Testing
- Complete synthesis chain testing
- Real-time performance validation
- Audio quality measurements

### 10.3 System Testing
- Extended operation testing
- Stress testing with continuous operation
- User interface validation

## 11. Future Enhancement Opportunities

### 11.1 Synthesis Features
- Multiple oscillators per voice
- Additional filter types (high-pass, band-pass, notch)
- LFOs for modulation
- Effects processing (reverb, delay, chorus)

### 11.2 Polyphony
- Multi-voice architecture
- Voice stealing algorithms
- Polyphonic mode implementation

### 11.3 Advanced Features
- Preset management and storage
- Real-time parameter automation
- External audio input processing
- USB audio interface implementation

---

*This specification provides a comprehensive framework for developing a basic subtractive synthesizer on the STM32F767ZI platform. The modular design allows for incremental development and future expansion while maintaining real-time audio performance.*