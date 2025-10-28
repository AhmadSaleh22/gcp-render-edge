# Remote Rendering Implementation - Code Examples

This document provides complete, production-ready code examples for implementing the Remote Rendering + WebRTC solution.

---

## Table of Contents

1. [Client Implementation](#1-client-implementation)
2. [Signaling Server](#2-signaling-server)
3. [GPU Worker](#3-gpu-worker)
4. [Orchestrator](#4-orchestrator)
5. [Infrastructure as Code](#5-infrastructure-as-code)
6. [Testing & Monitoring](#6-testing--monitoring)

---

## 1. Client Implementation

### 1.1 WebRTC Client Class

```typescript
// src/lib/remote-render-client.ts

export interface RemoteRenderConfig {
  signalingServerUrl: string
  stunServers?: string[]
  turnServers?: Array<{
    urls: string
    username?: string
    credential?: string
  }>
  videoElement: HTMLVideoElement
  onConnected?: () => void
  onDisconnected?: () => void
  onError?: (error: Error) => void
  onStats?: (stats: RTCStats) => void
}

export interface InputCommand {
  type: string
  timestamp: number
  data: any
}

export interface RTCStats {
  fps: number
  rtt: number
  packetsLost: number
  bytesReceived: number
  jitter: number
  bitrate: number
}

export class RemoteRenderClient {
  private config: RemoteRenderConfig
  private peerConnection: RTCPeerConnection | null = null
  private dataChannel: RTCDataChannel | null = null
  private websocket: WebSocket | null = null
  private sessionId: string | null = null
  private statsInterval: NodeJS.Timeout | null = null
  private connected: boolean = false

  constructor(config: RemoteRenderConfig) {
    this.config = config
  }

  async connect(userId: string, projectId: string): Promise<void> {
    try {
      // 1. Connect to signaling server
      await this.connectSignaling(userId, projectId)

      // 2. Create peer connection
      this.createPeerConnection()

      // 3. Create data channel for input
      this.createDataChannel()

      // 4. Create and send offer
      await this.createAndSendOffer()

      // 5. Start stats collection
      this.startStatsCollection()

    } catch (error) {
      this.config.onError?.(error as Error)
      throw error
    }
  }

  private async connectSignaling(userId: string, projectId: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const ws = new WebSocket(
        `${this.config.signalingServerUrl}?userId=${userId}&projectId=${projectId}`
      )

      ws.onopen = () => {
        console.log('[RemoteRenderClient] Signaling connected')
        this.websocket = ws
        resolve()
      }

      ws.onerror = (error) => {
        console.error('[RemoteRenderClient] Signaling error:', error)
        reject(new Error('Failed to connect to signaling server'))
      }

      ws.onmessage = async (event) => {
        await this.handleSignalingMessage(JSON.parse(event.data))
      }

      ws.onclose = () => {
        console.log('[RemoteRenderClient] Signaling disconnected')
        this.handleDisconnect()
      }
    })
  }

  private createPeerConnection(): void {
    const config: RTCConfiguration = {
      iceServers: [
        ...(this.config.stunServers?.map(url => ({ urls: url })) || [
          { urls: 'stun:stun.l.google.com:19302' },
          { urls: 'stun:stun1.l.google.com:19302' }
        ]),
        ...(this.config.turnServers || [])
      ],
      iceTransportPolicy: 'all',
      bundlePolicy: 'max-bundle',
      rtcpMuxPolicy: 'require'
    }

    this.peerConnection = new RTCPeerConnection(config)

    // Handle incoming tracks (video stream)
    this.peerConnection.ontrack = (event) => {
      console.log('[RemoteRenderClient] Received track:', event.track.kind)
      if (event.track.kind === 'video') {
        this.config.videoElement.srcObject = event.streams[0]
        this.config.videoElement.play().catch(err => {
          console.error('[RemoteRenderClient] Video play failed:', err)
        })
      }
    }

    // Handle ICE candidates
    this.peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        this.sendSignalingMessage({
          type: 'ice-candidate',
          candidate: event.candidate
        })
      }
    }

    // Handle connection state changes
    this.peerConnection.onconnectionstatechange = () => {
      const state = this.peerConnection?.connectionState
      console.log('[RemoteRenderClient] Connection state:', state)

      if (state === 'connected') {
        this.connected = true
        this.config.onConnected?.()
      } else if (state === 'disconnected' || state === 'failed' || state === 'closed') {
        this.handleDisconnect()
      }
    }

    // Handle ICE connection state
    this.peerConnection.oniceconnectionstatechange = () => {
      const state = this.peerConnection?.iceConnectionState
      console.log('[RemoteRenderClient] ICE connection state:', state)

      if (state === 'failed') {
        // Restart ICE
        this.peerConnection?.restartIce()
      }
    }
  }

  private createDataChannel(): void {
    if (!this.peerConnection) {
      throw new Error('Peer connection not initialized')
    }

    this.dataChannel = this.peerConnection.createDataChannel('input', {
      ordered: false,         // Unordered for lowest latency
      maxRetransmits: 0,      // No retransmits
      maxPacketLifeTime: 100  // Drop packets older than 100ms
    })

    this.dataChannel.onopen = () => {
      console.log('[RemoteRenderClient] Data channel opened')
    }

    this.dataChannel.onclose = () => {
      console.log('[RemoteRenderClient] Data channel closed')
    }

    this.dataChannel.onerror = (error) => {
      console.error('[RemoteRenderClient] Data channel error:', error)
    }

    this.dataChannel.onmessage = (event) => {
      // Handle messages from server (state updates, etc.)
      try {
        const message = JSON.parse(event.data)
        this.handleServerMessage(message)
      } catch (err) {
        console.error('[RemoteRenderClient] Failed to parse data channel message:', err)
      }
    }
  }

  private async createAndSendOffer(): Promise<void> {
    if (!this.peerConnection) {
      throw new Error('Peer connection not initialized')
    }

    const offer = await this.peerConnection.createOffer({
      offerToReceiveVideo: true,
      offerToReceiveAudio: false
    })

    await this.peerConnection.setLocalDescription(offer)

    this.sendSignalingMessage({
      type: 'offer',
      sdp: offer
    })
  }

  private async handleSignalingMessage(message: any): Promise<void> {
    switch (message.type) {
      case 'session-created':
        this.sessionId = message.sessionId
        console.log('[RemoteRenderClient] Session created:', this.sessionId)
        break

      case 'answer':
        if (this.peerConnection) {
          await this.peerConnection.setRemoteDescription(
            new RTCSessionDescription(message.sdp)
          )
          console.log('[RemoteRenderClient] Answer received')
        }
        break

      case 'ice-candidate':
        if (this.peerConnection && message.candidate) {
          await this.peerConnection.addIceCandidate(
            new RTCIceCandidate(message.candidate)
          )
        }
        break

      case 'error':
        console.error('[RemoteRenderClient] Signaling error:', message.error)
        this.config.onError?.(new Error(message.error))
        break
    }
  }

  private sendSignalingMessage(message: any): void {
    if (this.websocket?.readyState === WebSocket.OPEN) {
      this.websocket.send(JSON.stringify({
        ...message,
        sessionId: this.sessionId
      }))
    } else {
      console.warn('[RemoteRenderClient] Websocket not ready, message dropped')
    }
  }

  private handleServerMessage(message: any): void {
    // Handle state updates, acknowledgments, etc.
    console.log('[RemoteRenderClient] Server message:', message)
  }

  private startStatsCollection(): void {
    this.statsInterval = setInterval(async () => {
      if (!this.peerConnection || !this.connected) return

      try {
        const stats = await this.peerConnection.getStats()
        const rtcStats = this.parseStats(stats)
        this.config.onStats?.(rtcStats)
      } catch (err) {
        console.error('[RemoteRenderClient] Failed to collect stats:', err)
      }
    }, 1000) // Collect every second
  }

  private parseStats(stats: RTCStatsReport): RTCStats {
    let fps = 0
    let rtt = 0
    let packetsLost = 0
    let bytesReceived = 0
    let jitter = 0
    let bitrate = 0

    stats.forEach((report) => {
      if (report.type === 'inbound-rtp' && report.kind === 'video') {
        fps = report.framesPerSecond || 0
        packetsLost = report.packetsLost || 0
        bytesReceived = report.bytesReceived || 0
        jitter = report.jitter || 0

        // Calculate bitrate (bytes/sec to Mbps)
        const prevBytes = (report as any).prevBytesReceived || 0
        const prevTime = (report as any).prevTimestamp || report.timestamp
        const duration = (report.timestamp - prevTime) / 1000 // seconds
        if (duration > 0) {
          bitrate = ((bytesReceived - prevBytes) * 8) / duration / 1_000_000 // Mbps
        }

        // Store for next calculation
        ;(report as any).prevBytesReceived = bytesReceived
        ;(report as any).prevTimestamp = report.timestamp
      }

      if (report.type === 'candidate-pair' && report.state === 'succeeded') {
        rtt = (report.currentRoundTripTime || 0) * 1000 // Convert to ms
      }
    })

    return { fps, rtt, packetsLost, bytesReceived, jitter, bitrate }
  }

  // Public API for sending input commands

  sendCameraOrbit(deltaX: number, deltaY: number): void {
    this.sendInput({
      type: 'camera:orbit',
      timestamp: Date.now(),
      data: { deltaX, deltaY }
    })
  }

  sendCameraZoom(delta: number): void {
    this.sendInput({
      type: 'camera:zoom',
      timestamp: Date.now(),
      data: { delta }
    })
  }

  sendDeviceClick(deviceId: string, screenX: number, screenY: number): void {
    this.sendInput({
      type: 'device:click',
      timestamp: Date.now(),
      data: { deviceId, screenX, screenY }
    })
  }

  sendDeviceDrag(deviceId: string, position: [number, number, number]): void {
    this.sendInput({
      type: 'device:drag',
      timestamp: Date.now(),
      data: { deviceId, position }
    })
  }

  sendConnectionCreate(sourceDeviceId: string, targetDeviceId: string, type: string): void {
    this.sendInput({
      type: 'connection:create',
      timestamp: Date.now(),
      data: { sourceDeviceId, targetDeviceId, type }
    })
  }

  private sendInput(command: InputCommand): void {
    if (this.dataChannel?.readyState === 'open') {
      try {
        this.dataChannel.send(JSON.stringify(command))
      } catch (err) {
        console.error('[RemoteRenderClient] Failed to send input:', err)
      }
    } else {
      console.warn('[RemoteRenderClient] Data channel not ready, input dropped')
    }
  }

  private handleDisconnect(): void {
    this.connected = false
    this.config.onDisconnected?.()

    if (this.statsInterval) {
      clearInterval(this.statsInterval)
      this.statsInterval = null
    }
  }

  disconnect(): void {
    console.log('[RemoteRenderClient] Disconnecting...')

    if (this.dataChannel) {
      this.dataChannel.close()
      this.dataChannel = null
    }

    if (this.peerConnection) {
      this.peerConnection.close()
      this.peerConnection = null
    }

    if (this.websocket) {
      this.websocket.close()
      this.websocket = null
    }

    this.handleDisconnect()
  }
}
```

### 1.2 React Hook for Remote Rendering

```typescript
// src/hooks/use-remote-render.ts

import { useEffect, useRef, useState, useCallback } from 'react'
import { RemoteRenderClient, RTCStats } from '@/lib/remote-render-client'

export interface UseRemoteRenderOptions {
  signalingServerUrl: string
  userId: string
  projectId: string
  autoConnect?: boolean
  onError?: (error: Error) => void
}

export function useRemoteRender(options: UseRemoteRenderOptions) {
  const [connected, setConnected] = useState(false)
  const [stats, setStats] = useState<RTCStats | null>(null)
  const [error, setError] = useState<Error | null>(null)
  const [loading, setLoading] = useState(false)

  const videoRef = useRef<HTMLVideoElement>(null)
  const clientRef = useRef<RemoteRenderClient | null>(null)

  const connect = useCallback(async () => {
    if (!videoRef.current) {
      const err = new Error('Video element not ready')
      setError(err)
      options.onError?.(err)
      return
    }

    setLoading(true)
    setError(null)

    try {
      const client = new RemoteRenderClient({
        signalingServerUrl: options.signalingServerUrl,
        videoElement: videoRef.current,
        onConnected: () => {
          console.log('[useRemoteRender] Connected')
          setConnected(true)
          setLoading(false)
        },
        onDisconnected: () => {
          console.log('[useRemoteRender] Disconnected')
          setConnected(false)
        },
        onError: (err) => {
          console.error('[useRemoteRender] Error:', err)
          setError(err)
          setLoading(false)
          options.onError?.(err)
        },
        onStats: (rtcStats) => {
          setStats(rtcStats)
        }
      })

      await client.connect(options.userId, options.projectId)
      clientRef.current = client
    } catch (err) {
      const error = err as Error
      setError(error)
      setLoading(false)
      options.onError?.(error)
    }
  }, [options.signalingServerUrl, options.userId, options.projectId])

  const disconnect = useCallback(() => {
    clientRef.current?.disconnect()
    clientRef.current = null
    setConnected(false)
  }, [])

  // Auto-connect on mount if enabled
  useEffect(() => {
    if (options.autoConnect) {
      connect()
    }

    return () => {
      disconnect()
    }
  }, [options.autoConnect, connect, disconnect])

  // Input command helpers
  const sendCameraOrbit = useCallback((deltaX: number, deltaY: number) => {
    clientRef.current?.sendCameraOrbit(deltaX, deltaY)
  }, [])

  const sendCameraZoom = useCallback((delta: number) => {
    clientRef.current?.sendCameraZoom(delta)
  }, [])

  const sendDeviceClick = useCallback((deviceId: string, screenX: number, screenY: number) => {
    clientRef.current?.sendDeviceClick(deviceId, screenX, screenY)
  }, [])

  const sendDeviceDrag = useCallback((deviceId: string, position: [number, number, number]) => {
    clientRef.current?.sendDeviceDrag(deviceId, position)
  }, [])

  const sendConnectionCreate = useCallback((sourceDeviceId: string, targetDeviceId: string, type: string) => {
    clientRef.current?.sendConnectionCreate(sourceDeviceId, targetDeviceId, type)
  }, [])

  return {
    videoRef,
    connected,
    loading,
    error,
    stats,
    connect,
    disconnect,
    sendCameraOrbit,
    sendCameraZoom,
    sendDeviceClick,
    sendDeviceDrag,
    sendConnectionCreate
  }
}
```

### 1.3 React Component Integration

```typescript
// src/pages/remote-manufacturing-workspace.tsx

import { useRemoteRender } from '@/hooks/use-remote-render'
import { Card } from '@/components/ui/card'
import { Button } from '@/components/ui/button'
import { Badge } from '@/components/ui/badge'
import { Loader2 } from 'lucide-react'

export default function RemoteManufacturingWorkspace() {
  const userId = 'user_123' // From auth context
  const projectId = 'project_abc' // From route params

  const {
    videoRef,
    connected,
    loading,
    error,
    stats,
    connect,
    disconnect,
    sendCameraOrbit,
    sendCameraZoom,
    sendDeviceClick
  } = useRemoteRender({
    signalingServerUrl: process.env.NEXT_PUBLIC_SIGNALING_URL!,
    userId,
    projectId,
    autoConnect: true,
    onError: (err) => {
      console.error('Remote render error:', err)
      // Show toast notification
    }
  })

  const handleMouseMove = (e: React.MouseEvent<HTMLVideoElement>) => {
    if (e.buttons === 1) { // Left mouse button
      const deltaX = e.movementX
      const deltaY = e.movementY
      sendCameraOrbit(deltaX, deltaY)
    }
  }

  const handleWheel = (e: React.WheelEvent<HTMLVideoElement>) => {
    e.preventDefault()
    sendCameraZoom(e.deltaY > 0 ? -1 : 1)
  }

  const handleClick = (e: React.MouseEvent<HTMLVideoElement>) => {
    const rect = e.currentTarget.getBoundingClientRect()
    const x = e.clientX - rect.left
    const y = e.clientY - rect.top
    // Device ID would be determined by server raycast
    sendDeviceClick('', x, y)
  }

  return (
    <div className="relative w-full h-screen bg-background">
      {/* Header */}
      <div className="absolute top-0 left-0 right-0 z-10 bg-background/95 backdrop-blur-sm border-b p-4">
        <div className="flex items-center justify-between">
          <div>
            <h1 className="text-2xl font-bold">Remote Manufacturing Workspace</h1>
            <p className="text-sm text-muted-foreground">
              Cloud-powered 3D visualization
            </p>
          </div>

          <div className="flex gap-2 items-center">
            <Badge variant={connected ? 'default' : 'secondary'}>
              {connected ? 'Connected' : 'Disconnected'}
            </Badge>

            {connected ? (
              <Button size="sm" variant="outline" onClick={disconnect}>
                Disconnect
              </Button>
            ) : (
              <Button size="sm" onClick={connect} disabled={loading}>
                {loading && <Loader2 className="w-4 h-4 mr-2 animate-spin" />}
                Connect
              </Button>
            )}
          </div>
        </div>
      </div>

      {/* Stats Panel */}
      {connected && stats && (
        <div className="absolute top-24 left-4 z-10">
          <Card className="p-3 bg-background/95 backdrop-blur-sm">
            <div className="text-sm font-semibold mb-2">Connection Stats</div>
            <div className="space-y-1.5 text-xs">
              <div className="flex justify-between items-center gap-4">
                <span className="text-muted-foreground">FPS:</span>
                <Badge variant="outline">{stats.fps.toFixed(0)}</Badge>
              </div>
              <div className="flex justify-between items-center gap-4">
                <span className="text-muted-foreground">Latency:</span>
                <Badge variant="outline">{stats.rtt.toFixed(0)} ms</Badge>
              </div>
              <div className="flex justify-between items-center gap-4">
                <span className="text-muted-foreground">Bitrate:</span>
                <Badge variant="outline">{stats.bitrate.toFixed(1)} Mbps</Badge>
              </div>
              <div className="flex justify-between items-center gap-4">
                <span className="text-muted-foreground">Packet Loss:</span>
                <Badge variant="outline">{stats.packetsLost}</Badge>
              </div>
            </div>
          </Card>
        </div>
      )}

      {/* Video Stream */}
      <div className="relative w-full h-full flex items-center justify-center bg-gray-900">
        {loading && (
          <div className="absolute inset-0 flex items-center justify-center">
            <Loader2 className="w-12 h-12 animate-spin text-white" />
            <span className="ml-4 text-white text-lg">Connecting to GPU worker...</span>
          </div>
        )}

        {error && (
          <div className="absolute inset-0 flex items-center justify-center">
            <Card className="p-6 max-w-md">
              <h3 className="text-lg font-semibold text-red-600 mb-2">Connection Error</h3>
              <p className="text-sm text-muted-foreground mb-4">{error.message}</p>
              <Button onClick={connect}>Retry Connection</Button>
            </Card>
          </div>
        )}

        <video
          ref={videoRef}
          className="w-full h-full object-contain"
          autoPlay
          playsInline
          muted
          onMouseMove={handleMouseMove}
          onWheel={handleWheel}
          onClick={handleClick}
          style={{ cursor: connected ? 'grab' : 'default' }}
        />
      </div>

      {/* Controls Guide */}
      <div className="absolute bottom-4 right-4 bg-black/80 text-white p-3 rounded-lg text-xs max-w-xs">
        <div className="font-semibold mb-2">Controls:</div>
        <div className="space-y-1">
          <div>• <strong>Left Drag</strong>: Rotate camera</div>
          <div>• <strong>Scroll</strong>: Zoom in/out</div>
          <div>• <strong>Click</strong>: Select device</div>
        </div>
      </div>
    </div>
  )
}
```

---

## 2. Signaling Server

### 2.1 Main Server Implementation

```typescript
// signaling-server/src/index.ts

import express from 'express'
import { WebSocketServer, WebSocket } from 'ws'
import Redis from 'ioredis'
import { v4 as uuidv4 } from 'uuid'
import http from 'http'

const app = express()
const server = http.createServer(app)
const wss = new WebSocketServer({ server })

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD
})

const orchestratorUrl = process.env.ORCHESTRATOR_URL || 'http://localhost:3000'

interface Session {
  sessionId: string
  clientId: string
  workerId: string
  clientWs: WebSocket
  workerWs: WebSocket | null
  createdAt: number
  lastActivity: number
}

const sessions = new Map<string, Session>()

// Client connection
wss.on('connection', async (ws: WebSocket, req) => {
  const url = new URL(req.url!, `ws://${req.headers.host}`)
  const userId = url.searchParams.get('userId')
  const projectId = url.searchParams.get('projectId')
  const clientType = url.searchParams.get('type') || 'client'

  if (!userId || !projectId) {
    ws.close(1008, 'Missing userId or projectId')
    return
  }

  console.log(`[Signaling] ${clientType} connected: ${userId}`)

  if (clientType === 'client') {
    await handleClientConnection(ws, userId, projectId)
  } else if (clientType === 'worker') {
    await handleWorkerConnection(ws, userId) // userId is workerId for workers
  }
})

async function handleClientConnection(ws: WebSocket, userId: string, projectId: string) {
  try {
    // 1. Request worker allocation from orchestrator
    const response = await fetch(`${orchestratorUrl}/allocate`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        clientId: userId,
        projectId,
        requiredGPU: 'A10G'
      })
    })

    if (!response.ok) {
      const error = await response.text()
      ws.send(JSON.stringify({ type: 'error', error: `Worker allocation failed: ${error}` }))
      ws.close()
      return
    }

    const { workerId, endpoint } = await response.json()

    // 2. Create session
    const sessionId = uuidv4()
    const session: Session = {
      sessionId,
      clientId: userId,
      workerId,
      clientWs: ws,
      workerWs: null,
      createdAt: Date.now(),
      lastActivity: Date.now()
    }

    sessions.set(sessionId, session)

    // 3. Store in Redis
    await redis.set(
      `session:${sessionId}`,
      JSON.stringify({
        sessionId,
        clientId: userId,
        workerId,
        projectId,
        createdAt: session.createdAt
      }),
      'EX',
      3600 // 1 hour expiry
    )

    // 4. Notify client
    ws.send(JSON.stringify({
      type: 'session-created',
      sessionId,
      workerId,
      endpoint
    }))

    // 5. Handle client messages
    ws.on('message', async (data) => {
      try {
        const message = JSON.parse(data.toString())
        await handleClientMessage(sessionId, message)
      } catch (err) {
        console.error('[Signaling] Failed to parse client message:', err)
      }
    })

    ws.on('close', () => {
      handleClientDisconnect(sessionId)
    })

  } catch (err) {
    console.error('[Signaling] Client connection error:', err)
    ws.send(JSON.stringify({ type: 'error', error: 'Internal server error' }))
    ws.close()
  }
}

async function handleWorkerConnection(ws: WebSocket, workerId: string) {
  console.log(`[Signaling] Worker connected: ${workerId}`)

  // Find session for this worker
  let session: Session | undefined
  for (const s of sessions.values()) {
    if (s.workerId === workerId && !s.workerWs) {
      session = s
      break
    }
  }

  if (!session) {
    console.warn(`[Signaling] No session found for worker ${workerId}`)
    ws.close(1008, 'No session found')
    return
  }

  session.workerWs = ws

  // Handle worker messages
  ws.on('message', async (data) => {
    try {
      const message = JSON.parse(data.toString())
      await handleWorkerMessage(session!.sessionId, message)
    } catch (err) {
      console.error('[Signaling] Failed to parse worker message:', err)
    }
  })

  ws.on('close', () => {
    handleWorkerDisconnect(session!.sessionId)
  })
}

async function handleClientMessage(sessionId: string, message: any) {
  const session = sessions.get(sessionId)
  if (!session) {
    console.warn(`[Signaling] Session not found: ${sessionId}`)
    return
  }

  session.lastActivity = Date.now()

  // Forward signaling messages to worker
  if (['offer', 'answer', 'ice-candidate'].includes(message.type)) {
    if (session.workerWs?.readyState === WebSocket.OPEN) {
      session.workerWs.send(JSON.stringify({
        ...message,
        sessionId,
        from: 'client'
      }))
    } else {
      console.warn(`[Signaling] Worker not connected for session ${sessionId}`)
    }
  }
}

async function handleWorkerMessage(sessionId: string, message: any) {
  const session = sessions.get(sessionId)
  if (!session) {
    console.warn(`[Signaling] Session not found: ${sessionId}`)
    return
  }

  session.lastActivity = Date.now()

  // Forward signaling messages to client
  if (['offer', 'answer', 'ice-candidate'].includes(message.type)) {
    if (session.clientWs?.readyState === WebSocket.OPEN) {
      session.clientWs.send(JSON.stringify({
        ...message,
        sessionId,
        from: 'worker'
      }))
    }
  }
}

function handleClientDisconnect(sessionId: string) {
  const session = sessions.get(sessionId)
  if (!session) return

  console.log(`[Signaling] Client disconnected: ${session.clientId}`)

  // Notify worker to cleanup
  if (session.workerWs?.readyState === WebSocket.OPEN) {
    session.workerWs.send(JSON.stringify({
      type: 'client-disconnected',
      sessionId
    }))
  }

  // Cleanup
  sessions.delete(sessionId)
  redis.del(`session:${sessionId}`)
}

function handleWorkerDisconnect(sessionId: string) {
  const session = sessions.get(sessionId)
  if (!session) return

  console.log(`[Signaling] Worker disconnected: ${session.workerId}`)

  // Notify client
  if (session.clientWs?.readyState === WebSocket.OPEN) {
    session.clientWs.send(JSON.stringify({
      type: 'error',
      error: 'Worker disconnected'
    }))
    session.clientWs.close()
  }

  // Cleanup
  sessions.delete(sessionId)
  redis.del(`session:${sessionId}`)
}

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    activeSessions: sessions.size,
    uptime: process.uptime()
  })
})

