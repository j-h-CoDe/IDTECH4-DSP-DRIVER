README: Custom DSP Driver for Ray-Traced Audio in id Tech 4 (Original Doom 3 GPL Source)
This guide provides a prototype for integrating a Texas Instruments TMS320C674x DSP PCB with the original id Tech 4 engine (Doom 3 GPL source code) to offload real-time ray-traced audio computations. The DSP handles stochastic ray tracing for reverb, occlusion, diffraction, and material-based absorption, using libusb for host-DSP communication. Latency targets <20ms. The prototype assumes a USB-connected TMS320C674x board and modifies the engine's sound system without breaking existing functionality (e.g., OSS/ALSA backends).
Assumptions:

id Tech 4 source from GitHub (id-Software/DOOM-3).
DSP development via Code Composer Studio (CCS) with DSPLIB.
Host OS: Linux (for libusb-1.0; adapt for Windows if needed, using provided sys/posix files).
Scene simplification: Use bounding boxes for geometry to reduce DSP load.
Stochastic rays: 1000 rays/source, up to 5 bounces.

Architecture Overview
The system extracts scene data (listener position, sound sources, simplified geometry) from the engine's sound system and sends it to the DSP via libusb bulk transfers. The DSP computes impulse responses (IRs) via ray tracing and convolves them with input audio buffers using DSPLIB (e.g., DSPF_sp_fir_gen for FIR filtering). Processed audio returns to the engine for mixing.
Data flow:

Engine: Extracts data in idSoundSystemLocal::AsyncUpdate (from snd_system.cpp), sends via idDSPDriver.
libusb: Bulk OUT for scene/audio data, Bulk IN for processed audio.
DSP: Receives data, traces rays (using .M unit for vector ops like DOTP4), computes IRs, convolves, sends back.

Diagram
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

Doom 3 Modifications
Hook into the sound system (sound/snd_system.cpp) to add DSP offload. Create idDSPDriver in sys/ for libusb handling. Use sys_thread for async operations to avoid blocking.
Key Changes

Modify sound/snd_system.cpp:

Add CVAR s_useDSP (bool, default false) to toggle DSP offload. Add to existing CVars like s_noSound.
In idSoundSystemLocal::AsyncUpdate or idSoundSystemLocal::AsyncMix, check s_useDSP and call idDSPDriver::Process if enabled.
Fallback to CPU if DSP fails (e.g., libusb error). Integrate with existing mixing (e.g., DoEnviroSuit for effects).


Extract Data:

In game/sound/idSoundWorld (from sound.h, idSoundWorld interface), gather:
Listener: Use PlaceListener params (idVec3 origin, idMat3 axis).
Sources: From idSoundEmitter (simplify to positions from active channels).
Scene: Access via gameLocal (game/Game_local.h) for entities/bounds; simplify meshes to idBounds.




idDSPDriver Class (new files: sys/dsp_driver.h/cpp):

Include <libusb-1.0/libusb.h>.
Methods:
InitializeDSP(libusb_device_handle* handle): Init libusb context, claim interface.
SendSceneData(const idVec3& listener, const idList<idSoundSource>& sources, const idBounds& scene): Serialize and send via bulk OUT.
ProcessAudioBuffer(float* input, float* output, int samples): Send input buffer, receive processed output.


Threading: Use sys-specific threading (from sys/sys_thread.h) for async bulk transfers.



Sample Code: sys/dsp_driver.h
// sys/dsp_driver.h
#ifndef __SYS_DSP_DRIVER_H__
#define __SYS_DSP_DRIVER_H__

#include <libusb-1.0/libusb.h>
#include "idlib/containers/List.h"
#include "idlib/math/Vector.h"
#include "idlib/math/Bounds.h"
#include "sys/sys_thread.h"  // For threading

struct idSoundSource {
    idVec3 position;
    // Add other props as needed (e.g., material ID)
};

class idDSPDriver {
public:
    idDSPDriver();
    ~idDSPDriver();

    bool InitializeDSP(libusb_device_handle* handle);
    void SendSceneData(const idVec3& listener, const idList<idSoundSource>& sources, const idBounds& scene);
    void ProcessAudioBuffer(float* input, float* output, int samples);

private:
    libusb_context* ctx;
    libusb_device_handle* dev_handle;
    sysThread_t asyncThread;  // Use sys thread type

    // Helper for bulk transfer
    int BulkTransfer(unsigned char endpoint, void* data, int length, int timeout);
};

#endif

Sample Code: sys/dsp_driver.cpp (Partial)
// sys/dsp_driver.cpp
#include "dsp_driver.h"
#include "sys/sys_public.h"  // For sys-specific includes

idDSPDriver::idDSPDriver() : ctx(nullptr), dev_handle(nullptr) {}

bool idDSPDriver::InitializeDSP(libusb_device_handle* handle) {
    int r = libusb_init(&ctx);
    if (r < 0) return false;
    dev_handle = handle;  // Assume handle from USB detection
    r = libusb_claim_interface(dev_handle, 0);
    if (r < 0) return false;
    // Setup async thread using sys_CreateThread or similar
    return true;
}

