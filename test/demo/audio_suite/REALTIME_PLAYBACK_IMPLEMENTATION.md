# Real-Time Playback Implementation Summary

## Overview
This document describes the implementation of real-time audio playback for automated testing, replacing the incorrect multi-pipeline approach with the correct single-pipeline method using `RealTimePlaying.cpp`.

## Problem Statement
**Issue**: PR #3 used `multiPipelineRealTimeSaveFileBuffer` for real-time playback, which:
- Returns null/empty buffers (designed for multi-pipeline offline processing)
- Button clicks had no response
- Incorrect implementation pattern

**Solution**: Use audio renderer with callbacks from `RealTimePlaying.cpp` for true real-time single-pipeline playback.

## Architecture

### Real-Time Playback Flow
```
User clicks button → executeRealTimePlayback()
    ↓
Initialize pipeline/input/effect nodes (common with offline mode)
    ↓
initAudioRenderer(sampleRate, channels, bitDepth)
    ↓
registerPlaybackFinishCallback(callback)
    ↓
setRecordFlag(true)
    ↓
startAudioRenderer() → Start pipeline → OH_AudioRenderer_Start()
    ↓
Audio Renderer Callback Loop (runs on audio thread):
    PlayAudioRendererOnWriteData()
        → OneRenDerFrame() / OneMulRenDerFrame()
        → OH_AudioSuiteEngine_RenderFrame()
        → Copy to g_playAudioData
        → If recording: copy to g_playTotalAudioData
        → Output to speaker
        → Check g_playFinishedFlag
        → If finished: Stop renderer, call callback
    ↓
TypeScript waits for playbackFinished flag (with 30s timeout)
    ↓
getRecordedAudioData() → Returns recorded audio buffer
    ↓
saveSingleFile() → Add WAV header and save
    ↓
Cleanup: stopAudioRenderer() + releaseAudioRenderer() + unregisterPlaybackFinishCallback()
```

### Standard Offline Flow (Unchanged)
```
User clicks button → executeStandardPlayback()
    ↓
Initialize pipeline/input/effect nodes
    ↓
multiPipelineSaveFileBuffer() in worker thread
    → Process all frames at once
    → Return complete buffer
    ↓
saveSingleFile() → Save with WAV header
```

## Implementation Details

### C++ NAPI Methods (AudioEdit.cpp)

#### 1. `initAudioRenderer(sampleRate, channels, bitDepth)`
- **Purpose**: Initialize audio renderer with output format
- **Returns**: 0 (SUCCESS) or 1 (FAILED)
- **Steps**:
  1. Release any existing renderer
  2. Create OH_AudioStreamBuilder
  3. Set audio parameters (sample rate, channels, format)
  4. Calculate frame size (g_playDataSize)
  5. Set callback: PlayAudioRendererOnWriteData
  6. Generate OH_AudioRenderer

#### 2. `startAudioRenderer()`
- **Purpose**: Start audio renderer and pipeline
- **Returns**: 0 (SUCCESS) or 1 (FAILED)
- **Steps**:
  1. Check audioRenderer != nullptr
  2. Call ProcessPipeline() to start audio pipeline
  3. Allocate g_playTotalAudioData if recording enabled
  4. Call OH_AudioRenderer_Start()

#### 3. `stopAudioRenderer()`
- **Purpose**: Stop audio renderer and pipeline
- **Returns**: 0 (SUCCESS) or error code
- **Steps**:
  1. Call OH_AudioRenderer_Stop()
  2. Call OH_AudioSuiteEngine_StopPipeline()

#### 4. `releaseAudioRenderer()`
- **Purpose**: Release all renderer resources
- **Calls**: ReleaseExistingResources()
  - OH_AudioRenderer_Release()
  - OH_AudioStreamBuilder_Destroy()

#### 5. `setRecordFlag(enable)`
- **Purpose**: Enable/disable recording of output
- **Sets**: g_isRecord = enable

#### 6. `getRecordedAudioData()`
- **Purpose**: Get recorded audio buffer
- **Returns**: ArrayBuffer with g_playResultTotalSize bytes, or null if no data
- **Data source**: g_playTotalAudioData