// Metrics endpoint (Prometheus format)
app.get('/metrics', (req, res) => {
  const metrics = `
# HELP signaling_active_sessions Number of active signaling sessions
# TYPE signaling_active_sessions gauge
signaling_active_sessions ${sessions.size}

# HELP signaling_uptime_seconds Server uptime in seconds
# TYPE signaling_uptime_seconds counter
signaling_uptime_seconds ${process.uptime()}
`
  res.set('Content-Type', 'text/plain')
  res.send(metrics)
})

// Session cleanup (idle sessions)
setInterval(() => {
  const now = Date.now()
  const timeout = 10 * 60 * 1000 // 10 minutes

  for (const [sessionId, session] of sessions.entries()) {
    if (now - session.lastActivity > timeout) {
      console.log(`[Signaling] Cleaning up idle session: ${sessionId}`)
      handleClientDisconnect(sessionId)
    }
  }
}, 60000) // Check every minute

const PORT = parseInt(process.env.PORT || '8080')
server.listen(PORT, () => {
  console.log(`[Signaling] Server listening on port ${PORT}`)
})
```

### 2.2 Docker Configuration

```dockerfile
# signaling-server/Dockerfile

FROM node:20-alpine

WORKDIR /app

# Install dependencies
COPY package.json package-lock.json ./
RUN npm ci --production

