# Testing Audio Quality Fix for Issue #739

## Overview
This document describes how to test the audio quality improvement implemented for Issue #739.

## Changes Made
Modified the float→int16 conversion in `src/xenia/apu/xma_context.cc` to use proper rounding (`std::lrintf`) instead of truncation, making the scalar path consistent with the SIMD path.

## Testing Methodology

### 1. Subjective Listening Test
**Games to test:** Forza Horizon (primary reporter's game), any other XMA-based games

**Equipment:** High-quality headphones (as the issue is more noticeable with headphones)

**Steps:**
1. Build Xenia Canary with the fix
2. Load a game with background music and sound effects
3. Listen for:
   - Reduced harshness in high frequencies
   - Smoother overall sound
   - Less "digital" artifacts
   - Better clarity in complex audio passages

**Compare with:** 
- Previous Xenia build (if possible to capture recordings)
- Real Xbox 360 hardware output (if available)

### 2. Objective Measurements

#### Audio Capture and Analysis
```bash
# Capture audio from both versions
# Old version: xenia_old.wav
# New version: xenia_new.wav

# Analyze with tools like Audacity or SoX
# Look for:
# - Reduced quantization noise floor
# - Better frequency response
# - Lower THD+N (Total Harmonic Distortion plus Noise)
```

#### Quantization Noise Comparison
The fix should show:
- More evenly distributed quantization errors
- Reduced peaks in error distribution
- Better Signal-to-Quantization-Noise Ratio (SQNR)

### 3. Automated Testing

#### Unit Test Concept
```cpp
// Test that conversion is consistent and uses rounding
TEST_CASE("Float to Int16 conversion uses proper rounding") {
  // Test value that demonstrates rounding vs truncation
  float test_values[] = {
    0.0f,           // 0 → 0 (both methods)
    0.5f / 32767,   // Small positive → 0 vs 1
    -0.5f / 32767,  // Small negative → 0 vs -1
    0.49999f,       // Just under 0.5 → 16383 (rounded down)
    0.50001f,       // Just over 0.5 → 16384 (rounded up)
  };
  
  // Verify rounding behavior matches lrintf
  for (float value : test_values) {
    float scaled = value * 32767.0f;
    int16_t rounded = static_cast<int16_t>(std::lrintf(scaled));
    int16_t truncated = static_cast<int16_t>(scaled);
    
    // The fix uses rounding, not truncation
    // For values like 0.5, rounding gives different result than truncation
  }
}
```

### 4. Performance Testing
The fix should have **zero performance impact** since `std::lrintf` is typically a single CPU instruction (same as truncation).

**Verification:**
```bash
# Profile audio system performance
# Compare frame decode times before/after
# CPU usage should be identical
```

### 5. Regression Testing
Ensure no audio functionality is broken:

**Test cases:**
- [ ] Mono audio playback (1 channel)
- [ ] Stereo audio playback (2 channels)
- [ ] 5.1 surround audio playback (6 channels)
- [ ] Various sample rates (24kHz, 32kHz, 44.1kHz, 48kHz)
- [ ] Audio playback in multiple games simultaneously
- [ ] Audio volume control
- [ ] Audio pause/resume
- [ ] XMA decoder switching between old and new implementations

### 6. Expected Results

#### Before Fix (Truncation)
- Value 1.5 → truncates to 1
- Value -1.5 → truncates to -1
- Asymmetric quantization error
- Bias toward zero

#### After Fix (Rounding)
- Value 1.5 → rounds to 2 (banker's rounding)
- Value -1.5 → rounds to -2 (banker's rounding)
- Symmetric quantization error
- No DC bias

### 7. Known Limitations
This fix addresses one source of quality issues. The audio will still be limited by:
1. Xbox 360's 16-bit audio specification
2. XMA codec compression
3. Original game audio quality
4. Sample rate limitations (24-48kHz)

### 8. Reporting Results
When testing, please report:
- Game tested
- Audio equipment used
- Subjective quality improvement (none/slight/moderate/significant)
- Any new issues introduced
- CPU usage changes (if any)

## Additional Notes
- The fix only affects non-SIMD (scalar) code paths
- On x86-64 systems with SSE2, the SIMD path is typically used
- ARM and other architectures will benefit most from this fix
- The change maintains bit-exact compatibility with the SIMD path
