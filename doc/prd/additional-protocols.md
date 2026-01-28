# PRD: Additional Protocol Patterns (Phase 2)

**Issue**: sp-ms6.8
**Status**: Draft
**Author**: Claude
**Date**: 2026-01-27

## Overview

This document specifies the remaining Scalability Protocol patterns beyond REQ/REP: PUB/SUB for broadcast messaging, PIPELINE for work distribution, SURVEY for request-response fanout, BUS for peer-to-peer communication, and PAIR for bidirectional channels. These patterns build on the foundation established by the REQ/REP implementation.

## Common Design Elements

All protocols share these elements from the REQ/REP foundation:

- **State Machine Pattern**: Each socket type has a well-defined state machine
- **Protocol Goroutine**: Dedicated goroutine(s) manage protocol logic
- **I/O Worker Integration**: SendCh/RecvCh channels connect to WorkerPair
- **Shared Infrastructure**: BufferPool, PeerRegistry, ConnRegistry
- **Error Types**: Consistent error handling (ErrClosed, ErrTimeout, etc.)

---

## PUB/SUB Pattern

### Overview

PUB/SUB provides one-to-many message distribution. A PUB (publisher) socket broadcasts messages to all connected SUB (subscriber) sockets. Subscribers can filter messages by topic prefix.

### Requirements

| ID | Requirement |
|----|-------------|
| PS-1 | PUB socket broadcasts messages to all connected subscribers |
| PS-2 | SUB socket receives messages matching subscribed topics |
| PS-3 | Topic matching uses prefix-based filtering |
| PS-4 | Subscribers can subscribe/unsubscribe to multiple topics |
| PS-5 | Empty subscription matches all messages |
| PS-6 | No reply expected - fire and forget semantics |

### State Machines

**PUB Socket**: Stateless (always ready to send)
```
┌──────┐
│ OPEN │──► Send() broadcasts to all peers
└──────┘
```

**SUB Socket**: Stateless (always ready to receive)
```
┌──────┐
│ OPEN │──► Recv() returns matching messages
└──────┘
```

### Design

```go
// PubSocket broadcasts messages to all subscribers.
type PubSocket struct {
    base   *BaseSocket
    peers  *PeerRegistry

    sendCh chan<- *Message  // Broadcast to all peers
}

// Send broadcasts a message to all connected subscribers.
// Non-blocking if buffers available; blocks if all peer buffers full.
func (s *PubSocket) Send(data []byte) error

// SubSocket receives messages from publishers.
type SubSocket struct {
    base    *BaseSocket

    // Subscription management
    subsMu  sync.RWMutex
    subs    map[string]struct{}  // Topic prefixes

    recvCh  <-chan *Message
    filteredCh chan *Message  // After topic filtering
}

// Subscribe adds a topic prefix subscription.
// Messages with this prefix will be received.
func (s *SubSocket) Subscribe(topic string) error

// Unsubscribe removes a topic subscription.
func (s *SubSocket) Unsubscribe(topic string) error

// Recv receives the next message matching subscriptions.
// Blocks until a matching message arrives.
func (s *SubSocket) Recv() ([]byte, error)
```

### Topic Filtering

```go
func (s *SubSocket) matchesTopic(msg []byte) bool {
    s.subsMu.RLock()
    defer s.subsMu.RUnlock()

    // Empty subscription matches everything
    if len(s.subs) == 0 {
        return true
    }

    for topic := range s.subs {
        if bytes.HasPrefix(msg, []byte(topic)) {
            return true
        }
    }
    return false
}
```

### Fan-Out Strategy

PUB sends to all connected peers:
```go
func (s *PubSocket) broadcast(msg *Message) {
    peers := s.peers.All()
    for _, peer := range peers {
        // Clone message for each peer
        peerMsg := msg.Clone()
        peerMsg.PeerID = peer.ID

        select {
        case peer.SendCh <- peerMsg:
            // Sent
        default:
            // Peer buffer full, drop message for this peer
            peerMsg.Release()
        }
    }
    msg.Release()
}
```

---

## PIPELINE Pattern

### Overview

PIPELINE distributes work among multiple workers. PUSH sockets send tasks, PULL sockets receive them. Tasks are load-balanced across available workers.

### Requirements

| ID | Requirement |
|----|-------------|
| PL-1 | PUSH socket sends messages to one connected PULL socket |
| PL-2 | PULL socket receives messages from connected PUSH sockets |
| PL-3 | Messages are load-balanced across PULL sockets (round-robin) |
| PL-4 | No reply expected - one-way data flow |
| PL-5 | Backpressure when all workers busy |

### State Machines

**PUSH Socket**: Stateless (always ready to send)
```
┌──────┐
│ OPEN │──► Send() delivers to one worker
└──────┘
```

**PULL Socket**: Stateless (always ready to receive)
```
┌──────┐
│ OPEN │──► Recv() returns next task
└──────┘
```

### Design