int idDSPDriver::BulkTransfer(unsigned char endpoint, void* data, int length, int timeout) {
    int transferred;
    int r = libusb_bulk_transfer(dev_handle, endpoint, (unsigned char*)data, length, &transferred, timeout);
    if (r < 0 || transferred != length) {
        // Handle error, fallback to CPU
        return -1;
    }
    return transferred;
}

// Implement SendSceneData and ProcessAudioBuffer similarly, serializing data into buffers for transfer.

Integrate in sound/snd_system.cpp: Create global idDSPDriver* g_dspDriver = new idDSPDriver(); and call methods in AsyncUpdate.
TI DSP Code Implementation
Use CCS for TMS320C674x project. Include DSPLIB for convolution/FFT. Implement ray tracing in C with inline assembly for speed (e.g., MPYSP on .M unit for float multiplies). Handle libusb endpoints for data I/O.
Setup

New CCS project: Target TMS320C674x, include DSPLIB headers.
Use interrupts/DMA for audio buffers.
Communication: Custom protocol over USB bulk (e.g., header + scene data + audio).

Ray Tracing Algorithm
Stochastic ray tracing: Fire 1000 rays from source, track up to 5 bounces, compute delays/absorption based on materials.
Pseudo-code with instructions:
// dsp_raytrace.c
#include <c6x.h>  // For intrinsics
#include "dsplib.h"  // DSPF_sp_fir_gen, etc.

#define MAX_RAYS 1000
#define MAX_BOUNCES 5

typedef struct { float x, y, z; } vec3;

void ComputeIR(float* ir, int length, vec3 source, vec3 listener, scene_t scene) {
    for (int r = 0; r < MAX_RAYS; r++) {
        vec3 dir = random_dir();  // Stochastic direction
        float dist = 0.0f;
        int bounces = 0;
        while (bounces < MAX_BOUNCES) {
            // Trace ray: Use .L/.S units for ADD/SUB/MPY
            // Inline asm example:
            asm(" MPYSP .M1 A0, A1, A2");  // Float multiply on .M unit
            dist += hit_distance(scene, dir);
            if (hit_listener(dir, listener)) break;
            bounces++;
        }
        int idx = (int)(dist / SPEED_OF_SOUND * SAMPLE_RATE);  // Delay to IR index
        if (idx < length) ir[idx] += absorption_factor;  // Accumulate with ACC3 for sums
    }
}

void ConvolveAudio(float* input, float* ir, float* output, int samples) {
    // Use DSPLIB FIR: DSPF_sp_fir_gen(input, ir, output, IR_LEN, samples);
    // Optimize with compact 16-bit instr (e.g., NOP 4 as 00006000)
}

Real-Time Handling

Use EDMA for buffer transfers.
Interrupt on USB data receipt: Parse, compute, send back.
Fixed-point if needed (Q15 via INTSP).

Communication Protocol

Endpoint 0x01 OUT: [Header (4B: type, size)] + [Scene data] + [Audio buffer].
Endpoint 0x81 IN: [Processed audio].

Driver Integration
Host-Side (Doom 3)

In idDSPDriver: Use libusb-1.0 for init/transfer (see sample above).
Error Handling: Monitor latency with timestamps; fallback if >20ms or libusb_error.
Testing: Unit test with mock scene (e.g., simple room echo).

DSP-Side

Implement USB stack (use TI's USB library if available) to handle bulk endpoints.

Optimization and Edge Cases

Reduce rays to 500 for speed; cache IRs in SRAM for static scenes.
Dynamic updates: Retrace on source/listener movement > threshold.
DSP Load: Profile in CCS; avoid stalls (e.g., LDW + NOPs for delay slots). Limit 1X cross-path/cycle.
Compatibility: Preserve OSS/ALSA in sound.cpp/sound_alsa.cpp; mix DSP output into channels.
Edge Cases: No sources (silent), DSP disconnect (fallback), high bounce counts (cap at 5).

Sample Code Snippets

Full idDSPDriver: See above (expand ProcessAudioBuffer similarly).
DSP Main:

// main.c (CCS)
#include <usblib.h>  // Assume TI USB lib
void main() {
    // Init USB, DSPLIB
    while(1) {
        // Wait for USB interrupt, receive data
        ComputeIR(ir, IR_LEN, source, listener, scene);
        ConvolveAudio(input, ir, output, samples);
        // Send output via USB bulk
    }
}


Inline Asm Example for Dot Product:

DOTP4 .M1X A4, B4, A5  ; Dot product on .M unit, cross-path 1X

Build Instructions

Doom 3: Use SCons (from repo): scons with custom flag for DSP. Add libusb dependency. Compile with scons.
DSP: In CCS, import project, build, flash via JTAG/USB.
Link: Run game with DSP connected; detect via libusb_open_device_with_vid_pid (assume VID/PID known).

Resources and References

TI: SPRUFE8B (provided), DSPLIB docs.
Doom 3: Provided sound files (snd_system.cpp, sound.h); Fabien Sanglard's review (fabiensanglard.net/doom3/).
Audio: US8139780B2 patent, vaRays library.
libusb: Official docs for bulk transfers.
