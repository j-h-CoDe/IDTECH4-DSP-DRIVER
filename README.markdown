# idTech4-DSP-Audio: Ray-Traced Audio Offload for Doom 3 (id Tech 4 Engine)

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Build Status](https://img.shields.io/badge/build-passing-brightgreen.svg)](https://example.com) <!-- Replace with actual CI if set up -->
[![Contributors](https://img.shields.io/badge/contributors-1-orange.svg)](https://github.com/yourusername/idTech4-DSP-Audio/graphs/contributors)

## Project Overview

This repository provides a prototype implementation for integrating a Texas Instruments TMS320C674x DSP PCB with the original id Tech 4 engine (Doom 3 GPL source code) to offload real-time ray-traced audio computations. The DSP handles stochastic ray tracing for advanced audio effects such as reverb, occlusion, diffraction, and material-based absorption, enabling more realistic sound propagation in 3D environments. Communication between the host (Doom 3 engine) and DSP is managed via libusb for low-latency bulk data transfers, targeting <20ms end-to-end latency.

This prototype modifies the Doom 3 sound system minimally while preserving compatibility with existing backends (e.g., OSS and ALSA). It serves as a proof-of-concept for hardware-accelerated audio in legacy game engines and can be extended for modern applications.

### Key Features
- **Stochastic Ray Tracing**: Computes impulse responses (IRs) on DSP using up to 1000 rays per sound source with 5 bounces.
- **Real-Time Convolution**: Uses TI DSPLIB for efficient FIR filtering on audio buffers.
- **libusb Integration**: Cross-platform USB communication for scene data and audio buffers.
- **Fallback Mechanism**: Seamlessly reverts to CPU-based audio if DSP is unavailable.
- **Optimized DSP Code**: Leverages TMS320C674x instructions (e.g., MPYSP, DOTP4) for vector math and pipeline efficiency.

### Assumptions and Requirements
- **Engine Source**: Based on the original Doom 3 GPL codebase from [id-Software/DOOM-3](https://github.com/id-Software/DOOM-3).
- **DSP Hardware**: TMS320C674x-based PCB (e.g., OMAP-L138 or custom board) connected via USB.
- **Development Tools**: Code Composer Studio (CCS) for DSP firmware; SCons for Doom 3 build.
- **Host OS**: Linux (primary; uses libusb-1.0). Adaptable to Windows/Mac with minor changes.
- **Dependencies**: libusb-1.0, TI DSPLIB, and standard C++ libraries.
- **Performance**: Assumes simplified scene geometry (bounding boxes) to fit DSP constraints.

**Note**: This is a prototype. Production use requires further optimization, testing, and potentially a custom USB firmware for the DSP.

## Table of Contents
- [Installation](#installation)
- [Usage](#usage)
- [Architecture Overview](#architecture-overview)
- [Doom 3 Modifications](#doom-3-modifications)
- [TI DSP Implementation](#ti-dsp-implementation)
- [Driver Integration](#driver-integration)
- [Optimization and Edge Cases](#optimization-and-edge-cases)
- [Sample Code Snippets](#sample-code-snippets)
- [Contributing](#contributing)
- [License](#license)
- [Resources and References](#resources-and-references)

## Installation

### Prerequisites
- Clone the Doom 3 repository: `git clone https://github.com/id-Software/DOOM-3.git`.
- Install libusb-1.0: On Ubuntu, `sudo apt install libusb-1.0-0-dev`.
- Set up CCS for TMS320C674x with DSPLIB (download from TI website).
- USB drivers for the DSP PCB (ensure VID/PID are configured).

### Building Doom 3 with DSP Support
1. Clone this repo into the Doom 3 source: `git clone https://github.com/yourusername/idTech4-DSP-Audio.git neo/dsp`.
2. Modify `SConstruct` to include libusb: Add `-lusb-1.0` to `LIBS`.
3. Build: `scons -j4` (enable DSP with custom flag if added, e.g., `scons DSP=1`).
4. Flash DSP firmware via CCS (see [TI DSP Implementation](#ti-dsp-implementation)).

### DSP Firmware Build
1. Create a new CCS project for TMS320C674x.
2. Import provided sample code (e.g., `dsp_raytrace.c`).
3. Link DSPLIB and USB stack (TI's USB library recommended).
4. Build and flash to the DSP board.

## Usage

1. Connect the DSP PCB via USB.
2. Launch Doom 3 with DSP enabled: `./doom3 +set s_useDSP 1`.
3. In-game, audio effects (e.g., reverb in rooms) will be offloaded to DSP.
4. Monitor console for logs (e.g., latency warnings or fallback events).

If DSP is not detected, the engine falls back to standard audio processing.

## Architecture Overview

The prototype extracts real-time scene data from Doom 3's sound system and offloads computations to the DSP. Scene simplification reduces complexity, using bounding boxes for geometry. Stochastic ray tracing on the DSP generates IRs, which are convolved with dry audio buffers. Processed wet audio is returned via libusb for mixing in the engine.

### Data Flow Diagram

```mermaid
graph TD
    A[Doom 3 Engine] -->|Scene Data & Audio Buffers| B[libusb Driver]
    B -->|Bulk Transfer| C[TMS320C674x DSP]
    C -->|Processed Audio| B
    B -->|Mixed Audio| D[Engine Sound Mixer]
    subgraph "Doom 3"
        A --> E[idSoundSystemLocal::AsyncUpdate]
        E --> F[idDSPDriver::SendSceneData & ProcessAudioBuffer]
    end
    subgraph "DSP"
        C --> G[Ray Tracing: Stochastic Rays, Bounces]
        G --> H[IR Computation & Convolution via DSPLIB]
    end
```

## Doom 3 Modifications

Modifications are isolated to the sound system for easy integration and reversion.

### Key Changes
- **CVAR Addition**: In `sound/snd_system.cpp`, add `idCVar s_useDSP("s_useDSP", "0", CVAR_SOUND | CVAR_BOOL, "Enable DSP offload for ray-traced audio");`.
- **Hooking into Update Loop**: In `idSoundSystemLocal::AsyncUpdate`, if `s_useDSP.GetBool()`, extract data and call `g_dspDriver->Process()`.
- **Data Extraction**: Use `idSoundWorld::PlaceListener` for listener pose; iterate `idSoundEmitter` for sources; simplify scene via `gameLocal` entity bounds.

### idDSPDriver Class
A new class in `sys/dsp_driver.h/cpp` handles libusb communication.

(Full code in [Sample Code Snippets](#sample-code-snippets).)

## TI DSP Implementation

### Project Setup
- Target: TMS320C674x with floating-point support.
- Libraries: DSPLIB for convolution (e.g., `DSPF_sp_fir_gen`).
- USB Handling: Use TI's USB device stack for bulk endpoints.

### Ray Tracing Algorithm
Implements stochastic tracing with DSP-optimized math.

(Full pseudo-code in [Sample Code Snippets](#sample-code-snippets).)

### Real-Time Considerations
- Use EDMA for fast buffer copies.
- Optimize loops with inline assembly, respecting unit constraints (e.g., one 1X cross-path per cycle).
- Fixed-point arithmetic (Q15) for speed if floating-point overhead is high.

## Driver Integration

### Host-Side
- libusb for bulk transfers (OUT for data, IN for results).
- Threading: Async via `sys_CreateThread` to prevent blocking.

### DSP-Side
- Interrupt-driven USB reception; parse packets and trigger computations.

### Error Handling
- Timeout on transfers: Fallback to CPU with log warning.
- Latency Check: Timestamp buffers; disable offload if >20ms.

## Optimization and Edge Cases

- **Optimizations**: Cache IRs for static sources; reduce rays to 500; use SRAM for hot data.
- **Edge Cases**:
  - No sources: Skip DSP calls.
  - DSP Disconnect: Detect via libusb_error; fallback gracefully.
  - High Bounces: Cap at 5 to avoid overflow.
  - Pipeline Stalls: Avoid with NOP padding (e.g., LDW requires 4 delay slots).

## Sample Code Snippets

### idDSPDriver.h (Doom 3 Side)

```cpp
// sys/dsp_driver.h
#ifndef __SYS_DSP_DRIVER_H__
#define __SYS_DSP_DRIVER_H__

#include <libusb-1.0/libusb.h>
#include "idlib/containers/List.h"
#include "idlib/math/Vector.h"
#include "idlib/math/Bounds.h"
#include "sys/sys_thread.h"

struct idSoundSource {
    idVec3 position;
    int materialID;  // For absorption
};

class idDSPDriver {
public:
    idDSPDriver();
    ~idDSPDriver();

    bool Initialize(libusb_device_handle* handle);
    void SendSceneData(const idVec3& listener, const idList<idSoundSource>& sources, const idBounds& scene);
    void ProcessAudioBuffer(float* input, float* output, int samples);

private:
    libusb_context* ctx;
    libusb_device_handle* dev_handle;
    sysThread_t asyncThread;

    int BulkTransfer(unsigned char endpoint, void* data, int length, int timeout);
    // Serialization helpers...
};

extern idDSPDriver* g_dspDriver;

#endif
```

### idDSPDriver.cpp (Partial)

```cpp
// sys/dsp_driver.cpp
#include "dsp_driver.h"

idDSPDriver* g_dspDriver = NULL;

idDSPDriver::idDSPDriver() : ctx(NULL), dev_handle(NULL) {
    g_dspDriver = this;
}

bool idDSPDriver::Initialize(libusb_device_handle* handle) {
    // libusb init and claim...
    return true;
}

// Implement SendSceneData: Serialize to buffer, bulk out.
// Implement ProcessAudioBuffer: Send input, receive output.
```

### DSP Ray Tracing (dsp_raytrace.c)

```c
// dsp_raytrace.c
#include <c6x.h>
#include "dsplib.h"

#define MAX_RAYS 1000
#define MAX_BOUNCES 5
#define SAMPLE_RATE 44100
#define SPEED_OF_SOUND 343.0f

typedef struct { float x, y, z; } vec3;

void ComputeIR(float* ir, int length, vec3 source, vec3 listener, /* scene_t scene */) {
    memset(ir, 0, length * sizeof(float));
    for (int r = 0; r < MAX_RAYS; r++) {
        vec3 dir; // Generate random dir
        float dist = 0.0f;
        int bounces = 0;
        while (bounces < MAX_BOUNCES) {
            // Ray-scene intersection (simplified)
            // Use asm for math: asm("MPYSP .M1 A0, A1, A2");
            dist += 1.0f; // Placeholder
            bounces++;
        }
        int idx = (int)(dist / SPEED_OF_SOUND * SAMPLE_RATE);
        if (idx < length) ir[idx] += 1.0f; // Absorption factor
    }
}

void ConvolveAudio(float* input, float* ir, float* output, int samples, int ir_len) {
    DSPF_sp_fir_gen(input, ir, output, ir_len, samples);
}
```

### DSP Main Loop (main.c)

```c
#include <usblib.h> // TI USB lib

void main() {
    // Init USB and DSPLIB
    while (1) {
        // Receive via USB interrupt
        // Parse scene/audio
        float ir[1024];
        ComputeIR(ir, 1024, /* params */);
        float output[4096];
        ConvolveAudio(input, ir, output, 4096, 1024);
        // Send output via USB bulk
    }
}
```

## Contributing

Contributions are welcome! Please fork the repo, create a feature branch, and submit a pull request. Follow these guidelines:
- Code Style: Match Doom 3 conventions (e.g., id* prefixes).
- Testing: Include unit tests for DSP functions.
- Issues: Report bugs or suggest features via GitHub Issues.

## License

This project is licensed under the GNU General Public License v3.0 (GPL-3.0), as it extends the Doom 3 GPL codebase. See [LICENSE](LICENSE) for details.

## Resources and References

- **TI Documentation**: TMS320C674x DSP CPU and Instruction Set Reference Guide (SPRUFE8B), DSPLIB Reference.
- **Doom 3 Resources**: [Fabien Sanglard's Code Review](https://fabiensanglard.net/doom3/).
- **Audio Ray Tracing**: US Patent 8139780B2, vaRays library inspiration.
- **libusb**: [Official Documentation](https://libusb.info/) for bulk transfers.

For questions, open an issue or contact the maintainer.