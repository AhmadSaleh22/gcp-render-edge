# Remote Rendering + WebRTC Solution
## Comprehensive Technical Documentation & Implementation Plan

---

## Document Information

**Project:** Manufacturing Hub 3D Workspace - Remote Rendering Solution
**Version:** 1.0
**Last Updated:** January 2025
**Status:** Planning Phase
**Authors:** Technical Architecture Team

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Analysis](#2-current-state-analysis)
3. [Solution Overview](#3-solution-overview)
4. [Technical Architecture](#4-technical-architecture)
5. [Implementation Roadmap](#5-implementation-roadmap)
6. [Infrastructure & Deployment](#6-infrastructure--deployment)
7. [Cost Analysis & Optimization](#7-cost-analysis--optimization)
8. [Performance Specifications](#8-performance-specifications)
9. [Security & Compliance](#9-security--compliance)
10. [Monitoring & Operations](#10-monitoring--operations)
11. [Risk Assessment & Mitigation](#11-risk-assessment--mitigation)
12. [Similar Projects & Case Studies](#12-similar-projects--case-studies)
13. [Appendices](#13-appendices)

---

## 1. Executive Summary

### 1.1 Problem Statement

The current Manufacturing Hub 3D Workspace application faces significant performance challenges:

- **Client-Side Rendering Bottlenecks**: Complex 3D scenes with production lines, work centers, and robotic arms cause performance degradation on low-to-mid-range devices
- **Asset Loading Issues**: Large STL/IGS models (manufacturing equipment) require substantial download bandwidth and client-side processing
- **Browser Limitations**: WebGL memory constraints and varying GPU capabilities across devices lead to inconsistent user experiences
- **Scalability Concerns**: As manufacturing facilities add more equipment and connections, client performance degrades exponentially

### 1.2 Proposed Solution

Implement a **Cloud-Based Remote Rendering + WebRTC Streaming** architecture that:

1. **Offloads 3D rendering** from client browsers to powerful GPU-equipped cloud servers
2. **Streams rendered frames** as H.264/H.265 video to clients via WebRTC with sub-50ms latency
3. **Handles user interactions** (camera movement, device selection, connection management) through lightweight WebRTC DataChannels
4. **Provides hybrid fallback** to client-side rendering for capable devices
5. **Scales automatically** based on concurrent user sessions

### 1.3 Key Benefits

| Benefit | Impact |
|---------|--------|
| **Universal Device Support** | Run on tablets, low-end laptops, and mobile devices without dedicated GPUs |
| **Consistent Performance** | Stable 60 FPS regardless of scene complexity (1,000+ devices in workspace) |
| **Reduced Bandwidth** | Stream 5-15 Mbps video vs. 100+ MB model downloads |
| **Enhanced Security** | Sensitive CAD models never leave cloud infrastructure |
| **Faster Load Times** | Sub-2 second session start vs. 10-30 second asset loading |

### 1.4 Investment Overview

- **Phase 1 PoC**: 2-3 weeks, $5,000-$10,000 (AWS NICE DCV testing)
- **Phase 2 MVP**: 6-8 weeks, $50,000-$75,000 (Custom WebRTC implementation)
- **Phase 3 Production**: 8-12 weeks, $75,000-$125,000 (Autoscaling + Monitoring)
- **Ongoing Operational Costs**: $0.50-$2.00 per user hour (AWS G5/G6 instances)

---

## 2. Current State Analysis

### 2.1 Technology Stack Assessment

#### Current Implementation

```
Frontend:
├── React 19.1.1
├── React Three Fiber 9.3.0
├── Three.js 0.169.0
├── @react-three/drei 10.7.6
└── Tailwind CSS 4.1.11

3D Rendering:
├── STLModelLoader (custom)
├── IGSModelLoader (three-iges-loader)
├── Device objects with drag-and-drop
├── Connection lines with animations
└── Camera views per device

Performance Optimizations (Already Implemented):
├── LOD (Level of Detail) system
├── Frustum culling
├── Distance-based culling
├── Optimized connection rendering
├── Performance monitoring
└── RequestAnimationFrame throttling
```

#### Current Performance Profile

Based on code analysis:

| Metric | Light Scene (5-10 devices) | Medium Scene (20-50 devices) | Heavy Scene (100+ devices) |
|--------|----------------------------|------------------------------|----------------------------|
| Load Time | 3-5s | 8-15s | 20-40s |
| FPS (High-end GPU) | 60 FPS | 55-60 FPS | 40-55 FPS |
| FPS (Integrated GPU) | 45-60 FPS | 25-40 FPS | 10-25 FPS |
| Memory Usage | 200-400 MB | 500-800 MB | 1-2 GB |
| Initial Download | 15-30 MB | 40-80 MB | 100-200 MB |

### 2.2 Key Pain Points

#### Performance Issues

1. **Device Objects**: Each device loads STL/IGS models with `STLModelLoader` and `IGSModelLoader`, causing:
   - Network latency for model downloads
   - CPU bottleneck during geometry parsing
   - GPU memory pressure from high polygon counts

2. **Connection Lines**: With 100+ devices and 200+ connections:
   - `EnhancedConnectionLine` components cause expensive re-renders
   - Animated flow particles (via shaders) stress fragment processing
   - Line raycasting for selection impacts frame rate

3. **Drag-and-Drop Operations**: Current implementation uses:
   - Continuous raycasting on pointer move (throttled to 32ms but still costly)
   - Real-time position updates triggering React re-renders
   - Plane intersection calculations every frame during drag

4. **Memory Leaks**: Identified in code comments:
   - Three.js geometries not properly disposed
   - Texture references held by disposed meshes
   - Event listeners not cleaned up on unmount

#### User Experience Challenges

- **Inconsistent Performance**: Engineers with MacBook Pros get 60 FPS, while factory floor tablets struggle at 15-20 FPS
- **Long Initial Load**: Downloading 100+ STL files sequentially takes 30-60 seconds
- **Mobile Device Failure**: iPads and Android tablets run out of memory with 50+ devices
- **Network Sensitivity**: Users in remote factory locations with limited bandwidth cannot load scenes

### 2.3 Target User Profiles

| User Profile | Device | Network | Use Case | Pain Point |
|--------------|--------|---------|----------|------------|
| **Factory Floor Manager** | iPad Pro | 4G/Wi-Fi (5-20 Mbps) | Real-time monitoring | Lag when viewing 100+ devices |
| **Remote Engineer** | Windows Laptop (Intel HD) | Home Wi-Fi (50 Mbps) | Production planning | Slow load times, choppy navigation |
| **Executive Dashboard** | MacBook Pro (M3) | Office Wi-Fi (200 Mbps) | High-level overview | Works fine but asset duplication waste |
| **Field Technician** | Android Tablet | 3G/4G (1-10 Mbps) | On-site diagnostics | App crashes with large scenes |

---

## 3. Solution Overview

### 3.1 Remote Rendering Architecture

Instead of rendering in the browser, **shift rendering to the cloud**:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT BROWSER                          │
│  ┌──────────────┐    ┌─────────────────┐   ┌────────────────┐  │
│  │  UI Controls │───▶│ WebRTC Client   │◀─▶│ Video Player   │  │
│  │  (React DOM) │    │ (DataChannel +  │   │ (<video> tag)  │  │
│  │              │    │  Video Stream)  │   │                │  │
│  └──────────────┘    └─────────────────┘   └────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │  ▲
                              │  │
                   User Input │  │ H.264/H.265 Video + Audio
                   (Commands) │  │ (5-15 Mbps)
                              ▼  │
┌─────────────────────────────────────────────────────────────────┐
│                      SIGNALING SERVER                           │
│                    (WebSocket / SDP/ICE)                        │
└─────────────────────────────────────────────────────────────────┘
                              │  ▲
                              ▼  │
┌─────────────────────────────────────────────────────────────────┐
│                   GPU RENDER WORKER (Cloud)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  React Three Fiber Scene (Server-Side Rendering)         │  │
│  │  ├── Device Objects (STL/IGS models loaded locally)      │  │
│  │  ├── Connection Lines                                    │  │
│  │  ├── Grid & Environment                                  │  │
│  │  └── Camera Control (from DataChannel)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  GPU Frame Buffer (1920x1080 @ 60 FPS)                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  NVENC Hardware Encoder                                   │  │
│  │  (H.264 Low-Latency / H.265 Main Profile)                │  │
│  │  Bitrate: 5-15 Mbps, Keyframe: 1s, Latency: <16ms        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                            ▼                                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  WebRTC Server (libwebrtc / GStreamer)                   │  │
│  │  ├── RTP Packet Streaming                                │  │
│  │  ├── DTLS-SRTP Encryption                                │  │
│  │  └── DataChannel for Input Commands                      │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Instance: AWS G5.xlarge (NVIDIA A10G GPU, 4 vCPU, 16 GB RAM)  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Interaction Flow

#### User Interaction Path

1. **User clicks device in browser**:
   ```
   Browser → DataChannel → GPU Worker
   Message: { type: "device:click", deviceId: "wc-001", timestamp: 1234567890 }
   ```

2. **GPU Worker processes**:
   - Performs raycasting with full-fidelity geometry
   - Updates selection state
   - Renders new frame with selection highlight
   - Encodes frame to H.264

3. **Video stream returns**:
   ```
   GPU Worker → WebRTC → Browser → <video> element
   Latency: 30-50ms total (10ms encode + 20-40ms network RTT)
   ```

#### Camera Control Path

1. **User drags to orbit**:
   ```
   Browser: onPointerMove → throttle → DataChannel
   Message: { type: "camera:orbit", deltaX: 10, deltaY: -5 }
   ```

2. **GPU Worker applies**:
   - Updates Three.js OrbitControls
   - Renders frame from new angle
   - Streams back to client

3. **Smooth experience through**:
   - Local echo prediction (optimistic UI updates)
   - Server-authoritative correction if mismatch
   - Frame interpolation on client side

### 3.3 Hybrid Mode Strategy

Not all users need remote rendering. Implement **intelligent routing**:

```javascript
// Client-side capability detection
async function detectRenderingCapability() {
  const { gpu, memory, bandwidth } = await detectHardware()

  if (gpu.tier >= 2 && memory > 4096 && bandwidth > 10) {
    return 'CLIENT_RENDER' // Use existing React Three Fiber
  } else if (gpu.tier === 1 || memory <= 2048) {
    return 'REMOTE_RENDER' // Force cloud rendering
  } else {
    return 'ADAPTIVE' // Start local, switch if FPS < 30 for 5s
  }
}
```

#### Adaptive Switching

```javascript
// Monitor client performance
const performanceMonitor = {
  averageFPS: 60,
  onFrame: () => {
    if (performanceMonitor.averageFPS < 30) {
      // Trigger switch to remote rendering
      switchToRemoteMode()
    }
  }
}
```

---

## 4. Technical Architecture

### 4.1 System Components

#### 4.1.1 Client Application

**Technology Stack**:
- React 19 (existing)
- WebRTC Browser API
- HTML5 `<video>` element for stream playback
- DataChannel for bidirectional communication

**Responsibilities**:
- UI rendering (panels, buttons, device cards)
- Capturing user input (mouse, touch, keyboard)
- Sending interaction commands via DataChannel
- Receiving and displaying video stream
- Local performance monitoring

**Implementation Example**:

```typescript
// WebRTC client setup
class RemoteRenderClient {
  private peerConnection: RTCPeerConnection
  private dataChannel: RTCDataChannel
  private videoElement: HTMLVideoElement

  async connect(signalingServerUrl: string, sessionId: string) {
    // 1. Create peer connection
    this.peerConnection = new RTCPeerConnection({
      iceServers: [
        { urls: 'stun:stun.l.google.com:19302' },
        {
          urls: 'turn:turn.example.com:3478',
          username: 'user',
          credential: 'pass'
        }
      ]
    })

    // 2. Setup data channel for input
    this.dataChannel = this.peerConnection.createDataChannel('input', {
      ordered: false, // Low latency over reliability
      maxRetransmits: 0
    })

    // 3. Handle incoming video stream
    this.peerConnection.ontrack = (event) => {
      this.videoElement.srcObject = event.streams[0]
    }

    // 4. Signaling via WebSocket
    const ws = new WebSocket(signalingServerUrl)
    ws.onmessage = async (msg) => {
      const signal = JSON.parse(msg.data)
      if (signal.type === 'offer') {
        await this.peerConnection.setRemoteDescription(signal.sdp)
        const answer = await this.peerConnection.createAnswer()
        await this.peerConnection.setLocalDescription(answer)
        ws.send(JSON.stringify({ type: 'answer', sdp: answer, sessionId }))
      } else if (signal.type === 'ice-candidate') {
        await this.peerConnection.addIceCandidate(signal.candidate)
      }
    }

    // 5. Send offer
    const offer = await this.peerConnection.createOffer()
    await this.peerConnection.setLocalDescription(offer)
    ws.send(JSON.stringify({ type: 'offer', sdp: offer, sessionId }))
  }

  sendInput(command: InputCommand) {
    if (this.dataChannel.readyState === 'open') {
      this.dataChannel.send(JSON.stringify(command))
    }
  }
}

// Usage in React component
function RemoteWorkspace() {
  const videoRef = useRef<HTMLVideoElement>(null)
  const clientRef = useRef<RemoteRenderClient>()

  useEffect(() => {
    const client = new RemoteRenderClient()
    client.videoElement = videoRef.current!
    client.connect('wss://signal.example.com', sessionStorage.getItem('sessionId'))
    clientRef.current = client
  }, [])

  const handleMouseMove = (e: React.MouseEvent) => {
    clientRef.current?.sendInput({
      type: 'pointer:move',
      x: e.clientX,
      y: e.clientY,
      timestamp: Date.now()
    })
  }

  return (
    <div onMouseMove={handleMouseMove}>
      <video ref={videoRef} autoPlay playsInline />
      <ControlPanel /> {/* React UI overlay */}
    </div>
  )
}
```

#### 4.1.2 Signaling Server

**Technology Stack**:
- Node.js 20+
- WebSocket (ws library)
- Redis (session state)

**Responsibilities**:
- WebRTC SDP/ICE exchange
- Session management
- Client ↔ GPU Worker pairing
- Health checks

**Implementation Example**:

```typescript
// signaling-server.ts
import WebSocket from 'ws'
import Redis from 'ioredis'

const redis = new Redis(process.env.REDIS_URL)
const wss = new WebSocket.Server({ port: 8080 })

interface Session {
  clientId: string
  workerId: string
  status: 'connecting' | 'active' | 'disconnected'
}

wss.on('connection', async (ws, req) => {
  const clientId = new URL(req.url!, 'ws://localhost').searchParams.get('clientId')

  ws.on('message', async (data) => {
    const message = JSON.parse(data.toString())

    switch (message.type) {
      case 'request-session':
        // Allocate GPU worker
        const workerId = await allocateWorker(clientId)
        await redis.set(`session:${clientId}`, JSON.stringify({
          clientId,
          workerId,
          status: 'connecting'
        }))

        // Connect client to worker
        const workerWs = await getWorkerWebSocket(workerId)
        ws.send(JSON.stringify({ type: 'worker-assigned', workerId }))
        break

      case 'offer':
      case 'answer':
      case 'ice-candidate':
        // Forward to peer
        const session: Session = JSON.parse(await redis.get(`session:${clientId}`) || '{}')
        const peerWs = message.from === 'client'
          ? await getWorkerWebSocket(session.workerId)
          : await getClientWebSocket(session.clientId)
        peerWs.send(data)
        break
    }
  })
})

async function allocateWorker(clientId: string): Promise<string> {
  // Query orchestrator for available GPU worker
  const response = await fetch('http://orchestrator:3000/allocate', {
    method: 'POST',
    body: JSON.stringify({ clientId, requiredGPU: 'A10G' })
  })
  const { workerId } = await response.json()
  return workerId
}
```

#### 4.1.3 GPU Render Workers

**Technology Stack**:
- Node.js 20+ (for WebRTC server)
- GStreamer 1.24+ with gst-plugins-good/bad/ugly
- NVENC plugin (hardware encoding)
- Three.js / React Three Fiber (headless rendering via Xvfb or EGL)

**Alternative Stack**:
- Unreal Engine 5 with Pixel Streaming plugin
- Unity with Render Streaming package
- Custom C++/OpenGL with WebRTC C++ library

**Responsibilities**:
- Load and render 3D scene
- Process input commands from DataChannel
- Encode frames with NVENC
- Stream via WebRTC to client

**Implementation Example (Node.js + GStreamer)**:

```typescript
// gpu-worker.ts
import { spawn } from 'child_process'
import { Server as WebSocketServer } from 'ws'

class GPURenderWorker {
  private gstreamerProcess: ChildProcess
  private scene: ThreeJSScene

  async start() {
    // 1. Initialize Three.js scene (headless)
    this.scene = await initializeScene()

    // 2. Start GStreamer pipeline
    const gstPipeline = `
      appsrc name=source format=time is-live=true
      ! video/x-raw,format=RGB,width=1920,height=1080,framerate=60/1
      ! nvh264enc preset=low-latency-hq rc-mode=cbr bitrate=8000 gop-size=60
      ! h264parse
      ! rtph264pay config-interval=1 pt=96
      ! udpsink host=127.0.0.1 port=5000
    `

    this.gstreamerProcess = spawn('gst-launch-1.0', gstPipeline.split(' '))

    // 3. Render loop
    this.renderLoop()
  }

  private async renderLoop() {
    const targetFPS = 60
    const frameTime = 1000 / targetFPS

    while (true) {
      const startTime = Date.now()

      // Render frame
      const frameBuffer = this.scene.render()

      // Send to GStreamer appsrc
      this.gstreamerProcess.stdin.write(frameBuffer)

      // Frame pacing
      const elapsed = Date.now() - startTime
      if (elapsed < frameTime) {
        await sleep(frameTime - elapsed)
      }
    }
  }

  handleInputCommand(command: InputCommand) {
    switch (command.type) {
      case 'camera:orbit':
        this.scene.camera.rotation.x += command.deltaY * 0.01
        this.scene.camera.rotation.y += command.deltaX * 0.01
        break
      case 'device:click':
        const device = this.scene.raycast(command.x, command.y)
        if (device) {
          this.scene.selectDevice(device.id)
        }
        break
    }
  }
}
```

**Better Approach: Unreal Engine Pixel Streaming**

```ini
; UnrealEngine/Config/DefaultPixelStreaming.ini
[/Script/PixelStreaming.PixelStreamingSettings]
bStartOnLaunch=True
StreamerPort=8888
StreamFPS=60
WebRTCMaxBitrate=15000
WebRTCMinBitrate=5000
WebRTCTargetBitrate=8000
EnableFillerData=False
CaptureUseFence=True
UseBackBufferCaptureSize=True
```

#### 4.1.4 Orchestrator / Session Manager

**Technology Stack**:
- Node.js / Go / Python
- PostgreSQL (session metadata)
- Redis (session state)
- AWS SDK (EC2 Auto Scaling)

**Responsibilities**:
- Provision GPU workers on demand
- Route clients to workers
- Monitor worker health
- Scale workers based on queue depth
- Manage session lifecycle

**Implementation Example**:

```typescript
// orchestrator.ts
import express from 'express'
import AWS from 'aws-sdk'
import Redis from 'ioredis'

const app = express()
const ec2 = new AWS.EC2({ region: 'me-south-1' }) // Bahrain for Egypt users
const redis = new Redis()

interface WorkerPool {
  workers: Worker[]
  warmPool: number
  maxWorkers: number
}

const pool: WorkerPool = {
  workers: [],
  warmPool: 2, // Keep 2 workers ready
  maxWorkers: 50
}

app.post('/allocate', async (req, res) => {
  const { clientId, requiredGPU } = req.body

  // 1. Find available worker
  let worker = pool.workers.find(w => w.status === 'idle')

  // 2. If none available, provision new one
  if (!worker) {
    if (pool.workers.length >= pool.maxWorkers) {
      return res.status(503).json({ error: 'No workers available, queue full' })
    }

    worker = await provisionWorker(requiredGPU)
    pool.workers.push(worker)
  }

  // 3. Assign to client
  worker.status = 'busy'
  worker.clientId = clientId
  await redis.set(`worker:${worker.id}`, JSON.stringify(worker))

  res.json({ workerId: worker.id, endpoint: worker.endpoint })
})

async function provisionWorker(gpuType: string): Promise<Worker> {
  const instanceType = gpuType === 'A10G' ? 'g5.xlarge' : 'g4dn.xlarge'

  const params = {
    ImageId: 'ami-0123456789abcdef0', // Custom AMI with our worker software
    InstanceType: instanceType,
    MinCount: 1,
    MaxCount: 1,
    UserData: Buffer.from(`
      #!/bin/bash
      cd /opt/gpu-worker
      npm start
    `).toString('base64'),
    TagSpecifications: [{
      ResourceType: 'instance',
      Tags: [
        { Key: 'Service', Value: 'remote-render-worker' },
        { Key: 'GPU', Value: gpuType }
      ]
    }]
  }

  const result = await ec2.runInstances(params).promise()
  const instanceId = result.Instances![0].InstanceId!

  // Wait for instance to be running
  await ec2.waitFor('instanceRunning', { InstanceIds: [instanceId] }).promise()

  // Get public IP
  const description = await ec2.describeInstances({ InstanceIds: [instanceId] }).promise()
  const publicIp = description.Reservations![0].Instances![0].PublicIpAddress!

  return {
    id: instanceId,
    endpoint: `ws://${publicIp}:8080`,
    status: 'idle',
    gpuType,
    createdAt: new Date()
  }
}

// Auto-scaling logic
setInterval(async () => {
  const idleWorkers = pool.workers.filter(w => w.status === 'idle').length
  const totalWorkers = pool.workers.length

  // Scale up if no idle workers and not at max
  if (idleWorkers === 0 && totalWorkers < pool.maxWorkers) {
    await provisionWorker('A10G')
  }

  // Scale down if too many idle for >5 minutes
  const oldIdle = pool.workers.filter(w =>
    w.status === 'idle' &&
    Date.now() - w.lastUsed > 5 * 60 * 1000
  )

  if (oldIdle.length > pool.warmPool) {
    const toTerminate = oldIdle.slice(pool.warmPool)
    for (const worker of toTerminate) {
      await ec2.terminateInstances({ InstanceIds: [worker.id] }).promise()
      pool.workers = pool.workers.filter(w => w.id !== worker.id)
    }
  }
}, 30000) // Every 30 seconds
```

#### 4.1.5 STUN/TURN Servers

**STUN (Session Traversal Utilities for NAT)**:
- Public STUN servers: `stun:stun.l.google.com:19302`
- Or self-hosted: coturn on t3.small instance

**TURN (Traversal Using Relays around NAT)**:
- Required when direct P2P fails (corporate firewalls)
- Self-hosted coturn recommended for security

**Coturn Setup**:

```bash
# Install coturn
sudo apt install coturn

# Edit /etc/turnserver.conf
listening-port=3478
tls-listening-port=5349
external-ip=YOUR_PUBLIC_IP
realm=turn.example.com
server-name=turn.example.com
lt-cred-mech
user=username:password
no-tcp-relay
no-multicast-peers
```

### 4.2 Data Flow Specifications

#### 4.2.1 Video Stream Encoding

**Codec Selection**:
- **H.264** (baseline/main profile): Maximum compatibility, all browsers support
- **H.265/HEVC**: 30-50% better compression, limited browser support (Safari only)
- **AV1**: Best compression, very limited support, high CPU decode cost

**Recommended: H.264 with NVENC**

**Encoding Parameters**:

```
Encoder: NVENC (NVIDIA GPU hardware encoder)
Profile: High
Level: 4.2
Preset: low-latency-hq (low latency, high quality)
Rate Control: CBR (Constant Bitrate) or VBR (Variable Bitrate)
Bitrate:
  - 1080p30: 5-8 Mbps
  - 1080p60: 8-12 Mbps
  - 1440p60: 10-20 Mbps
  - 4K60: 20-40 Mbps
GOP (Group of Pictures): 60 frames (1 second at 60 FPS)
Keyframe Interval: 1-2 seconds
Intra Refresh: Enabled (reduces keyframe size)
B-Frames: 0 (minimizes latency)
Slices: 4-8 (parallel encoding)
```

**GStreamer Pipeline Example**:

```bash
gst-launch-1.0 \
  appsrc name=source format=time is-live=true \
  ! video/x-raw,format=RGB,width=1920,height=1080,framerate=60/1 \
  ! nvh264enc \
      preset=low-latency-hq \
      rc-mode=cbr \
      bitrate=8000 \
      gop-size=60 \
      zerolatency=true \
      vbv-buffer-size=8000 \
      aq-strength=15 \
  ! h264parse config-interval=1 \
  ! rtph264pay config-interval=1 pt=96 mtu=1200 \
  ! udpsink host=127.0.0.1 port=5000
```

#### 4.2.2 Input Command Protocol

**DataChannel Configuration**:

```typescript
const dataChannel = peerConnection.createDataChannel('input', {
  ordered: false,        // Unordered for lowest latency
  maxRetransmits: 0,     // No retransmits, drop old packets
  maxPacketLifeTime: 100 // Drop packets older than 100ms
})
```

**Message Format (JSON)**:

```typescript
interface InputCommand {
  type: string
  timestamp: number
  data: any
}

// Camera orbit
{
  type: 'camera:orbit',
  timestamp: 1704067200000,
  data: { deltaX: 10, deltaY: -5 }
}

// Device click
{
  type: 'device:click',
  timestamp: 1704067200000,
  data: {
    screenX: 500,
    screenY: 300,
    deviceId: 'wc-001' // Optional, if detected client-side
  }
}

// Device drag
{
  type: 'device:drag',
  timestamp: 1704067200000,
  data: {
    deviceId: 'wc-001',
    positionX: 10.5,
    positionY: 0.2,
    positionZ: -5.3
  }
}

// Connection create
{
  type: 'connection:create',
  timestamp: 1704067200000,
  data: {
    sourceDeviceId: 'line-001',
    targetDeviceId: 'wc-001',
    type: 'production_flow'
  }
}
```

**Bandwidth Estimation**:
- Average command size: 100-200 bytes
- Frequency: 10-60 commands/second (during interaction)
- Bandwidth: ~1-10 KB/s (negligible compared to video)

#### 4.2.3 State Synchronization

**Server-Authoritative Model**:
- All scene state (device positions, connections, selections) stored on GPU worker
- Client sends commands, server validates and applies
- Client receives state updates via DataChannel or embedded in video (overlay)

**State Update Messages**:

```typescript
// Scene state snapshot (sent on connect or reconnect)
{
  type: 'state:full',
  timestamp: 1704067200000,
  data: {
    devices: [
      { id: 'wc-001', type: 'work_center', position: [0, 0.2, -6], ... },
      // ... all devices
    ],
    connections: [
      { id: 'conn-1', sourceDeviceId: 'line-001', targetDeviceId: 'wc-001', ... },
      // ... all connections
    ],
    selectedDevice: 'wc-001',
    cameraPosition: [20, 15, 20],
    cameraRotation: [0.5, 0.3, 0]
  }
}

// Incremental update (sent after user action)
{
  type: 'state:update',
  timestamp: 1704067200000,
  data: {
    devicesUpdated: [
      { id: 'wc-001', position: [5, 0.2, -6] } // Only changed device
    ],
    selectedDevice: 'wc-001'
  }
}
```

### 4.3 Latency Budget

**Target: Glass-to-Glass < 80ms**

Breakdown:

| Stage | Target Latency | Notes |
|-------|----------------|-------|
| Input Capture | 1-2ms | Browser event → DataChannel |
| Network Upload | 10-40ms | RTT/2, depends on region |
| Input Processing | 1-3ms | Deserialize JSON, apply to scene |
| Scene Render | 16ms | 60 FPS = 16.67ms per frame |
| NVENC Encoding | 3-8ms | Hardware encoder latency |
| Network Download | 10-40ms | RTT/2 |
| Video Decode | 5-10ms | Browser hardware decoder |
| Display | 8-16ms | Monitor refresh (60-120 Hz) |
| **Total** | **54-137ms** | Typical: 60-80ms |

**Optimization Strategies**:

1. **Reduce Network RTT**:
   - Deploy workers in region closest to users (Bahrain for Egypt)
   - Use AWS Global Accelerator for optimized routing
   - Target RTT < 30ms

2. **Reduce Encoding Latency**:
   - Use NVENC low-latency preset (3-5ms vs. 8-12ms for quality preset)
   - Minimize GOP size (60 frames = 1 second at 60 FPS)
   - Disable B-frames

3. **Reduce Rendering Time**:
   - Use LOD system on server (same as client)
   - Cull off-screen objects
   - Limit scene complexity (1000 devices max per session)

4. **Optimize Input Handling**:
   - Prioritize input messages over state updates
   - Use unordered DataChannel to skip old packets

---

## 5. Implementation Roadmap

### 5.1 Phase 0: Proof of Concept (2-3 weeks)

**Goal**: Validate feasibility with minimal code changes

**Approach**: Use AWS NICE DCV or Amazon AppStream 2.0 (managed services)

#### Tasks:

1. **Week 1: Infrastructure Setup**
   - [ ] Create AWS account / set up billing alerts
   - [ ] Launch G5.xlarge instance in me-south-1 (Bahrain) region
   - [ ] Install NICE DCV on instance
   - [ ] Install Node.js, npm, and project dependencies
   - [ ] Clone repository and build production bundle
   - [ ] Run app on instance with `npm run build && npm run preview`
   - [ ] Configure NICE DCV to stream desktop at 1080p60, 8 Mbps

2. **Week 2: Testing & Metrics**
   - [ ] Test from Egypt with various devices:
     - Desktop (high-end GPU)
     - Laptop (integrated GPU)
     - Tablet (iPad/Android)
   - [ ] Measure latency with manual stopwatch test:
     - Click device → highlight appears
     - Target: < 100ms perceived latency
   - [ ] Measure bandwidth usage:
     - Monitor network panel in browser DevTools
     - Target: 5-10 Mbps steady
   - [ ] Load test with complex scene (100+ devices)
   - [ ] Document findings in `docs/poc-results.md`

3. **Week 3: Stakeholder Demo & Decision**
   - [ ] Record demo video showing:
     - Tablet running heavy scene smoothly
     - Latency comparison (local vs. remote)
     - Multiple concurrent users
   - [ ] Present cost analysis for 10, 50, 100 concurrent users
   - [ ] Get approval to proceed to Phase 1 (MVP)

#### Deliverables:

- [ ] Working NICE DCV setup streaming current app
- [ ] Performance benchmark report
- [ ] Cost projection spreadsheet
- [ ] Go/No-Go decision document

#### Success Criteria:

- ✅ Latency < 80ms from Cairo/Alexandria to Bahrain
- ✅ Smooth 60 FPS on tablet with 100+ device scene
- ✅ Cost < $2/user/hour
- ✅ Stakeholder approval

### 5.2 Phase 1: Custom WebRTC MVP (6-8 weeks)

**Goal**: Build production-ready custom WebRTC solution with autoscaling

#### Architecture:

```
[Client Browser] ←WebRTC→ [Signaling Server] ←→ [GPU Worker Pool]
                              ↕
                        [Orchestrator]
                              ↕
                        [AWS Auto Scaling]
```

#### Tasks:

**Week 1-2: Client Application**

- [ ] Create `RemoteRenderClient` class wrapping WebRTC APIs
- [ ] Implement SDP/ICE exchange with signaling server
- [ ] Display video stream in `<video>` element
- [ ] Send input commands via DataChannel:
  - Mouse/touch events
  - Camera controls
  - Device selection
- [ ] Add connection quality monitoring (RTT, packet loss, bitrate)
- [ ] Implement local echo for camera movements (optimistic updates)

**Week 3-4: Signaling Server**

- [ ] Create Node.js WebSocket server
- [ ] Implement SDP/ICE relay between client and worker
- [ ] Add Redis for session state storage
- [ ] Implement session lifecycle:
  - Create session on client connect
  - Allocate worker via orchestrator
  - Pair client ↔ worker
  - Cleanup on disconnect
- [ ] Add health check endpoint for load balancer
- [ ] Deploy to AWS ECS Fargate (serverless containers)

**Week 5-6: GPU Render Worker**

Option A: **Node.js + GStreamer** (Recommended for React Three Fiber)

- [ ] Set up headless rendering with Xvfb or EGL
- [ ] Initialize React Three Fiber scene server-side
- [ ] Load devices and connections from API
- [ ] Implement input command handlers:
  - Update camera position/rotation
  - Update selected device
  - Add/remove devices and connections
- [ ] Set up GStreamer pipeline with NVENC
- [ ] Feed rendered frames to GStreamer appsrc
- [ ] Implement WebRTC server (libwebrtc or GStreamer webrtcbin)
- [ ] Create AMI with all dependencies pre-installed

Option B: **Unreal Engine Pixel Streaming** (Better for future VR/AR)

- [ ] Rebuild scene in Unreal Engine 5
- [ ] Import STL/IGS models as Static Meshes
- [ ] Recreate device interaction logic in Blueprints
- [ ] Enable Pixel Streaming plugin
- [ ] Configure encoding settings (see 4.2.1)
- [ ] Test local streaming
- [ ] Package as Linux binary
- [ ] Create AMI with Unreal app + Pixel Streaming

**Week 7: Orchestrator**

- [ ] Create Node.js/Go service for worker management
- [ ] Implement `/allocate` endpoint:
  - Find idle worker OR provision new EC2 instance
  - Return worker endpoint to signaling server
- [ ] Implement auto-scaling logic:
  - Monitor queue depth (waiting clients)
  - Scale up if queue > 0 and workers < max
  - Scale down if idle workers > warm pool for > 5 min
- [ ] Add warm pool (2-5 instances ready)
- [ ] Integrate with AWS Auto Scaling Groups
- [ ] Add CloudWatch metrics

**Week 8: Integration & Testing**

- [ ] End-to-end testing:
  - Client connects → signaling → worker allocated → stream starts
  - Interact with scene → verify commands processed
  - Disconnect → worker returns to pool or terminates
- [ ] Load testing with 10, 50, 100 concurrent users (Locust or Artillery)
- [ ] Latency testing from multiple regions
- [ ] Failover testing (kill worker mid-session)
- [ ] Security audit (WebRTC encryption, worker isolation)

#### Deliverables:

- [ ] Client library: `@manufacturing-hub/remote-render-client`
- [ ] Signaling server: `signaling-server/` (Docker image)
- [ ] GPU worker: `gpu-worker/` (AMI image)
- [ ] Orchestrator: `orchestrator/` (Docker image)
- [ ] Infrastructure as Code: Terraform or CloudFormation templates
- [ ] Deployment guide: `docs/deployment-guide.md`

#### Success Criteria:

- ✅ Sub-50ms latency (P95) from target regions
- ✅ 60 FPS stable with 100+ device scenes
- ✅ Autoscaling works: 0→100 users in < 2 minutes
- ✅ < 1% packet loss on good networks
- ✅ Worker provisioning time < 60 seconds (warm pool)

### 5.3 Phase 2: Production Hardening (8-12 weeks)

**Goal**: Enterprise-grade reliability, security, and monitoring

#### Tasks:

**Week 1-3: Hybrid Mode**

- [ ] Implement client capability detection:
  - GPU tier (WebGL renderer string)
  - Available RAM (navigator.deviceMemory)
  - Network bandwidth (navigator.connection.downlink)
- [ ] Create routing logic:
  - High-end devices: client render
  - Low-end devices: remote render
  - Mid-tier: adaptive (start client, switch if FPS < 30)
- [ ] Implement mode switching without page reload:
  - Persist scene state in backend API
  - Reload state in new mode
- [ ] Add UI toggle for manual mode selection (user preference)

**Week 4-6: Advanced Features**

- [ ] Multi-user collaboration:
  - Multiple clients view same scene (read-only)
  - One "driver" can edit, others observe
  - Cursor synchronization (show other users' pointers)
- [ ] Recording and playback:
  - Record video stream server-side (FFmpeg)
  - Save session commands for replay
  - "Time travel" debugging (replay past sessions)
- [ ] Mobile optimizations:
  - Touch gesture handling (pinch zoom, two-finger pan)
  - Adaptive bitrate for cellular networks
  - Low-power mode (30 FPS, lower resolution)
- [ ] Accessibility:
  - Keyboard-only navigation
  - Screen reader support for UI controls
  - High contrast mode

**Week 7-9: Security**

- [ ] Authentication & Authorization:
  - JWT tokens for session creation
  - Role-based access (admin, engineer, viewer)
  - Per-project permissions
- [ ] Worker isolation:
  - Docker containers for each session
  - Network isolation (VPC, security groups)
  - No shared GPU memory between tenants
- [ ] Encryption:
  - DTLS-SRTP for WebRTC (enabled by default)
  - TLS for signaling server (WSS)
  - Encrypted storage for session recordings
- [ ] DDoS protection:
  - AWS WAF in front of signaling server
  - Rate limiting (max 5 sessions per user)
  - CloudFront CDN with DDoS protection
- [ ] Compliance:
  - GDPR: data deletion on request
  - Audit logs: all session activity logged

**Week 10-12: Monitoring & Operations**

- [ ] Metrics collection:
  - Client: FPS, latency, packet loss, bitrate
  - Worker: GPU utilization, VRAM usage, encoding latency
  - Orchestrator: queue depth, active workers, scale events
- [ ] Dashboards (Grafana):
  - Real-time session map (world map with user locations)
  - Performance heatmap (latency by region)
  - Cost tracking (GPU hours, bandwidth)
- [ ] Alerting (PagerDuty/Slack):
  - P0: Orchestrator down, > 50% workers failed
  - P1: Avg latency > 100ms, GPU utilization > 90%
  - P2: Worker provisioning slow (> 90s)
- [ ] Logging (ELK or CloudWatch Logs Insights):
  - Centralized logs from all services
  - Structured JSON logging
  - Log retention: 30 days
- [ ] Tracing (OpenTelemetry → Jaeger):
  - End-to-end trace: client request → signaling → worker → response
  - Latency breakdown by stage
- [ ] Incident response playbook:
  - Runbooks for common issues
  - On-call rotation
  - Post-mortem template

#### Deliverables:

- [ ] Hybrid mode implementation
- [ ] Multi-user collaboration feature
- [ ] Security audit report
- [ ] Monitoring dashboards
- [ ] Incident response playbook
- [ ] Production deployment to AWS (multi-region)

#### Success Criteria:

- ✅ 99.9% uptime (< 45 min downtime/month)
- ✅ < 100ms P99 latency
- ✅ Zero security vulnerabilities (automated scanning)
- ✅ Complete observability (logs, metrics, traces)
- ✅ Successful DR drill (failover to backup region)

### 5.4 Phase 3: Scale & Optimize (Ongoing)

**Goal**: Support 1,000+ concurrent users, reduce cost by 50%

#### Tasks:

**Cost Optimization**:

- [ ] Spot instance support for workers (70% cost savings)
- [ ] Multi-session per GPU (2-4 sessions on G5.xlarge)
- [ ] Regional pricing analysis (cheapest regions)
- [ ] Reserved instances for base load
- [ ] S3 Intelligent-Tiering for recordings

**Performance Optimization**:

- [ ] WebRTC extensions:
  - Dependency Descriptor (L1T2/L1T3 spatial layers)
  - Frame marking (prioritize important frames)
- [ ] Edge deployment (AWS Wavelength on 5G networks)
- [ ] AV1 codec support (future-proofing)
- [ ] GPU ray tracing for photorealistic rendering

**Feature Enhancements**:

- [ ] VR/AR support (WebXR streaming)
- [ ] AI-assisted camera (auto-frame selected device)
- [ ] Voice commands ("Select work center 3")
- [ ] Real-time analytics (detect bottlenecks in production flow)

---

## 6. Infrastructure & Deployment

### 6.1 Cloud Provider Selection

**Recommended: AWS**

Reasons:
- **GPU Instances**: G5 (NVIDIA A10G), G4dn (T4), G6 (L4) available
- **Global Reach**: 33 regions, including me-south-1 (Bahrain) near Egypt
- **Managed Services**: NICE DCV, AppStream 2.0 for quick PoC
- **Ecosystem**: Strong support for WebRTC (AWS Kinesis Video Streams, AWS Chime SDK)

**Alternative: Azure**

- **GPU Instances**: NVv4 (AMD), NCv3 (V100), NVadsA10 v5 (A10)
- **Services**: Azure Remote Rendering (deprecated Sept 2025, but shows Microsoft investment)
- **Good for**: If already using Azure for other services

**Alternative: GCP**

- **GPU Instances**: A2 (A100), G2 (L4)
- **Good for**: Machine learning workloads, but less WebRTC tooling

### 6.2 Instance Types & Sizing

#### GPU Render Workers

| Instance Type | GPU | vCPU | RAM | Cost/hr | Sessions/GPU | Use Case |
|---------------|-----|------|-----|---------|--------------|----------|
| **AWS G5.xlarge** | A10G (24 GB) | 4 | 16 GB | ~$1.00 | 1-2 | Production (recommended) |
| **AWS G5.2xlarge** | A10G | 8 | 32 GB | ~$1.50 | 2-4 | High concurrency |
| **AWS G4dn.xlarge** | T4 (16 GB) | 4 | 16 GB | ~$0.50 | 1 | Cost-optimized |
| **AWS G6.xlarge** | L4 (24 GB) | 4 | 16 GB | ~$0.90 | 1-2 | AI + rendering hybrid |
| **Azure NVadsA10 v5** | A10 | 6 | 55 GB | ~$1.10 | 2-3 | Azure alternative |

**Recommendation for MVP**: G5.xlarge (best performance/cost balance)

**Spot Instance Strategy**: Use spot instances for 50-70% of fleet (70% discount), reserved/on-demand for baseline

#### Supporting Services

| Service | Instance Type | Cost/mo | Notes |
|---------|---------------|---------|-------|
| **Signaling Server** | ECS Fargate (0.5 vCPU, 1 GB RAM) | ~$15 | Autoscaling |
| **Orchestrator** | ECS Fargate (1 vCPU, 2 GB RAM) | ~$30 | Stateful, needs HA |
| **Redis (Session State)** | ElastiCache r6g.large | ~$150 | 13.07 GB RAM |
| **PostgreSQL (Metadata)** | RDS t4g.small | ~$30 | 2 GB RAM |
| **TURN Server** | t3.small | ~$15 | Self-hosted coturn |

### 6.3 Network Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                     Internet / Users                           │
└───────────────────────────┬────────────────────────────────────┘
                            │
                            ▼
        ┌───────────────────────────────────────┐
        │   CloudFront (Global CDN)             │
        │   - DDoS protection                   │
        │   - TLS termination                   │
        └───────────────┬───────────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────────┐
        │   ALB (Application Load Balancer)     │
        │   - Health checks                     │
        │   - SSL certificates                  │
        └───────────────┬───────────────────────┘
                        │
        ┌───────────────┴────────────────┐
        │                                │
        ▼                                ▼
┌──────────────────┐           ┌──────────────────┐
│ Signaling Server │           │   Static Assets  │
│  (ECS Fargate)   │           │   (S3 + CDN)     │
│  - WebSocket     │           │   - React app    │
│  - SDP/ICE relay │           │   - GLB models   │
└────────┬─────────┘           └──────────────────┘
         │
         ▼
┌──────────────────┐
│  Orchestrator    │──────────────┐
│  (ECS Fargate)   │              │
│  - Worker mgmt   │              │ AWS SDK
│  - Auto-scaling  │              │
└────────┬─────────┘              │
         │                        ▼
         │                ┌──────────────────────┐
         │                │  EC2 Auto Scaling    │
         │                │  - Launch templates  │
         │                │  - Scaling policies  │
         │                └──────────┬───────────┘
         │                           │
         ▼                           ▼
┌────────────────────────────────────────────┐
│         VPC (10.0.0.0/16)                  │
│  ┌──────────────────────────────────────┐ │
│  │  Public Subnet (10.0.1.0/24)         │ │
│  │  ┌────────────────────────────────┐  │ │
│  │  │  NAT Gateway                    │  │ │
│  │  └────────────────────────────────┘  │ │
│  └──────────────────────────────────────┘ │
│                                            │
│  ┌──────────────────────────────────────┐ │
│  │  Private Subnet (10.0.10.0/24)       │ │
│  │  ┌────────────────────────────────┐  │ │
│  │  │  GPU Worker Fleet              │  │ │
│  │  │  - G5.xlarge instances         │  │ │
│  │  │  - WebRTC endpoints            │  │ │
│  │  │  - Security Group: 8080/tcp    │  │ │
│  │  └────────────────────────────────┘  │ │
│  │                                       │ │
│  │  ┌────────────────────────────────┐  │ │
│  │  │  ElastiCache (Redis)           │  │ │
│  │  │  - Session state               │  │ │
│  │  └────────────────────────────────┘  │ │
│  │                                       │ │
│  │  ┌────────────────────────────────┐  │ │
│  │  │  RDS (PostgreSQL)              │  │ │
│  │  │  - User data, projects         │  │ │
│  │  └────────────────────────────────┘  │ │
│  └──────────────────────────────────────┘ │
└────────────────────────────────────────────┘
```

### 6.4 Deployment Process

#### Terraform Infrastructure as Code

```hcl
# main.tf
provider "aws" {
  region = "me-south-1" # Bahrain (closest to Egypt)
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support = true

  tags = {
    Name = "remote-render-vpc"
  }
}

# Public Subnet (for ALB, NAT)
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "me-south-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "remote-render-public"
  }
}

# Private Subnet (for GPU workers, databases)
resource "aws_subnet" "private" {
  vpc_id = aws_vpc.main.id
  cidr_block = "10.0.10.0/24"
  availability_zone = "me-south-1a"

  tags = {
    Name = "remote-render-private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
}

# NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id = aws_subnet.public.id
}

# Security Group for GPU Workers
resource "aws_security_group" "gpu_workers" {
  name = "gpu-workers-sg"
  vpc_id = aws_vpc.main.id

  # WebRTC signaling
  ingress {
    from_port = 8080
    to_port = 8080
    protocol = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }

  # WebRTC media (UDP)
  ingress {
    from_port = 49152
    to_port = 65535
    protocol = "udp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Outbound all
  egress {
    from_port = 0
    to_port = 0
    protocol = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Launch Template for GPU Workers
resource "aws_launch_template" "gpu_worker" {
  name = "gpu-worker-template"
  image_id = "ami-0123456789abcdef0" # Custom AMI with worker software
  instance_type = "g5.xlarge"

  iam_instance_profile {
    name = aws_iam_instance_profile.gpu_worker.name
  }

  vpc_security_group_ids = [aws_security_group.gpu_workers.id]

  user_data = base64encode(<<-EOF
    #!/bin/bash
    cd /opt/gpu-worker
    export ORCHESTRATOR_URL=${aws_lb.orchestrator.dns_name}
    npm start
  EOF
  )

  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "gpu-worker"
      Service = "remote-render"
    }
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "gpu_workers" {
  name = "gpu-workers-asg"
  min_size = 0
  max_size = 50
  desired_capacity = 2 # Warm pool

  vpc_zone_identifier = [aws_subnet.private.id]

  launch_template {
    id = aws_launch_template.gpu_worker.id
    version = "$Latest"
  }

  tag {
    key = "Name"
    value = "gpu-worker"
    propagate_at_launch = true
  }
}

# CloudWatch Alarms for Auto-Scaling
resource "aws_cloudwatch_metric_alarm" "scale_up" {
  alarm_name = "gpu-workers-scale-up"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods = 2
  metric_name = "QueueDepth"
  namespace = "RemoteRender"
  period = 60
  statistic = "Average"
  threshold = 1

  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}

resource "aws_autoscaling_policy" "scale_up" {
  name = "scale-up"
  scaling_adjustment = 1
  adjustment_type = "ChangeInCapacity"
  cooldown = 60
  autoscaling_group_name = aws_autoscaling_group.gpu_workers.name
}

# ECS Cluster for Signaling Server & Orchestrator
resource "aws_ecs_cluster" "main" {
  name = "remote-render-cluster"
}

# ECS Task Definition: Signaling Server
resource "aws_ecs_task_definition" "signaling_server" {
  family = "signaling-server"
  network_mode = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu = "512"
  memory = "1024"

  container_definitions = jsonencode([{
    name = "signaling-server"
    image = "${aws_ecr_repository.signaling_server.repository_url}:latest"
    portMappings = [{
      containerPort = 8080
      protocol = "tcp"
    }]
    environment = [
      { name = "REDIS_URL", value = "redis://${aws_elasticache_cluster.redis.cache_nodes[0].address}:6379" }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group" = aws_cloudwatch_log_group.signaling_server.name
        "awslogs-region" = "me-south-1"
        "awslogs-stream-prefix" = "signaling"
      }
    }
  }])
}

# ElastiCache (Redis)
resource "aws_elasticache_cluster" "redis" {
  cluster_id = "remote-render-redis"
  engine = "redis"
  node_type = "cache.r6g.large"
  num_cache_nodes = 1
  parameter_group_name = "default.redis7"
  subnet_group_name = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
}

# RDS (PostgreSQL)
resource "aws_db_instance" "postgres" {
  identifier = "remote-render-db"
  engine = "postgres"
  engine_version = "15.4"
  instance_class = "db.t4g.small"
  allocated_storage = 20
  storage_type = "gp3"
  db_name = "remoterender"
  username = "admin"
  password = random_password.db_password.result
  vpc_security_group_ids = [aws_security_group.postgres.id]
  db_subnet_group_name = aws_db_subnet_group.main.name
  skip_final_snapshot = true
}
```

#### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy Remote Render System

on:
  push:
    branches: [main]

jobs:
  build-client:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 20
      - run: npm ci
      - run: npm run build
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: me-south-1
      - run: aws s3 sync dist/ s3://remote-render-frontend/
      - run: aws cloudfront create-invalidation --distribution-id ${{ secrets.CF_DISTRIBUTION_ID }} --paths "/*"

  build-signaling-server:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: docker/setup-buildx-action@v2
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: me-south-1
      - run: |
          aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.ECR_REGISTRY }}
          docker build -t signaling-server ./signaling-server
          docker tag signaling-server:latest ${{ secrets.ECR_REGISTRY }}/signaling-server:latest
          docker push ${{ secrets.ECR_REGISTRY }}/signaling-server:latest
      - run: aws ecs update-service --cluster remote-render-cluster --service signaling-server --force-new-deployment

  build-gpu-worker-ami:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-packer@v2
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: me-south-1
      - run: |
          cd packer/gpu-worker
          packer build gpu-worker.pkr.hcl
      - run: |
          # Update launch template with new AMI
          NEW_AMI=$(aws ec2 describe-images --owners self --filters "Name=name,Values=gpu-worker-*" --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' --output text)
          aws ec2 create-launch-template-version --launch-template-name gpu-worker-template --source-version '$Latest' --launch-template-data "{\"ImageId\":\"$NEW_AMI\"}"
```

---

## 7. Cost Analysis & Optimization

### 7.1 Cost Breakdown

#### Baseline Assumptions:
- **100 concurrent users** during peak hours (8 hours/day)
- **20 concurrent users** during off-peak (16 hours/day)
- **22 business days per month**
- **Region**: AWS me-south-1 (Bahrain)

#### Monthly Cost Estimate:

| Component | Spec | Unit Cost | Usage | Monthly Cost |
|-----------|------|-----------|-------|--------------|
| **GPU Workers (Peak)** | G5.xlarge | $1.00/hr | 100 × 8hr × 22d = 17,600 hr | $17,600 |
| **GPU Workers (Off-Peak)** | G5.xlarge | $1.00/hr | 20 × 16hr × 22d = 7,040 hr | $7,040 |
| **Signaling Server** | ECS Fargate (0.5 vCPU, 1 GB) | $0.02/hr | 730 hr | $15 |
| **Orchestrator** | ECS Fargate (1 vCPU, 2 GB) | $0.04/hr | 730 hr | $30 |
| **Redis** | ElastiCache r6g.large | $0.21/hr | 730 hr | $153 |
| **PostgreSQL** | RDS t4g.small | $0.04/hr | 730 hr | $30 |
| **TURN Server** | t3.small | $0.02/hr | 730 hr | $15 |
| **Load Balancer** | ALB | $23/mo + $0.008/LCU | ~5 LCU | $63 |
| **Data Transfer Out** | Internet egress | $0.09/GB | 100 users × 10 Mbps × 8hr × 22d × 3.6 = 6,336 GB | $570 |
| **S3 Storage** | Standard | $0.025/GB | 50 GB (models, recordings) | $1.25 |
| **CloudWatch Logs** | Standard | $0.50/GB | 20 GB/month | $10 |
| **CloudFront** | CDN | $0.085/GB | 500 GB (static assets) | $42.50 |
| **Total** | | | | **$25,570/mo** |

**Per User Cost**: $25,570 / (100 peak + 20 off-peak) ≈ **$213/user/month** or **$1.21/user/hour**

### 7.2 Cost Optimization Strategies

#### 1. Spot Instances (70% Savings)

Use Spot Instances for 70% of GPU workers:

```
On-Demand: 30% × 17,600 hr × $1.00 = $5,280
Spot: 70% × 17,600 hr × $0.30 = $3,696
Savings: $8,624/month (35% total reduction)
```

**Implementation**:
- Auto Scaling Group with mixed instance policy
- Spot interruption handling (graceful session migration)

#### 2. Multi-Session Per GPU (2-4x Efficiency)

Render 2-4 sessions on one G5.xlarge:

```
Before: 100 users = 100 instances = $1,760/day
After: 100 users = 33 instances = $587/day
Savings: $1,173/day = $25,806/month (50% reduction)
```

**Implementation**:
- Docker containers per session (isolate GPU memory)
- NVIDIA MIG (Multi-Instance GPU) for A100 instances
- Load balancer routes multiple clients to same worker

#### 3. Reserved Instances for Base Load

Reserve 20 instances (baseline demand):

```
On-Demand: 20 × 730 hr × $1.00 = $14,600/year
Reserved (1-year): 20 × 730 hr × $0.65 = $9,490/year
Savings: $5,110/year = $426/month
```

#### 4. Regional Pricing Arbitrage

| Region | G5.xlarge Cost | Latency to Cairo | Recommendation |
|--------|----------------|-------------------|----------------|
| me-south-1 (Bahrain) | $1.00/hr | 20-40ms | **Best latency** |
| eu-central-1 (Frankfurt) | $0.90/hr | 40-60ms | 10% cheaper, acceptable latency |
| eu-south-1 (Milan) | $0.95/hr | 30-50ms | Balanced |
| me-central-1 (UAE) | $1.05/hr | 15-30ms | Lowest latency, 5% premium |

**Strategy**: Primary in Bahrain, failover to Frankfurt (save 10% on off-peak)

#### 5. Bandwidth Optimization

Reduce bitrate based on scene complexity:

| Scene Complexity | Bitrate | Bandwidth/hr | Cost/hr (100 users) |
|------------------|---------|--------------|---------------------|
| Low (< 20 devices) | 5 Mbps | 2.25 GB | $20.25 |
| Medium (20-50 devices) | 8 Mbps | 3.6 GB | $32.40 |
| High (50+ devices) | 12 Mbps | 5.4 GB | $48.60 |

**Dynamic Adaptation**: Start at 8 Mbps, reduce to 5 Mbps if network congestion detected

#### 6. Edge Caching (CloudFront)

Cache static assets (models, textures) at edge:

```
Before: 500 GB/month from S3 = $0.09/GB = $45
After: 500 GB/month from CloudFront = $0.085/GB = $42.50
Savings: $2.50/month (minor but reduces S3 egress)
```

### 7.3 Revised Cost Estimate (Optimized)

| Optimization | Monthly Savings | New Total |
|--------------|-----------------|-----------|
| **Baseline** | - | $25,570 |
| Spot Instances (70%) | -$8,624 | $16,946 |
| Multi-Session (2x) | -$12,903 | $4,043 |
| Reserved Instances | -$426 | $3,617 |
| Regional Arbitrage (10%) | -$362 | $3,255 |
| Bandwidth Optimization | -$200 | $3,055 |
| **Final Optimized** | **-$22,515** | **$3,055/mo** |

**Optimized Per User Cost**: $3,055 / 120 users ≈ **$25/user/month** or **$0.17/user/hour**

### 7.4 Cost Comparison vs. Alternatives

| Solution | Setup Cost | Monthly Cost (100 users) | Latency | Pros | Cons |
|----------|------------|-------------------------|---------|------|------|
| **Client Rendering (Current)** | $0 | $0 | 0ms | Free, instant | Performance issues, device limitations |
| **AWS NICE DCV** | $5,000 | $15,000 | 40ms | Managed, fast PoC | Limited customization, desktop streaming |
| **Custom WebRTC (Optimized)** | $75,000 | $3,055 | 40ms | Full control, scalable | Development cost, maintenance |
| **Unreal Pixel Streaming** | $100,000 | $4,000 | 30ms | Best visuals, VR-ready | Requires Unreal rebuild, higher cost |

### 7.5 ROI Analysis

#### Scenario: Replace 50 dedicated engineering workstations

**Before (Client Rendering + Workstations)**:
- 50 × Dell Precision 5680 (RTX 3500 Ada) = $3,500 each = $175,000 upfront
- 3-year lifespan = $58,333/year = $4,861/month
- Maintenance, IT support: $500/month
- **Total: $5,361/month**

**After (Remote Rendering)**:
- Development cost: $75,000 (amortized over 3 years = $2,083/month)
- Cloud cost (50 users, 8hr/day): $1,528/month (optimized)
- **Total: $3,611/month**

**Savings**: $5,361 - $3,611 = **$1,750/month = $21,000/year**

**Payback Period**: $75,000 / $21,000 = **3.6 years**

---

## 8. Performance Specifications

### 8.1 Latency Targets

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| **Glass-to-Glass Latency (P50)** | < 60ms | High-speed camera filming screen + input device |
| **Glass-to-Glass Latency (P95)** | < 80ms | Same as above, 95th percentile over 100 samples |
| **Glass-to-Glass Latency (P99)** | < 120ms | Same as above, 99th percentile |
| **Network RTT** | < 30ms | ICMP ping from client region to worker region |
| **Encoding Latency** | < 10ms | NVENC metrics from GPU worker |
| **Decoding Latency** | < 10ms | VideoDecoder stats from browser |
| **Input Processing** | < 5ms | Server timestamp - client timestamp |

#### Latency Testing Script

```javascript
// Client-side latency measurement
class LatencyTester {
  async measureGlassToGlass() {
    const results = []

    for (let i = 0; i < 100; i++) {
      // 1. Send click command
      const sendTime = performance.now()
      this.remoteClient.sendInput({
        type: 'test:click',
        x: 960,
        y: 540,
        timestamp: sendTime
      })

      // 2. Wait for visual response (pixel change detection)
      const responseTime = await this.detectPixelChange(960, 540)

      // 3. Calculate latency
      const latency = responseTime - sendTime
      results.push(latency)

      await sleep(1000) // Wait 1s between tests
    }

    return {
      p50: percentile(results, 0.5),
      p95: percentile(results, 0.95),
      p99: percentile(results, 0.99),
      avg: average(results),
      min: Math.min(...results),
      max: Math.max(...results)
    }
  }

  async detectPixelChange(x, y) {
    const canvas = document.createElement('canvas')
    canvas.width = 1
    canvas.height = 1
    const ctx = canvas.getContext('2d')

    const videoElement = document.querySelector('video')
    const initialColor = this.getPixelColor(videoElement, x, y, ctx)

    return new Promise(resolve => {
      const checkFrame = () => {
        const currentColor = this.getPixelColor(videoElement, x, y, ctx)
        if (currentColor !== initialColor) {
          resolve(performance.now())
        } else {
          requestAnimationFrame(checkFrame)
        }
      }
      requestAnimationFrame(checkFrame)
    })
  }

  getPixelColor(video, x, y, ctx) {
    ctx.drawImage(video, x, y, 1, 1, 0, 0, 1, 1)
    const pixel = ctx.getImageData(0, 0, 1, 1).data
    return `${pixel[0]},${pixel[1]},${pixel[2]}`
  }
}
```

### 8.2 Throughput Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **Concurrent Sessions per G5.xlarge** | 1-2 | 1 for high quality, 2 for medium |
| **Sessions per G5.2xlarge** | 2-4 | More vCPU for encoding overhead |
| **Max Concurrent Users (System)** | 1,000+ | With 500 workers (auto-scaled) |
| **New Session Provisioning Time** | < 10s | With warm pool |
| **Cold Start (No Warm Pool)** | < 60s | EC2 instance boot time |

### 8.3 Video Quality Targets

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Frame Rate (P95)** | 60 FPS | Server-side frame counter |
| **Frame Rate (P99)** | 55 FPS | Allow occasional drops |
| **Resolution** | 1920×1080 (Full HD) | Client requests, server renders |
| **Bitrate (Average)** | 8 Mbps | Encoder output stats |
| **Bitrate (Peak)** | 15 Mbps | During high motion scenes |
| **Packet Loss (P95)** | < 1% | WebRTC stats |
| **PSNR (Peak Signal-to-Noise Ratio)** | > 40 dB | Video quality metric |

### 8.4 Resource Utilization Targets

| Resource | Target | Notes |
|----------|--------|-------|
| **GPU Utilization** | 70-90% | Encoding + rendering load |
| **GPU Memory** | < 18 GB / 24 GB | Leave buffer for spikes |
| **CPU Utilization** | < 70% | Avoid CPU bottleneck on encoding |
| **Network Bandwidth (Per Session)** | 5-15 Mbps | Adaptive based on network |
| **Worker Memory** | < 12 GB / 16 GB | Scene data + OS + buffers |

---

## 9. Security & Compliance

### 9.1 Threat Model

#### Assets to Protect:
1. **3D Models (STL/IGS files)**: Proprietary manufacturing equipment designs
2. **Production Data**: Device positions, connections, facility layouts
3. **User Credentials**: Authentication tokens, session IDs
4. **Video Streams**: Real-time view of manufacturing workspace
5. **Session Data**: Commands, interactions, recordings

#### Threats:

| Threat | Impact | Likelihood | Mitigation |
|--------|--------|------------|------------|
| **Man-in-the-Middle Attack** | Attacker intercepts video stream | Medium | DTLS-SRTP encryption (WebRTC) |
| **Session Hijacking** | Unauthorized access to user session | Medium | JWT with short expiry, IP pinning |
| **DDoS Attack** | Service unavailable | High | CloudFront DDoS protection, rate limiting |
| **GPU Memory Leak** | Cross-session data exposure | Low | Containerization, GPU memory clearing |
| **Model Exfiltration** | Attacker downloads proprietary models | Medium | Models never sent to client, watermarking |
| **Credential Stuffing** | Brute force login | High | MFA, account lockout, CAPTCHA |
| **Insider Threat** | Malicious admin accesses sessions | Low | Audit logs, role-based access, encryption at rest |

### 9.2 Authentication & Authorization

#### JWT-Based Authentication

```typescript
// Auth flow
async function createSession(userId: string, projectId: string) {
  // 1. Verify user permissions
  const user = await db.users.findById(userId)
  const project = await db.projects.findById(projectId)

  if (!user.hasPermission(project, 'view')) {
    throw new Error('Unauthorized')
  }

  // 2. Generate JWT
  const sessionToken = jwt.sign({
    userId,
    projectId,
    role: user.role, // 'admin', 'engineer', 'viewer'
    iat: Date.now(),
    exp: Date.now() + 3600000 // 1 hour
  }, process.env.JWT_SECRET)

  // 3. Store session in Redis
  await redis.set(`session:${sessionToken}`, JSON.stringify({
    userId,
    projectId,
    createdAt: Date.now(),
    ipAddress: req.ip
  }), 'EX', 3600)

  return sessionToken
}

// Middleware to verify JWT
function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1]

  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET)
    req.user = payload
    next()
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' })
  }
}
```

#### Role-Based Access Control (RBAC)

| Role | Permissions |
|------|------------|
| **Viewer** | View scene, navigate camera, view device details |
| **Engineer** | Viewer + Select devices, create connections, move devices |
| **Admin** | Engineer + Add/delete devices, manage users, view recordings |

### 9.3 Encryption

#### Data in Transit

- **WebRTC Media**: DTLS-SRTP (enabled by default in WebRTC)
  - DTLS 1.2 for key exchange
  - SRTP (Secure Real-time Transport Protocol) for media
  - 128-bit AES-GCM encryption

- **Signaling**: WSS (WebSocket Secure) with TLS 1.3
  - Certificate: AWS Certificate Manager (ACM)
  - Cipher suites: TLS_AES_128_GCM_SHA256, TLS_AES_256_GCM_SHA384

- **API Calls**: HTTPS with TLS 1.3

#### Data at Rest

- **S3 (Models, Recordings)**: AES-256 server-side encryption (SSE-S3)
- **RDS (Database)**: AES-256 encryption at rest
- **EBS (Worker Volumes)**: AES-256 encryption
- **Redis**: No encryption (session data is ephemeral, stored in VPC)

### 9.4 Network Security

#### VPC Architecture

```
Internet → CloudFront (DDoS protection)
         → ALB (Public subnet, TLS termination)
         → Signaling Server (Private subnet)
         → GPU Workers (Private subnet, no public IP)

GPU Workers → NAT Gateway → Internet (for S3/API access)
```

#### Security Groups

**Signaling Server**:
- Inbound: 8080/tcp from ALB only
- Outbound: All (to Redis, RDS, GPU workers)

**GPU Workers**:
- Inbound: 8080/tcp from signaling server, 49152-65535/udp from Internet (WebRTC)
- Outbound: All

**Redis**:
- Inbound: 6379/tcp from signaling server, orchestrator only
- Outbound: None

**RDS**:
- Inbound: 5432/tcp from signaling server, orchestrator only
- Outbound: None

#### DDoS Protection

- **CloudFront**: AWS Shield Standard (automatic, free)
- **AWS WAF**: Rate limiting (100 requests/min per IP)
- **Auto-Scaling**: Absorb traffic spikes
- **Optional**: AWS Shield Advanced ($3,000/month) for 24/7 DDoS response team

### 9.5 Session Isolation

#### Docker Containers

Each GPU worker runs multiple sessions in isolated Docker containers:

```dockerfile
# Dockerfile for session container
FROM nvidia/cuda:12.2.0-runtime-ubuntu22.04

# Install Node.js, GStreamer, etc.
RUN apt-get update && apt-get install -y \
    nodejs npm gstreamer1.0-tools gstreamer1.0-plugins-bad \
    gstreamer1.0-plugins-good gstreamer1.0-plugins-ugly

# Copy worker application
COPY ./gpu-worker /app
WORKDIR /app
RUN npm install

# Non-root user
RUN useradd -m -s /bin/bash sessionuser
USER sessionuser

# Limit resources
# (CPU/memory limits set by Docker run command)

CMD ["node", "index.js"]
```

**Resource Limits**:
```bash
docker run \
  --gpus '"device=0"' \
  --cpus="2" \
  --memory="4g" \
  --network="isolated_session_network" \
  --name="session_abc123" \
  gpu-worker:latest
```

#### GPU Memory Isolation

- **NVIDIA MIG (Multi-Instance GPU)**: Partition A100 GPU into isolated instances
  - Not available on G5 (A10G), but supported on P4d (A100)
- **Manual Cleanup**: Zero GPU memory on session end
  ```python
  # In GPU worker shutdown hook
  import torch
  torch.cuda.empty_cache()
  # Also clear any lingering buffers
  ```

### 9.6 Audit Logging

#### Events to Log

| Event | Data Logged | Retention |
|-------|-------------|-----------|
| **Session Start** | userId, projectId, workerId, timestamp, IP | 90 days |
| **Session End** | sessionId, duration, data transferred | 90 days |
| **Device Action** | userId, action (move/select/delete), deviceId, timestamp | 30 days |
| **Connection Created** | userId, sourceDeviceId, targetDeviceId, timestamp | 30 days |
| **Authentication** | userId, success/failure, IP, timestamp | 90 days |
| **Admin Action** | adminId, action, targetResource, timestamp | 1 year |

#### Audit Log Format (JSON)

```json
{
  "timestamp": "2025-01-15T14:30:00.000Z",
  "eventType": "session.start",
  "userId": "user_abc123",
  "projectId": "proj_xyz789",
  "workerId": "i-0a1b2c3d4e5f6g7h8",
  "ipAddress": "41.34.56.78",
  "userAgent": "Mozilla/5.0...",
  "sessionId": "sess_123abc",
  "metadata": {
    "workerRegion": "me-south-1",
    "deviceType": "tablet"
  }
}
```

#### Compliance

- **GDPR**: User data deletion on request (right to be forgotten)
  - API endpoint: `DELETE /users/:id/data`
  - Deletes: account, session logs, recordings
- **SOC 2 Type II**: Audit logs, encryption, access controls
  - Annual third-party audit required for enterprise customers

---

## 10. Monitoring & Operations

### 10.1 Metrics Collection

#### Client Metrics

Collected via browser and sent to backend:

```typescript
class ClientMetrics {
  private metrics: any = {}

  collectWebRTCStats() {
    setInterval(async () => {
      const stats = await this.peerConnection.getStats()

      stats.forEach(report => {
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
          this.metrics.videoStats = {
            framesReceived: report.framesReceived,
            framesDropped: report.framesDropped,
            framesDecoded: report.framesDecoded,
            bytesReceived: report.bytesReceived,
            jitter: report.jitter,
            packetsLost: report.packetsLost,
            fps: report.framesPerSecond || 0
          }
        }

        if (report.type === 'candidate-pair' && report.state === 'succeeded') {
          this.metrics.networkStats = {
            rtt: report.currentRoundTripTime * 1000, // Convert to ms
            availableBandwidth: report.availableOutgoingBitrate / 1000000 // Convert to Mbps
          }
        }
      })

      // Send to backend
      await fetch('/api/metrics', {
        method: 'POST',
        body: JSON.stringify({
          sessionId: this.sessionId,
          timestamp: Date.now(),
          ...this.metrics
        })
      })
    }, 5000) // Every 5 seconds
  }
}
```

#### Server Metrics

Collected from GPU workers:

```typescript
// gpu-worker/metrics.ts
import { Registry, Gauge, Histogram } from 'prom-client'

const register = new Registry()

const gpuUtilization = new Gauge({
  name: 'gpu_utilization_percent',
  help: 'GPU utilization percentage',
  registers: [register]
})

const gpuMemoryUsed = new Gauge({
  name: 'gpu_memory_used_bytes',
  help: 'GPU memory used in bytes',
  registers: [register]
})

const encodingLatency = new Histogram({
  name: 'encoding_latency_ms',
  help: 'NVENC encoding latency in milliseconds',
  buckets: [1, 2, 5, 10, 20, 50],
  registers: [register]
})

const renderingLatency = new Histogram({
  name: 'rendering_latency_ms',
  help: 'Frame rendering latency in milliseconds',
  buckets: [5, 10, 16, 20, 33, 50],
  registers: [register]
})

// Collect NVIDIA GPU metrics
setInterval(() => {
  exec('nvidia-smi --query-gpu=utilization.gpu,memory.used --format=csv,noheader,nounits', (err, stdout) => {
    if (err) return
    const [util, mem] = stdout.trim().split(', ')
    gpuUtilization.set(parseFloat(util))
    gpuMemoryUsed.set(parseFloat(mem) * 1024 * 1024) // Convert MB to bytes
  })
}, 5000)

// Export metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.end(await register.metrics())
})
```

### 10.2 Dashboards

#### Grafana Dashboard: Real-Time Sessions

**Panels**:

1. **Active Sessions (Time Series)**
   - Query: `count(remote_render_sessions{status="active"})`
   - Visualization: Line graph, 5-minute window

2. **Session Map (World Map)**
   - Query: `remote_render_sessions{status="active"}`
   - Group by: `client_country`
   - Visualization: Geomap with circle markers

3. **Latency Heatmap (Heatmap)**
   - Query: `histogram_quantile(0.95, rate(glass_to_glass_latency_ms_bucket[5m]))`
   - Group by: `worker_region`, `client_region`
   - Visualization: Heatmap (X=worker region, Y=client region, color=latency)

4. **GPU Utilization (Gauge)**
   - Query: `avg(gpu_utilization_percent{job="gpu-worker"})`
   - Visualization: Gauge (0-100%)

5. **Bandwidth Usage (Graph)**
   - Query: `sum(rate(video_bytes_sent[5m])) * 8 / 1000000`
   - Visualization: Line graph, unit=Mbps

6. **Frame Rate (Graph)**
   - Query: `avg(video_fps{job="gpu-worker"})`
   - Target line: 60 FPS
   - Visualization: Line graph with threshold

#### Grafana Dashboard: Cost Tracking

**Panels**:

1. **Hourly GPU Cost (Time Series)**
   - Query: `count(gpu_workers{type="g5.xlarge"}) * 1.00`
   - Visualization: Stacked area (on-demand vs. spot)

2. **Monthly Projected Cost (Stat)**
   - Query: `sum_over_time(hourly_cost[30d]) / 30 * 30`
   - Visualization: Single stat, large number

3. **Cost per User (Table)**
   - Query: `sum(hourly_cost) / count(active_sessions)`
   - Group by: hour
   - Visualization: Table

### 10.3 Alerting

#### Alert Rules (Prometheus Alertmanager)

```yaml
# alerts.yml
groups:
  - name: remote_render_alerts
    interval: 30s
    rules:
      # P0: Critical
      - alert: OrchestratorDown
        expr: up{job="orchestrator"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Orchestrator is down"
          description: "No new sessions can be created"

      - alert: AllWorkersDown
        expr: count(up{job="gpu-worker"} == 1) == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "All GPU workers are down"
          description: "No active sessions possible"

      # P1: High Priority
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(glass_to_glass_latency_ms_bucket[5m])) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 latency exceeds 100ms"
          description: "Current: {{ $value }}ms"

      - alert: HighGPUUtilization
        expr: avg(gpu_utilization_percent{job="gpu-worker"}) > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Average GPU utilization is {{ $value }}%"
          description: "Consider scaling up workers"

      - alert: HighPacketLoss
        expr: avg(packet_loss_percent{job="gpu-worker"}) > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "Packet loss is {{ $value }}%"
          description: "Network quality degraded"

      # P2: Medium Priority
      - alert: SlowWorkerProvisioning
        expr: histogram_quantile(0.95, rate(worker_provision_time_seconds_bucket[10m])) > 90
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "Worker provisioning is slow"
          description: "P95 provisioning time: {{ $value }}s"

      - alert: LowWarmPoolSize
        expr: count(gpu_workers{status="idle"}) < 2
        for: 5m
        labels:
          severity: info
        annotations:
          summary: "Warm pool size is low"
          description: "Only {{ $value }} workers idle"
```

#### Alert Routing (Alertmanager)

```yaml
# alertmanager.yml
route:
  receiver: default
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    # Critical alerts → PagerDuty (24/7 on-call)
    - match:
        severity: critical
      receiver: pagerduty
      continue: true

    # Warnings → Slack channel
    - match:
        severity: warning
      receiver: slack

    # Info → Email
    - match:
        severity: info
      receiver: email

receivers:
  - name: pagerduty
    pagerduty_configs:
      - service_key: '<pagerduty_service_key>'

  - name: slack
    slack_configs:
      - api_url: '<slack_webhook_url>'
        channel: '#remote-render-alerts'
        title: 'Remote Render Alert'
        text: '{{ .CommonAnnotations.description }}'

  - name: email
    email_configs:
      - to: 'ops@example.com'
        from: 'alerts@example.com'
        smarthost: 'smtp.example.com:587'
        auth_username: 'alerts@example.com'
        auth_password: '<password>'
```

### 10.4 Log Aggregation

#### ELK Stack (Elasticsearch, Logstash, Kibana)

**Architecture**:
```
GPU Workers → CloudWatch Logs → Lambda (forward to Elasticsearch) → Elasticsearch
Signaling Server → CloudWatch Logs → Lambda → Elasticsearch
Orchestrator → CloudWatch Logs → Lambda → Elasticsearch

Kibana → Query/Visualize Elasticsearch
```

**Example Kibana Query**:

```
# Find all sessions with high latency
fields @timestamp, sessionId, userId, latency_ms
| filter latency_ms > 100
| sort @timestamp desc
| limit 100
```

### 10.5 Distributed Tracing

#### OpenTelemetry + Jaeger

**Instrumentation**:

```typescript
// signaling-server.ts
import { trace, context } from '@opentelemetry/api'
import { JaegerExporter } from '@opentelemetry/exporter-jaeger'
import { NodeTracerProvider } from '@opentelemetry/sdk-trace-node'
import { registerInstrumentations } from '@opentelemetry/instrumentation'
import { ExpressInstrumentation } from '@opentelemetry/instrumentation-express'

const provider = new NodeTracerProvider()
provider.addSpanProcessor(new BatchSpanProcessor(new JaegerExporter({
  endpoint: 'http://jaeger:14268/api/traces'
})))
provider.register()

registerInstrumentations({
  instrumentations: [new ExpressInstrumentation()]
})

const tracer = trace.getTracer('signaling-server')

// Example traced function
async function handleSessionRequest(req, res) {
  const span = tracer.startSpan('handleSessionRequest')

  try {
    span.setAttribute('userId', req.body.userId)
    span.setAttribute('projectId', req.body.projectId)

    const workerId = await allocateWorker(req.body.userId)
    span.addEvent('worker_allocated', { workerId })

    res.json({ workerId })
  } catch (err) {
    span.recordException(err)
    span.setStatus({ code: SpanStatusCode.ERROR })
    res.status(500).json({ error: err.message })
  } finally {
    span.end()
  }
}
```

**Jaeger UI**: View end-to-end trace from client request → signaling → orchestrator → worker → response

### 10.6 Runbooks

#### Runbook: High Latency

**Symptoms**: P95 latency > 100ms for > 5 minutes

**Investigation**:
1. Check Grafana dashboard for latency by region
2. If specific region affected:
   - Check AWS Service Health Dashboard for outages
   - Run `ping` and `traceroute` from test client in that region
3. If all regions affected:
   - Check GPU worker metrics: High CPU/GPU utilization?
   - Check encoding latency: NVENC overloaded?
   - Check network bandwidth: Throttling?

**Resolution**:
- If GPU overloaded: Scale up workers
- If network congestion: Reduce bitrate (update encoder config)
- If regional issue: Redirect users to alternate region

**Prevention**:
- Set up auto-scaling to trigger earlier (at 70% GPU utilization instead of 80%)
- Enable multi-region failover

#### Runbook: Worker Provisioning Slow

**Symptoms**: P95 provisioning time > 90 seconds

**Investigation**:
1. Check CloudWatch Logs for orchestrator errors
2. Check EC2 Auto Scaling activity: Scaling up?
3. Check AMI availability: Is it cached in region?

**Resolution**:
- If ASG limit reached: Increase max instances
- If AMI slow: Pre-warm AMI cache (launch & terminate instance periodically)
- If boot slow: Optimize UserData script (reduce package installs)

**Prevention**:
- Maintain larger warm pool (5 instances instead of 2)
- Use pre-built AMIs with all dependencies

---

## 11. Risk Assessment & Mitigation

### 11.1 Technical Risks

| Risk | Impact | Likelihood | Mitigation Strategy |
|------|--------|------------|---------------------|
| **WebRTC Browser Incompatibility** | High | Low | Test on Chrome, Firefox, Safari, Edge; Provide fallback (HLS streaming) |
| **NVENC Encoding Bottleneck** | High | Medium | Multi-session per GPU; Use H.264 instead of H.265; Reduce resolution |
| **Network Congestion (Egypt ISPs)** | High | Medium | Adaptive bitrate; Multi-CDN; Test with major ISPs |
| **GPU Driver Crash** | Medium | Low | Automated restart; Health checks; Watchdog process |
| **WebRTC NAT Traversal Failure** | Medium | Medium | TURN server as fallback; Test with corporate firewalls |
| **S3/CloudFront Outage** | Medium | Very Low | Multi-region S3 replication; Local cache |

### 11.2 Operational Risks

| Risk | Impact | Likelihood | Mitigation Strategy |
|------|--------|------------|---------------------|
| **Unexpected Cost Spike** | High | Medium | Billing alerts; Cost caps; Spot instance limits |
| **Key Personnel Departure** | High | Low | Documentation; Cross-training; Managed services where possible |
| **AWS Region Outage** | High | Very Low | Multi-region deployment; Automated failover |
| **DDoS Attack** | Medium | Medium | CloudFront + AWS Shield; Rate limiting; WAF rules |
| **Data Breach (Session Hijacking)** | High | Low | Short-lived JWTs; IP pinning; Audit logs |

### 11.3 Business Risks

| Risk | Impact | Likelihood | Mitigation Strategy |
|------|--------|------------|---------------------|
| **User Adoption Resistance** | High | Medium | Pilot program with early adopters; Training; Gradual rollout |
| **Performance Expectations Not Met** | High | Low | PoC phase to validate; Clear SLAs; Fallback to client rendering |
| **Budget Overrun** | Medium | Medium | Phased approach; Cost monitoring; Optimizations (spot, multi-session) |
| **Vendor Lock-In (AWS)** | Medium | Low | Abstract infrastructure (Terraform); Multi-cloud strategy (Azure backup) |
| **Regulatory Compliance (GDPR)** | High | Low | Legal review; Data deletion API; EU region option |

### 11.4 Mitigation Plan: Multi-Region Failover

**Architecture**:
```
Primary Region: me-south-1 (Bahrain)
Secondary Region: eu-central-1 (Frankfurt)

Route 53 Health Checks → Failover to Secondary
```

**Setup**:

1. **Duplicate Infrastructure in Secondary Region**
   - Deploy full stack (signaling, orchestrator, workers) to eu-central-1
   - Use Terraform with region parameter

2. **Cross-Region Data Replication**
   - RDS: Read replica in Frankfurt
   - S3: Cross-region replication (CRR)
   - Redis: Manual failover (acceptable for session data)

3. **DNS Failover**
   ```
   Route 53 Health Check:
   - Primary: https://me-south-1.api.example.com/health
   - If unhealthy for 3 consecutive checks (30s):
     → Failover to eu-central-1.api.example.com
   ```

4. **Testing**
   - Quarterly DR drill: Shut down primary region, verify failover
   - Measure RTO (Recovery Time Objective): Target < 5 minutes
   - Measure RPO (Recovery Point Objective): Target < 1 minute (data loss)

---

## 12. Similar Projects & Case Studies

### 12.1 Industry Examples

#### 1. **Volkswagen + AWS NICE DCV**

**Overview**:
- 1,000+ automotive engineers access CAD workstations remotely
- Used for 3D simulations and computational fluid dynamics
- Deployed on AWS G4dn instances

**Results**:
- Engineers work from home with same performance as office
- Reduced hardware costs (no need for individual workstations)
- Improved collaboration (shared sessions)

**Lessons Learned**:
- Importance of low latency for CAD work (< 50ms)
- Network bandwidth critical (target 10+ Mbps)
- User training necessary for adoption

**Reference**: [AWS Blog: Enabling High Performance CAD](https://aws.amazon.com/blogs/industries/enabling-hp-cad-for-remote-workers/)

---

#### 2. **Microsoft 3D Streaming Toolkit**

**Overview**:
- Open-source toolkit for WebRTC-based 3D streaming
- Uses NVENC for encoding, WebRTC for transport
- Demonstrated with HoloLens AR streaming

**Architecture**:
- C++ server (DirectX rendering)
- WebRTC DataChannel for input
- H.264 Low-Latency profile
- Sub-50ms latency achieved

**Key Features**:
- Spatial audio support
- Multi-user collaboration
- VR/AR support (stereoscopic rendering)

**Lessons Learned**:
- DataChannel unordered mode essential for low latency
- GOP size tuning critical (1-2 seconds optimal)
- Client-side prediction reduces perceived latency

**Reference**: [3D Streaming Toolkit Documentation](https://3dstreamingtoolkit.github.io/docs-3dstk/)

---

#### 3. **Unreal Engine Pixel Streaming (Epic Games)**

**Overview**:
- Built-in feature for streaming Unreal Engine applications
- Used for automotive configurators (BMW, Audi)
- Deployed on AWS, Azure, GCP

**Technical Details**:
- Cirrus signaling server (Node.js)
- Matchmaker for session routing
- NVENC encoding with WebRTC
- Custom UE plugin handles input

**Performance**:
- 4K60 at 20-40 Mbps
- Sub-50ms latency on good networks
- Scales to 10,000+ concurrent users (BMW configurator)

**Lessons Learned**:
- Multi-session per GPU feasible (2-4 sessions on G5.2xlarge)
- Warm pool critical for instant session start
- Regional deployment needed for global audience

**Reference**: [Unreal Pixel Streaming Documentation](https://docs.unrealengine.com/5.2/en-US/overview-of-pixel-streaming-in-unreal-engine/)

---

#### 4. **Azure Remote Rendering (Microsoft)**

**Overview**:
- Managed service for remote 3D rendering
- Target: HoloLens AR, mixed reality
- GPU rendering on Azure NV-series (discontinued Sept 2025)

**Features**:
- Automatic LOD management
- Dynamic mesh simplification
- Occlusion culling server-side
- Collaborative multi-user sessions

**Pricing** (before retirement):
- $1.50/hour for standard VM
- Data transfer: $0.05/GB

**Why Retired?**:
- Limited adoption (niche use case)
- High operational cost
- Competition from Unreal Pixel Streaming

**Lessons Learned**:
- Niche products need strong ecosystem
- Managed services must be cost-competitive
- Open-source alternatives can disrupt

**Reference**: [Azure Remote Rendering](https://azure.microsoft.com/en-us/products/remote-rendering)

---

#### 5. **Unity Render Streaming**

**Overview**:
- Unity's solution for WebRTC streaming
- Similar to Unreal Pixel Streaming
- Used for automotive, architecture, retail

**Technical Details**:
- Unity C# server
- WebRTC C++ library (libwebrtc)
- NVENC encoding
- Cross-platform (Windows, Linux)

**Use Case Example**: **Toyota Showroom**
- Virtual car configurator
- 3D customization (color, wheels, interior)
- Deployed on AWS G4dn
- Handles 500+ concurrent users

**Lessons Learned**:
- Unity easier to adopt than Unreal (more developers)
- Performance on par with Unreal
- Asset optimization still critical (LOD, compression)

**Reference**: [Unity Render Streaming](https://docs.unity3d.com/Packages/com.unity.renderstreaming@3.1/manual/index.html)

---

### 12.2 Comparison Matrix

| Project | Technology | Target Use Case | Latency | Cost (Est.) | Open Source |
|---------|-----------|-----------------|---------|-------------|-------------|
| **Volkswagen CAD** | AWS NICE DCV | CAD/Engineering | 40-60ms | $1-2/user/hr | No (AWS) |
| **3D Streaming Toolkit** | Custom C++ + WebRTC | AR/VR/3D Apps | 30-50ms | Self-hosted | Yes |
| **Unreal Pixel Streaming** | Unreal Engine + WebRTC | Games/Configurators | 30-50ms | $1-2/user/hr | Partial (UE license) |
| **Azure Remote Rendering** | Azure Managed Service | HoloLens AR | 50-80ms | $1.50/hr | No (Azure) |
| **Unity Render Streaming** | Unity + WebRTC | Games/Automotive | 40-60ms | $1-2/user/hr | Partial (Unity license) |
| **Our Solution** | React Three Fiber + WebRTC | Manufacturing 3D | 40-60ms | $0.17-1.21/user/hr | Custom |

### 12.3 Key Takeaways

1. **Latency is Critical**: Sub-50ms requires optimal region selection and NVENC low-latency preset
2. **Multi-Session is Viable**: 2-4 sessions per GPU reduces cost by 50-75%
3. **Warm Pool is Essential**: Cold start (60s) hurts UX; maintain 2-5 idle instances
4. **WebRTC is Mature**: Browser support excellent, ecosystem strong (libraries, tools)
5. **Cost Optimization is Key**: Spot instances + multi-session + regional pricing → 85% savings
6. **Hybrid Mode Best**: Let high-end devices render locally, force low-end to remote

---

## 13. Appendices

### 13.1 Glossary

| Term | Definition |
|------|------------|
| **WebRTC** | Web Real-Time Communication; browser API for peer-to-peer video/audio streaming |
| **DataChannel** | WebRTC feature for sending arbitrary data (JSON, binary) between peers |
| **SDP (Session Description Protocol)** | Format for describing WebRTC session parameters (codecs, bitrate, etc.) |
| **ICE (Interactive Connectivity Establishment)** | Protocol for NAT traversal in WebRTC |
| **STUN (Session Traversal Utilities for NAT)** | Server that helps discover public IP for NAT traversal |
| **TURN (Traversal Using Relays around NAT)** | Relay server for WebRTC when direct connection fails |
| **NVENC** | NVIDIA GPU hardware video encoder (H.264/H.265) |
| **DTLS-SRTP** | Encryption protocol for WebRTC media streams |
| **GOP (Group of Pictures)** | Sequence of video frames between keyframes |
| **Keyframe** | Full video frame (not delta); used for seeking and error recovery |
| **Bitrate** | Video encoding quality, measured in Mbps (megabits per second) |
| **RTT (Round-Trip Time)** | Network latency from client to server and back |
| **Glass-to-Glass Latency** | Total time from user input to visual response on screen |
| **LOD (Level of Detail)** | Technique to render simpler models at distance |
| **Frustum Culling** | Don't render objects outside camera view |
| **Orchestrator** | Service that manages GPU worker pool and assigns sessions |
| **Warm Pool** | Pre-provisioned instances ready for instant session start |
| **Spot Instance** | AWS discounted instance (70% off) that can be interrupted |

### 13.2 References

1. **WebRTC Specification**: https://www.w3.org/TR/webrtc/
2. **NVENC Programming Guide**: https://docs.nvidia.com/video-technologies/video-codec-sdk/nvenc-video-encoder-api-prog-guide/
3. **AWS G5 Instances**: https://aws.amazon.com/ec2/instance-types/g5/
4. **Unreal Pixel Streaming**: https://docs.unrealengine.com/5.2/en-US/overview-of-pixel-streaming-in-unreal-engine/
5. **GStreamer WebRTC**: https://gstreamer.freedesktop.org/documentation/webrtc/index.html
6. **Three.js Manual**: https://threejs.org/docs/
7. **React Three Fiber**: https://docs.pmnd.rs/react-three-fiber/
8. **Terraform AWS Provider**: https://registry.terraform.io/providers/hashicorp/aws/latest/docs

### 13.3 Contact & Support

**Technical Questions**:
- Email: engineering@example.com
- Slack: #remote-render-project

**Project Management**:
- Project Manager: [Name]
- Email: pm@example.com

**Vendor Support**:
- AWS Support: https://console.aws.amazon.com/support/
- NVIDIA Developer Forums: https://forums.developer.nvidia.com/

---

## Document Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-01-15 | Technical Team | Initial comprehensive plan |

---

## Next Steps

1. **Executive Review**: Present this plan to stakeholders for approval
2. **Budget Approval**: Secure funding for Phase 0 PoC ($5,000-$10,000)
3. **Team Formation**: Assign engineers to project (2-3 full-time)
4. **Phase 0 Kickoff**: Start PoC with AWS NICE DCV (Week 1)

---

**End of Document**
