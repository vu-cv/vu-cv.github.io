---
layout: article
title: WebRTC – Video Call cơ bản với NodeJS & Socket.io
tags: [webrtc, video-call, socket.io, nodejs, nestjs, real-time]
---
WebRTC cho phép trình duyệt giao tiếp peer-to-peer trực tiếp — video call, voice call, file sharing mà không qua server. NodeJS chỉ cần làm signaling server để hai peer tìm thấy nhau.

## 1. WebRTC hoạt động như thế nào?

```
Peer A                  Signaling Server (NestJS)            Peer B
  │                            │                              │
  ├──── offer (SDP) ─────────→ │ ──── forward offer ────────→ │
  │                            │                              │
  │ ←─── answer (SDP) ─────── │ ←── answer ────────────────── │
  │                            │                              │
  ├──── ICE candidate ────────→ │ ──── forward candidate ────→ │
  │ ←─── ICE candidate ─────── │ ←── ICE candidate ──────────  │
  │                            │                              │
  ├──────────────── Peer-to-Peer connection ─────────────────── │
  │         (video/audio trực tiếp, không qua server)          │
```

SDP (Session Description Protocol): Mô tả codec, resolution, bandwidth.
ICE (Interactive Connectivity Establishment): Tìm đường kết nối qua NAT.

## 2. Backend — Signaling Server

```typescript
// src/webrtc/webrtc.gateway.ts
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  OnGatewayConnection, OnGatewayDisconnect, ConnectedSocket, MessageBody,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';

interface Room {
  participants: Map<string, { socketId: string; userId: string; name: string }>;
}

@WebSocketGateway({
  cors: { origin: process.env.FRONTEND_URL, credentials: true },
  namespace: '/webrtc',
})
export class WebRTCGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private rooms = new Map<string, Room>();
  private socketToRoom = new Map<string, string>(); // socketId → roomId

  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    const roomId = this.socketToRoom.get(client.id);
    if (roomId) {
      this.leaveRoom(client, roomId);
    }
  }

  // Vào phòng
  @SubscribeMessage('join-room')
  handleJoinRoom(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { roomId: string; userId: string; name: string },
  ) {
    const { roomId, userId, name } = data;

    if (!this.rooms.has(roomId)) {
      this.rooms.set(roomId, { participants: new Map() });
    }

    const room = this.rooms.get(roomId)!;

    if (room.participants.size >= 10) {
      client.emit('room-full');
      return;
    }

    // Thông báo cho participants hiện tại biết có người mới
    const existingParticipants = Array.from(room.participants.values());
    client.emit('existing-participants', existingParticipants);

    room.participants.set(client.id, { socketId: client.id, userId, name });
    this.socketToRoom.set(client.id, roomId);

    client.join(roomId);

    // Thông báo cho tất cả (trừ người vừa join)
    client.to(roomId).emit('participant-joined', { socketId: client.id, userId, name });
  }

  // Relay WebRTC signals
  @SubscribeMessage('offer')
  handleOffer(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { targetSocketId: string; offer: RTCSessionDescriptionInit },
  ) {
    this.server.to(data.targetSocketId).emit('offer', {
      offer: data.offer,
      fromSocketId: client.id,
    });
  }

  @SubscribeMessage('answer')
  handleAnswer(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { targetSocketId: string; answer: RTCSessionDescriptionInit },
  ) {
    this.server.to(data.targetSocketId).emit('answer', {
      answer: data.answer,
      fromSocketId: client.id,
    });
  }

  @SubscribeMessage('ice-candidate')
  handleIceCandidate(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { targetSocketId: string; candidate: RTCIceCandidateInit },
  ) {
    this.server.to(data.targetSocketId).emit('ice-candidate', {
      candidate: data.candidate,
      fromSocketId: client.id,
    });
  }

  // Tắt mic/camera
  @SubscribeMessage('media-state')
  handleMediaState(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { audio: boolean; video: boolean },
  ) {
    const roomId = this.socketToRoom.get(client.id);
    if (roomId) {
      client.to(roomId).emit('participant-media-state', {
        socketId: client.id,
        ...data,
      });
    }
  }

  private leaveRoom(client: Socket, roomId: string) {
    const room = this.rooms.get(roomId);
    if (!room) return;

    room.participants.delete(client.id);
    client.to(roomId).emit('participant-left', { socketId: client.id });
    client.leave(roomId);
    this.socketToRoom.delete(client.id);

    if (room.participants.size === 0) {
      this.rooms.delete(roomId);
    }
  }
}
```

