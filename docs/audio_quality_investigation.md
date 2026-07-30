# Audio Quality Investigation - Issue #739

## Problem Statement
Users are reporting that sound quality in Xenia Canary appears "low/compressed", particularly noticeable when using headphones in games like Forza Horizon.

## Investigation Findings

### 1. Audio Pipeline Overview

The audio subsystem in Xenia uses the following pipeline:

```
Game Audio Data (XMA compressed) 
    → XMA Decoder (FFmpeg)
    → Float Planar (FLTP) format
    → Conversion to int16 samples  
    → Ring buffer in guest memory
    → Game reads and submits to XAudio
    → XAudio2 driver (32-bit float output)
```

### 2. Potential Causes of Quality Loss

#### 2.1 Float to Int16 Conversion
**Location:** `src/xenia/apu/xma_context.cc:57-132`

The `ConvertFrame` function converts FFmpeg's float planar output to 16-bit integer samples. This conversion includes:
- Scaling from [-1.0, 1.0] float range to 16-bit integer range
- Clamping values that exceed [-1.0, 1.0]
- Saturation during conversion

**Impact:** This conversion reduces precision from 32-bit float (23-bit mantissa) to 16-bit integer, which is inherent to Xbox 360 audio architecture.

**Code snippet:**
```cpp
constexpr float scale = (1 << 15) - 1;  // 32767
float scaled_sample = xe::clamp_float(in[i], -1.0f, 1.0f) * scale;
auto sample = static_cast<int16_t>(scaled_sample);
```

#### 2.2 FFmpeg Decoder Configuration
**Location:** 
- `src/xenia/apu/xma_context_old.cc:939`
- `src/xenia/apu/xma_context_new.cc:654`

The XMA decoder is opened with minimal configuration:
```cpp
avcodec_open2(av_context_, av_codec_, NULL)
```

The NULL parameter means no codec options are being passed. Potential improvements:
- No explicit quality settings
- Default skip/discard settings
- No request for specific sample formats

#### 2.3 Sample Rate Handling
**Location:** `src/xenia/apu/xma_context_new.h:44`

Supported sample rates:
```cpp
static constexpr int kIdToSampleRate[4] = {24000, 32000, 44100, 48000};
```

These are correct for Xbox 360, but some games may use lower sample rates (24kHz, 32kHz) which inherently have lower quality than 44.1kHz or 48kHz.

#### 2.4 Audio Buffer Management
**Location:** `src/xenia/apu/audio_system.cc:38-43`

```cpp
DEFINE_uint32(apu_max_queued_frames, 8,
              "Allows changing max buffered audio frames to reduce audio "
              "delay. Lowering this value might cause performance issues. "
              "Value range: [4-64]",
              "APU");
```

Buffer size affects latency but could potentially cause quality issues if buffers are under-run or if timing issues cause audio artifacts.

### 3. Comparison with Real Hardware

The Xbox 360 audio system:
- Uses XMA (Xbox Media Audio) compression
- Outputs to 16-bit samples natively
- Supports sample rates from 24kHz to 48kHz
- Has hardware decompression

Xenia's emulation is accurate to the spec, but the perceived quality issue might stem from:
1. **Modern expectations:** Users comparing to modern 24-bit/96kHz+ audio
2. **Headphone amplification:** Direct monitoring may make compression artifacts more noticeable
3. **Missing post-processing:** Real Xbox 360 hardware may have had analog filtering or DAC characteristics that affected the sound

### 4. Recommended Investigation Steps

1. **Capture actual game audio:** Compare audio output from real Xbox 360 vs Xenia
2. **Analyze frequency response:** Check if high frequencies are being attenuated
3. **Check for resampling:** Verify if any unintended resampling is occurring
4. **Monitor for buffer issues:** Check for buffer underruns or timing problems
5. **Test with different games:** Determine if issue is game-specific or systemic

### 5. Potential Solutions (Ordered by Impact)

#### 5.1 Low Impact / Safe Changes
- ✅ **IMPLEMENTED:** Fix scalar conversion path to use proper rounding instead of truncation
- Add codec options to `avcodec_open2()` for quality optimization
- Document expected audio quality limitations
- Add audio quality information to FAQ

#### 5.2 Medium Impact Changes  
- Investigate if float samples can be preserved longer in the pipeline
- Add option for different dithering algorithms during float→int16 conversion
- Implement proper error diffusion dithering to reduce quantization noise

#### 5.3 High Impact / Risky Changes
- Attempt to bypass int16 conversion (may break compatibility)
- Implement custom XMA decoder (significant effort)
- Add upsampling or quality enhancement filters (may cause timing issues)

### 6. Implemented Improvements

#### Float→Int16 Conversion Rounding Fix
**File:** `src/xenia/apu/xma_context.cc`

**Problem:** The scalar (non-SIMD) conversion path was using truncation (`static_cast<int16_t>`) while the SIMD path uses proper rounding (`_mm_cvtps_epi32`). This inconsistency could cause subtle quality differences and truncation can introduce more distortion than rounding.

**Solution:** Changed the scalar path to use `std::lrintf()` which provides proper rounding (banker's rounding) consistent with the SIMD path. This ensures:
- Consistent behavior across SIMD and scalar code paths
- Reduced quantization noise compared to truncation
- Better distribution of rounding errors

**Code change:**
```cpp
// Before:
auto sample = static_cast<int16_t>(scaled_sample);

// After:  
auto sample = static_cast<int16_t>(std::lrintf(scaled_sample));
```

This is a safe, low-impact change that improves audio quality without affecting compatibility or performance.

### 7. Code References

Key files for audio quality:
- `src/xenia/apu/xma_context.cc` - Sample format conversion
- `src/xenia/apu/xma_context_old.cc` - Old XMA decoder implementation
- `src/xenia/apu/xma_context_new.cc` - New XMA decoder implementation  
- `src/xenia/apu/xaudio2/xaudio2_audio_driver.cc` - Windows audio output
- `src/xenia/apu/audio_system.cc` - Audio subsystem management

### 8. Conclusion

The "compressed" audio quality is likely a combination of:
1. Inherent limitations of 16-bit audio (Xbox 360 spec)
2. Possible lack of codec optimization in FFmpeg configuration
3. User expectations based on modern audio standards
4. Increased audibility of artifacts when using headphones

The most promising avenue for improvement without breaking compatibility would be:
1. Adding proper codec options to the FFmpeg decoder initialization
2. Implementing dithering during the float→int16 conversion
3. Ensuring sample rates are correctly detected and no unintended resampling occurs

However, it's important to note that if the audio quality matches real Xbox 360 hardware, then Xenia is working correctly, and the perceived quality issue is inherent to the original hardware limitations.