#### 7. `registerPlaybackFinishCallback(callback)`
- **Purpose**: Register JS callback for playback completion
- **Creates**: Threadsafe function (tsfnBoolean)
- **Note**: Should be called when renderer is stopped to avoid race conditions

#### 8. `unregisterPlaybackFinishCallback()`
- **Purpose**: Unregister callback and release threadsafe function
- **Releases**: tsfnBoolean

### TypeScript Methods (AutoTest.ets)

#### `executeSingleFileWithSyncInit(fileSummary, reportFlag)`
Main entry point, dispatches to real-time or offline mode based on `jsonFileInfo.realTime` flag.

#### `executeRealTimePlayback(jsonFileInfo, reportFlag)`
Handles real-time playback:
1. Validates single scene only
2. Initializes audio renderer
3. Registers completion callback
4. Enables recording
5. Starts renderer
6. Waits for completion (30s timeout)
7. Retrieves recorded data
8. Saves output file
9. Generates report
10. Cleanup in finally block

#### `executeStandardPlayback(jsonFileInfo, reportFlag)`
Handles offline processing (original logic with multi-pipeline).

## Key Differences: Real-Time vs Offline

| Aspect | Real-Time Playback | Offline Processing |
|--------|-------------------|-------------------|
| **Method** | Audio renderer callbacks | Multi-pipeline batch |
| **Execution** | Frame-by-frame on audio thread | All frames in worker thread |
| **Output** | To speaker + recording | Buffer only |
| **Speed** | Real-time (1x speed) | As fast as possible |
| **C++ API** | `OneRenDerFrame()` / `OneMulRenDerFrame()` | `multiPipelineSaveFileBuffer()` |
| **Pipeline** | Single pipeline | Can be multiple |
| **Use Case** | Live playback testing | Batch processing |

## Test JSON Format

### Real-Time Test Case
```json
{
  "summary": "Real-time playback test",
  "realTime": true,
  "scenes": [{
    "scene_name": "Real-time test",
    "description": "Test real-time audio rendering",
    "input_files": [{
      "path": "test_audio.wav",
      "sample_rate": 48000,
      "bit_depth": 16,
      "channels": 2,
      "format": "PCM",
      "effect": [
        {"type": "NR"},
        {"type": "EQ", "mode": 1}
      ]
    }],
    "mixer_info": [],
    "output_file": {
      "name_template": "output_realtime.wav",
      "format": "PCM",
      "sample_rate": 48000,
      "bit_depth": 16,
      "channels": 2
    },
    "expected_output": {
      "md5Value": ["expected_md5_hash"]
    }
  }]
}
```

**Important**: `"realTime": true` flag triggers real-time mode.

### Standard Offline Test Case
```json
{
  "summary": "Standard offline test",
  "realTime": false,
  "scenes": [
    // Multiple scenes supported
  ]
}
```

## Testing Checklist

### Prerequisites
- [ ] DevEco Studio installed with HarmonyOS SDK 6.0.0(20) or later
- [ ] HarmonyOS device or emulator available
- [ ] Test audio files prepared
- [ ] Test JSON files created with appropriate `realTime` flag

### Test Steps

#### 1. Basic Functionality
- [ ] Build project successfully in DevEco Studio
- [ ] Deploy to device/emulator
- [ ] Create test directory structure:
  - `/storage/Users/currentUser/Download/src.main.audiodemo/case_json/`
  - `/storage/Users/currentUser/Download/src.main.audiodemo/input_files/`
  - `/storage/Users/currentUser/Download/src.main.audiodemo/output_files/`

#### 2. Real-Time Playback Test
- [ ] Place real-time test JSON in `case_json/`
- [ ] Place input audio file in `input_files/`
- [ ] Open AutoTest page in app
- [ ] Select real-time test case from dropdown
- [ ] Click "Execute Selected Case" button
- [ ] **Expected**: Button becomes "Executing..." and disables
- [ ] **Expected**: Audio plays through speaker in real-time
- [ ] **Expected**: Progress dialog shows execution
- [ ] **Expected**: Playback completes within expected time
- [ ] **Expected**: Success message displayed
- [ ] Check output file in `output_files/`
- [ ] Verify output file size > 0
- [ ] Verify output file is valid WAV format
- [ ] Check test report generated