# Copy source
COPY . .

# Build TypeScript
RUN npm run build

# Non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001
USER nodejs

EXPOSE 8080

CMD ["node", "dist/index.js"]
```

```yaml
# signaling-server/docker-compose.yml

version: '3.8'

services:
  signaling-server:
    build: .
    ports:
      - "8080:8080"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - ORCHESTRATOR_URL=http://orchestrator:3000
    depends_on:
      - redis
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    restart: unless-stopped

volumes:
  redis-data:
```

---

**[Document continues with sections 3-6 covering GPU Worker, Orchestrator, Infrastructure as Code, and Testing examples. Due to length, I'm providing a comprehensive structure. Would you like me to continue with specific sections?]**

---

## Quick Start Guide

### Local Development Setup

```bash
# 1. Clone repository
git clone https://github.com/your-org/remote-render.git
cd remote-render

# 2. Install client dependencies
npm install

# 3. Start signaling server
cd signaling-server
npm install
docker-compose up -d
npm run dev

# 4. Start GPU worker (requires NVIDIA GPU)
cd ../gpu-worker
npm install
npm run dev

# 5. Start client application
cd ..
npm run dev

# 6. Open browser
open http://localhost:3000
```

### Production Deployment

```bash
# Deploy infrastructure
cd terraform
terraform init
terraform plan -out=plan.tfplan
terraform apply plan.tfplan

# Build and push Docker images
docker build -t signaling-server:latest signaling-server/
docker tag signaling-server:latest $ECR_REGISTRY/signaling-server:latest
docker push $ECR_REGISTRY/signaling-server:latest

# Update ECS services
aws ecs update-service --cluster remote-render --service signaling-server --force-new-deployment
```

---

## Troubleshooting

### Common Issues

1. **WebRTC connection fails**
   - Check STUN/TURN server configuration
   - Verify firewall allows UDP 49152-65535
   - Test NAT traversal with `webrtc-internals` (chrome://webrtc-internals)

2. **High latency**
   - Check network RTT (ping)
   - Verify GPU encoding latency (< 10ms)
   - Reduce bitrate or resolution

3. **Video stream freezes**
   - Check packet loss (> 5% is bad)
   - Verify GPU worker not overloaded
   - Check network bandwidth

4. **Worker provisioning slow**
   - Increase warm pool size
   - Pre-cache AMI in region
   - Check EC2 service limits

---

**End of Implementation Guide**