## 3. Frontend — WebRTC Client

```tsx
// hooks/useWebRTC.ts
'use client';
import { useEffect, useRef, useState, useCallback } from 'react';
import { io, Socket } from 'socket.io-client';

interface Participant {
  socketId: string;
  userId: string;
  name: string;
  stream?: MediaStream;
  audioEnabled: boolean;
  videoEnabled: boolean;
}

const ICE_SERVERS = {
  iceServers: [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
    // TURN server cho NAT traversal (cần cho production)
    // { urls: 'turn:turn.example.com:3478', username: 'user', credential: 'pass' }
  ],
};

export function useWebRTC(roomId: string, userId: string, name: string) {
  const [localStream, setLocalStream] = useState<MediaStream | null>(null);
  const [participants, setParticipants] = useState<Map<string, Participant>>(new Map());
  const socketRef = useRef<Socket>();
  const peersRef = useRef<Map<string, RTCPeerConnection>>(new Map());
  const localStreamRef = useRef<MediaStream>();

  // Khởi tạo local media
  const initLocalStream = useCallback(async () => {
    const stream = await navigator.mediaDevices.getUserMedia({
      video: { width: 1280, height: 720, frameRate: 30 },
      audio: { echoCancellation: true, noiseSuppression: true },
    });
    localStreamRef.current = stream;
    setLocalStream(stream);
    return stream;
  }, []);

  // Tạo PeerConnection với participant
  const createPeerConnection = useCallback((targetSocketId: string) => {
    const pc = new RTCPeerConnection(ICE_SERVERS);

    // Thêm local tracks
    localStreamRef.current?.getTracks().forEach(track => {
      pc.addTrack(track, localStreamRef.current!);
    });

    // Nhận remote tracks
    pc.ontrack = (event) => {
      setParticipants(prev => {
        const updated = new Map(prev);
        const participant = updated.get(targetSocketId);
        if (participant) {
          updated.set(targetSocketId, { ...participant, stream: event.streams[0] });
        }
        return updated;
      });
    };

    // Gửi ICE candidates
    pc.onicecandidate = (event) => {
      if (event.candidate) {
        socketRef.current?.emit('ice-candidate', {
          targetSocketId,
          candidate: event.candidate,
        });
      }
    };

    peersRef.current.set(targetSocketId, pc);
    return pc;
  }, []);

  useEffect(() => {
    const init = async () => {
      await initLocalStream();

      const socket = io(`${process.env.NEXT_PUBLIC_API_URL}/webrtc`);
      socketRef.current = socket;

      socket.on('connect', () => {
        socket.emit('join-room', { roomId, userId, name });
      });

      // Kết nối với participants đã có trong phòng
      socket.on('existing-participants', async (existingParticipants: Participant[]) => {
        for (const participant of existingParticipants) {
          setParticipants(prev => new Map(prev).set(participant.socketId, {
            ...participant, audioEnabled: true, videoEnabled: true,
          }));

          const pc = createPeerConnection(participant.socketId);
          const offer = await pc.createOffer();
          await pc.setLocalDescription(offer);

          socket.emit('offer', { targetSocketId: participant.socketId, offer });
        }
      });

      socket.on('participant-joined', (participant: Participant) => {
        setParticipants(prev => new Map(prev).set(participant.socketId, {
          ...participant, audioEnabled: true, videoEnabled: true,
        }));
        // Người mới join → họ sẽ gửi offer đến chúng ta
      });

      // Nhận offer từ người mới
      socket.on('offer', async ({ offer, fromSocketId }) => {
        const pc = createPeerConnection(fromSocketId);
        await pc.setRemoteDescription(offer);
        const answer = await pc.createAnswer();
        await pc.setLocalDescription(answer);
        socket.emit('answer', { targetSocketId: fromSocketId, answer });
      });

      socket.on('answer', async ({ answer, fromSocketId }) => {
        const pc = peersRef.current.get(fromSocketId);
        await pc?.setRemoteDescription(answer);
      });

      socket.on('ice-candidate', async ({ candidate, fromSocketId }) => {
        const pc = peersRef.current.get(fromSocketId);
        await pc?.addIceCandidate(candidate);
      });

      socket.on('participant-left', ({ socketId }) => {
        peersRef.current.get(socketId)?.close();
        peersRef.current.delete(socketId);
        setParticipants(prev => {
          const updated = new Map(prev);
          updated.delete(socketId);
          return updated;
        });
      });
    };

    init();

    return () => {
      localStreamRef.current?.getTracks().forEach(t => t.stop());
      peersRef.current.forEach(pc => pc.close());
      socketRef.current?.disconnect();
    };
  }, [roomId]);

  // Toggle mic/camera
  const toggleAudio = useCallback(() => {
    const audioTrack = localStreamRef.current?.getAudioTracks()[0];
    if (audioTrack) {
      audioTrack.enabled = !audioTrack.enabled;
      socketRef.current?.emit('media-state', {
        audio: audioTrack.enabled,
        video: localStreamRef.current?.getVideoTracks()[0]?.enabled ?? true,
      });
    }
  }, []);

  return { localStream, participants, toggleAudio };
}
```