#### 3. Offline Processing Test
- [ ] Place standard test JSON (realTime: false) in `case_json/`
- [ ] Select standard test case
- [ ] Click "Execute Selected Case"
- [ ] **Expected**: Processing completes faster than real-time
- [ ] **Expected**: Output file generated correctly
- [ ] Verify existing offline tests still work

#### 4. Error Handling Test
- [ ] Test with missing input file
- [ ] **Expected**: Error message, cleanup executed
- [ ] Test with invalid audio format
- [ ] **Expected**: Error message, cleanup executed
- [ ] Test with very long audio (trigger timeout)
- [ ] **Expected**: Timeout after 30s, cleanup executed

#### 5. Button Click Test
- [ ] Verify button responds to clicks
- [ ] Verify button disables during execution
- [ ] Verify button re-enables after completion
- [ ] Verify multiple sequential executions work

### Validation Criteria

#### Success Criteria
✅ Button clicks are responsive
✅ Real-time playback executes and completes
✅ Audio data is recorded (output file size > 0)
✅ Output file is valid WAV format
✅ MD5 verification passes (if expected MD5 provided)
✅ No crashes or errors during execution
✅ Proper cleanup after completion/errors
✅ Offline mode still works correctly

#### Performance Criteria
- Real-time playback time ≈ audio duration
- Offline processing faster than real-time
- Memory usage stays within limits
- No memory leaks after multiple executions

## Troubleshooting

### Issue: Button Click Has No Response
**Possible Causes**:
- `exeOneStateEffect` is false
- `selectedFileSummary` is empty
- JavaScript error in console

**Debug Steps**:
1. Add logs in onClick handler
2. Check DevEco Studio console for errors
3. Verify test case selection

### Issue: Real-Time Playback Returns No Data
**Possible Causes**:
- g_isRecord not set
- Callback not triggering
- Pipeline not started

**Debug Steps**:
1. Check C++ logs for "SetRecordFlag"
2. Check for "PlayAudioRendererOnWriteData" calls
3. Verify ProcessPipeline() succeeded

### Issue: Timeout After 30 Seconds
**Possible Causes**:
- Pipeline initialization failed
- Audio renderer not starting
- Callback never fires
- Very long audio file

**Debug Steps**:
1. Check pipeline initialization logs
2. Check "OH_AudioRenderer_Start" result
3. Verify input file length
4. Check for g_playFinishedFlag being set

### Issue: Crash During Playback
**Possible Causes**:
- Memory allocation failure
- Race condition with callbacks
- Invalid audio format

**Debug Steps**:
1. Check malloc results in logs
2. Verify audio format parameters
3. Check for null pointer accesses

## Code Review Summary

All code review issues have been addressed:
- ✅ Memory leaks fixed (proper cleanup on all paths)
- ✅ Error checking added for all NAPI calls
- ✅ Critical finally block bug fixed (rendererInitialized tracking)
- ✅ Type safety improved (removed catch-all interface member)
- ✅ Documentation added (call sequences, preconditions)
- ✅ Race conditions prevented (callback registration before start)
- ✅ Resource management improved (RAII-like patterns)

## Security Review

No security vulnerabilities introduced:
- ✅ All memory allocations have null checks
- ✅ No buffer overflows (using defined constants)
- ✅ Proper input validation on all NAPI methods
- ✅ Threadsafe functions properly managed
- ✅ No race conditions in cleanup

## Conclusion

This implementation correctly uses the single-pipeline real-time playback approach as designed in `RealTimePlaying.cpp`. The audio renderer callback-driven architecture ensures proper frame-by-frame processing with real-time output to the speaker, while also recording the processed audio for verification.

The code is production-ready pending successful testing on actual HarmonyOS devices/emulators with the DevEco Studio build system.