```go
// PushSocket sends tasks to workers.
type PushSocket struct {
    base    *BaseSocket
    peers   *PeerRegistry
    peerIdx atomic.Uint32  // Round-robin index
}

// Send sends a task to one worker (round-robin selection).
// Blocks if all worker buffers are full.
func (s *PushSocket) Send(data []byte) error

// PullSocket receives tasks from producers.
type PullSocket struct {
    base   *BaseSocket
    recvCh <-chan *Message
}

// Recv receives the next task.
// Blocks until a task is available.
func (s *PullSocket) Recv() ([]byte, error)
```

### Load Balancing

```go
func (s *PushSocket) selectWorker() (*Peer, error) {
    peers := s.peers.All()
    if len(peers) == 0 {
        return nil, ErrNoPeers
    }

    // Round-robin with wraparound
    idx := s.peerIdx.Add(1) % uint32(len(peers))
    return peers[idx], nil
}
```

---

## SURVEY Pattern

### Overview

SURVEY implements request-response fanout. A SURVEYOR socket sends a query to all RESPONDENT sockets and collects responses within a deadline.

### Requirements

| ID | Requirement |
|----|-------------|
| SV-1 | SURVEYOR sends survey to all connected respondents |
| SV-2 | RESPONDENT receives surveys and sends responses |
| SV-3 | SURVEYOR collects responses within a deadline |
| SV-4 | Survey ID correlates responses to the originating survey |
| SV-5 | Partial results available when deadline expires |

### State Machines

**SURVEYOR Socket**:
```
              ┌──────┐
              │ IDLE │
              └──┬───┘
                 │ Send() (survey)
                 ▼
          ┌────────────────┐
          │ COLLECTING     │◄─── Recv() returns responses
          └───────┬────────┘
                  │ deadline or all responded
                  ▼
              ┌──────┐
              │ IDLE │
              └──────┘
```

**RESPONDENT Socket**:
```
              ┌──────┐
              │ IDLE │
              └──┬───┘
                 │ Recv() (survey)
                 ▼
          ┌────────────────┐
          │ RESPONDING     │
          └───────┬────────┘
                  │ Send() (response)
                  ▼
              ┌──────┐
              │ IDLE │
              └──────┘
```

### Design

```go
// SurveyorSocket sends surveys and collects responses.
type SurveyorSocket struct {
    base       *BaseSocket
    state      atomic.Uint32

    // Survey tracking
    surveyID   atomic.Uint32
    pending    *pendingSurvey
    deadline   time.Duration

    peers      *PeerRegistry
}

// pendingSurvey tracks an active survey.
type pendingSurvey struct {
    id         uint32
    responses  []*Message
    deadline   time.Time
    done       chan struct{}
}

// Send broadcasts a survey to all respondents.
// Starts the collection period.
func (s *SurveyorSocket) Send(data []byte) error

// Recv receives the next response to the active survey.
// Returns ErrTimeout when deadline expires.
// After timeout, subsequent Recv() calls return ErrInvalidState.
func (s *SurveyorSocket) Recv() ([]byte, error)

// SetDeadline sets the survey collection deadline.
func (s *SurveyorSocket) SetDeadline(d time.Duration)

// RespondentSocket receives surveys and sends responses.
type RespondentSocket struct {
    base            *BaseSocket
    state           atomic.Uint32
    currentSurveyID uint32
    currentPeer     PeerID
}

// Recv receives the next survey.
func (s *RespondentSocket) Recv() ([]byte, error)

// Send sends a response to the current survey.
func (s *RespondentSocket) Send(data []byte) error
```

---

## BUS Pattern

### Overview

BUS provides many-to-many communication. Each BUS socket can send and receive. Messages are delivered to all other connected BUS sockets.

### Requirements

| ID | Requirement |
|----|-------------|
| BU-1 | BUS socket can send and receive messages |
| BU-2 | Sent messages delivered to all other connected BUS peers |
| BU-3 | Messages not echoed back to sender |
| BU-4 | No state machine - always ready to send/receive |

### State Machine

**BUS Socket**: Stateless
```
┌──────┐
│ OPEN │──► Send() broadcasts to all peers
│      │──► Recv() returns messages from any peer
└──────┘
```

### Design

```go
// BusSocket provides many-to-many communication.
type BusSocket struct {
    base   *BaseSocket
    peers  *PeerRegistry

    sendCh chan<- *Message
    recvCh <-chan *Message
}

// Send broadcasts a message to all connected peers.
func (s *BusSocket) Send(data []byte) error {
    msg := s.base.pool.NewMessage(data)

    peers := s.peers.All()
    for _, peer := range peers {
        peerMsg := msg.Clone()
        peerMsg.PeerID = peer.ID
        s.sendCh <- peerMsg
    }
    msg.Release()
    return nil
}

// Recv receives a message from any connected peer.
func (s *BusSocket) Recv() ([]byte, error) {
    select {
    case msg := <-s.recvCh:
        data := make([]byte, len(msg.Data))
        copy(data, msg.Data)
        msg.Release()
        return data, nil
    case <-s.base.ctx.Done():
        return nil, ErrClosed
    }
}
```

