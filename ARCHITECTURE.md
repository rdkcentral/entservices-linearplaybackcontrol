# LinearPlaybackControl Plugin Architecture

## Overview

The LinearPlaybackControl plugin is a WPEFramework (Thunder) plugin that provides control capabilities for linear broadcast playback through demuxer interfaces. It enables applications to manage live TV channel selection, trick play operations (pause, fast-forward, rewind), seeking capabilities, and stream status monitoring for STB (Set-Top Box) platforms.

## System Architecture

### Component Design

The plugin follows a modular architecture built on the Thunder framework, consisting of the following key components:

1. **LinearPlaybackControl Plugin**: Core plugin class implementing Thunder's `PluginHost::IPlugin` and `PluginHost::JSONRPC` interfaces, providing the main service entry point and lifecycle management.

2. **IDemuxer Interface**: Abstract demuxer interface defining generic operations for channel control, seek functionality, trick play, and stream status retrieval. This abstraction enables support for different hardware backends.

3. **DemuxerStreamFsFCC**: Concrete implementation of IDemuxer for Nokia FCC (Forward Content Cache) demuxer, utilizing filesystem-based control mechanisms through StreamFS.

4. **FileSelectListener**: Event monitoring component that watches trick play control files and triggers notifications on speed changes.

5. **LinearConfig**: Configuration management component handling plugin initialization parameters including mount points and feature flags.

### Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│         JSON-RPC API (Client Interface)             │
├─────────────────────────────────────────────────────┤
│      LinearPlaybackControl Plugin (Thunder)         │
├─────────────────────────────────────────────────────┤
│            IDemuxer (Abstraction Layer)             │
├─────────────────────────────────────────────────────┤
│  DemuxerStreamFsFCC   │  FileSelectListener        │
├─────────────────────────────────────────────────────┤
│      StreamFS Filesystem Interface (Nokia FCC)      │
├─────────────────────────────────────────────────────┤
│         Hardware Demuxer / Tuner Backend            │
└─────────────────────────────────────────────────────┘
```

## Component Interactions and Data Flow

### Initialization Flow

1. Thunder framework loads the LinearPlaybackControl plugin and invokes `Initialize()` with configuration parameters
2. Plugin parses configuration to extract mount point and StreamFS enablement flag
3. If StreamFS is enabled, creates DemuxerStreamFsFCC instance with configured mount point
4. DemuxerStreamFsFCC establishes filesystem paths for control interfaces (channel, seek, trickplay, status)
5. FileSelectListener is initialized to monitor trick play file for speed change events
6. Plugin registers JSON-RPC endpoints for external API access

### Runtime Operation Flow

**Channel Selection:**
- Client sends `set_channel` JSON-RPC request with channel identifier
- Plugin delegates to DemuxerStreamFsFCC's `setChannel()` method
- DemuxerStreamFsFCC writes channel data to StreamFS `chan_select` file
- Hardware demuxer processes the channel change request
- Plugin returns success/error status to client

**Seek Operation:**
- Client sends `set_seek` request with position in seconds
- Plugin validates request and forwards to DemuxerStreamFsFCC
- DemuxerStreamFsFCC writes seek position to StreamFS `seek` file
- Hardware updates playback position accordingly
- Client can query current position via `get_seek` endpoint

**Trick Play Control:**
- Client sends `set_trickplay` request with speed parameter (e.g., 2x, -1x)
- DemuxerStreamFsFCC writes speed value to `trick_play` file
- FileSelectListener detects file modification and triggers notification
- Plugin broadcasts `speedchanged` event to registered listeners
- Hardware adjusts playback rate based on speed parameter

**Status Monitoring:**
- Client periodically queries `get_status` endpoint
- Plugin reads from StreamFS `stream_status` file
- Returns stream health data including source loss status and loss count

## Plugin Framework Integration

The LinearPlaybackControl plugin integrates with Thunder framework through:

- **Plugin Lifecycle**: Implements `Initialize()` and `Deinitialize()` for resource management
- **Interface Mapping**: Exposes `IPlugin` and `IDispatcher` interfaces for Thunder runtime
- **JSON-RPC Communication**: All API endpoints accessible via JSON-RPC protocol
- **Configuration Management**: Utilizes Thunder's configuration system for runtime parameters
- **Event Notification**: Leverages Thunder's event mechanism for asynchronous notifications

## Dependencies and Interfaces

### External Dependencies

- **Thunder Framework**: Core plugin framework (WPEFramework SDK)
- **Thunder Plugins**: Base plugin infrastructure and utilities
- **Thunder Definitions**: Common type definitions and interfaces
- **Boost Algorithm**: String manipulation utilities
- **C++ Standard Library**: Threading, filesystem, streams

### API Interface

The plugin exposes the following JSON-RPC endpoints:

- `set_channel`: Configure demuxer channel selection
- `get_channel`: Retrieve current channel configuration
- `set_seek`: Set playback position in seconds
- `get_seek`: Query seek position and buffer status
- `set_trickplay`: Control playback speed for trick play
- `get_trickplay`: Retrieve current trick play speed
- `get_status`: Query stream status and health metrics
- `set_tracing`: Enable/disable debug tracing

### Events

- `speedchanged`: Notifies clients of trick play speed changes

## Technical Implementation Details

### StreamFS Integration

The plugin utilizes a filesystem-based control mechanism (StreamFS) where hardware demuxer operations are exposed as virtual files under a configured mount point. The typical structure:

```
{mountPoint}/fcc/
  ├── chan_select{demuxId}  (channel selection control)
  ├── seek{demuxId}         (seek position control)
  ├── trick_play{demuxId}   (playback speed control)
  └── stream_status         (status monitoring)
```

### Thread Safety

FileSelectListener operates on a separate monitoring thread to detect file changes without blocking the main plugin thread, ensuring responsive API performance.

### Error Handling

All demuxer operations return standardized IO_STATUS codes (OK, READ_ERROR, WRITE_ERROR, PARSE_ERROR) enabling consistent error propagation through the API stack.

### Configuration Schema

Plugin configuration includes:
- **MountPoint**: Base filesystem path for StreamFS control files
- **IsStreamFSEnabled**: Feature flag to enable/disable StreamFS functionality
- **Additional parameters**: Extensible for future demuxer implementations

## Scalability and Extensibility

The IDemuxer abstraction layer enables:
- Support for multiple demuxer backends (QAM, IP streaming, etc.)
- Multiple concurrent demuxer instances via demux ID indexing
- Vendor-specific implementations without API changes
- Future enhancement with minimal plugin modifications

---

*Architecture Version: 1.0.0*  
*Last Updated: February 2026*