## 4. Video UI Component

```tsx
// components/VideoRoom.tsx
'use client';
import { useWebRTC } from '@/hooks/useWebRTC';

export function VideoRoom({ roomId, userId, name }) {
  const { localStream, participants, toggleAudio } = useWebRTC(roomId, userId, name);

  return (
    <div className="grid grid-cols-2 gap-4 p-4 bg-gray-900 min-h-screen">
      {/* Local video */}
      <div className="relative">
        <video
          ref={el => { if (el && localStream) el.srcObject = localStream }}
          autoPlay muted playsInline
          className="w-full rounded-lg"
        />
        <span className="absolute bottom-2 left-2 text-white text-sm bg-black/50 px-2 rounded">
          Bạn
        </span>
      </div>

      {/* Remote videos */}
      {Array.from(participants.values()).map(participant => (
        <div key={participant.socketId} className="relative">
          <video
            ref={el => { if (el && participant.stream) el.srcObject = participant.stream }}
            autoPlay playsInline
            className="w-full rounded-lg"
          />
          <span className="absolute bottom-2 left-2 text-white text-sm bg-black/50 px-2 rounded">
            {participant.name}
          </span>
        </div>
      ))}

      {/* Controls */}
      <div className="col-span-2 flex justify-center gap-4">
        <button onClick={toggleAudio} className="p-3 rounded-full bg-gray-700 text-white">
          🎤
        </button>
      </div>
    </div>
  );
}
```

## 5. Kết luận

- **Signaling**: Server chỉ relay SDP và ICE — không xử lý media
- **STUN**: Tìm public IP — dùng Google STUN miễn phí cho dev
- **TURN**: Relay khi STUN thất bại (symmetric NAT) — cần cho production
- **ICE candidates**: Exchange liên tục cho đến khi tìm được path tốt nhất
- **SFU**: Khi > 4 participants, dùng SFU (Selective Forwarding Unit) như mediasoup thay vì full mesh

WebRTC full mesh phù hợp cho < 6 người — nhiều hơn cần SFU server để tiết kiệm băng thông.