---

## PAIR Pattern

### Overview

PAIR provides exclusive bidirectional communication between exactly two endpoints. Only one peer connection is allowed.

### Requirements

| ID | Requirement |
|----|-------------|
| PA-1 | PAIR socket connects to exactly one peer |
| PA-2 | Both endpoints can send and receive |
| PA-3 | Additional connection attempts rejected |
| PA-4 | No state machine - always ready to send/receive |

### State Machine

**PAIR Socket**: Stateless (but exclusive connection)
```
┌──────────────┐
│ DISCONNECTED │
└──────┬───────┘
       │ connect
       ▼
┌──────────────┐
│  CONNECTED   │──► Send()/Recv() available
└──────────────┘
```

### Design

```go
// PairSocket provides exclusive bidirectional communication.
type PairSocket struct {
    base   *BaseSocket

    // Exclusive peer
    peerMu sync.Mutex
    peer   *Peer

    sendCh chan<- *Message
    recvCh <-chan *Message
}

// Connect establishes connection to peer.
// Returns error if already connected.
func (s *PairSocket) Connect(addr string) error {
    s.peerMu.Lock()
    defer s.peerMu.Unlock()

    if s.peer != nil {
        return ErrAlreadyConnected
    }

    // Establish connection...
    return nil
}

// Send sends a message to the connected peer.
func (s *PairSocket) Send(data []byte) error {
    s.peerMu.Lock()
    peer := s.peer
    s.peerMu.Unlock()

    if peer == nil {
        return ErrNotConnected
    }

    msg := s.base.pool.NewMessage(data)
    msg.PeerID = peer.ID

    select {
    case s.sendCh <- msg:
        return nil
    case <-s.base.ctx.Done():
        msg.Release()
        return ErrClosed
    }
}

// Recv receives a message from the connected peer.
func (s *PairSocket) Recv() ([]byte, error) {
    select {
    case msg := <-s.recvCh:
        data := make([]byte, len(msg.Data))
        copy(data, msg.Data)
        msg.Release()
        return data, nil
    case <-s.base.ctx.Done():
        return nil, ErrClosed
    }
}
```

---

## Protocol Comparison

| Pattern | Topology | Direction | State Machine | Key Feature |
|---------|----------|-----------|---------------|-------------|
| REQ/REP | N:1 | Request-Reply | Yes | Correlation |
| PUB/SUB | 1:N | One-way | No | Topic filtering |
| PIPELINE | N:M | One-way | No | Load balancing |
| SURVEY | 1:N:1 | Query-Response | Yes | Deadline collection |
| BUS | N:N | Bidirectional | No | Peer mesh |
| PAIR | 1:1 | Bidirectional | No | Exclusive channel |

## Testing Strategy

### Pattern-Specific Tests

| Pattern | Tests |
|---------|-------|
| PUB/SUB | Fan-out, topic filtering, subscription management |
| PIPELINE | Load balancing, backpressure, worker failure |
| SURVEY | Response collection, deadline handling, partial results |
| BUS | Mesh connectivity, no echo, concurrent send/recv |
| PAIR | Exclusivity, bidirectional flow, reconnection |

### Integration Tests

| Test | Description |
|------|-------------|
| `TestPubSubMultipleSubscribers` | One publisher, many subscribers |
| `TestPubSubTopicFiltering` | Messages filtered by topic |
| `TestPipelineLoadBalance` | Work distributed evenly |
| `TestSurveyPartialResponses` | Some respondents don't reply |
| `TestBusMesh` | Three nodes, full connectivity |
| `TestPairExclusivity` | Second connection rejected |

## Acceptance Criteria

1. **PUB/SUB Implemented**: Broadcast with topic filtering
2. **PIPELINE Implemented**: Round-robin load balancing
3. **SURVEY Implemented**: Deadline-based response collection
4. **BUS Implemented**: Many-to-many mesh
5. **PAIR Implemented**: Exclusive bidirectional
6. **State Machines Correct**: All transitions validated
7. **Tests Pass**: Pattern-specific and integration tests
8. **Documentation**: GoDoc for all types/methods

## Dependencies

- REQ/REP Protocol Engine (sp-ms6.2) - Base patterns established
- Shared Infrastructure (sp-ms6.7) - BufferPool, registries
- I/O Workers (sp-ms6.3) - WorkerPair integration

## References

- [NNG PUB/SUB](https://nng.nanomsg.org/man/tip/nng_pub.7.html)
- [NNG PIPELINE](https://nng.nanomsg.org/man/tip/nng_push.7.html)
- [NNG SURVEY](https://nng.nanomsg.org/man/tip/nng_surveyor.7.html)
- [NNG BUS](https://nng.nanomsg.org/man/tip/nng_bus.7.html)
- [NNG PAIR](https://nng.nanomsg.org/man/tip/nng_pair.7.html)
