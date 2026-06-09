# Elevator System — Low Level Design

## Table of Contents
1. [Problem Clarifications](#1-problem-clarifications)
2. [Scope & Requirements](#2-scope--requirements)
3. [Entity Design](#3-entity-design)
4. [Class Design](#4-class-design)
5. [Core Algorithm — SCAN (Elevator)](#5-core-algorithm--scan-elevator)
6. [Implementation](#6-implementation)
7. [Edge Cases](#7-edge-cases)

---

## 1. Problem Clarifications

| Question | Decision |
|---|---|
| How many elevators and floors? | 3 elevators, 10 floors (0–9) |
| Hall calls: up/down or choose floor? | Direction-based (UP / DOWN) |
| What does "move efficiently" mean? | SCAN algorithm — elevator services requests in current direction before reversing |
| Can passengers select multiple destination floors? | Yes, `addRequest()` accepts any floor from inside the elevator |

---

## 2. Scope & Requirements

### In Scope
- System manages **3 elevators** serving **10 floors** (0–9)
- Users request an elevator from any floor (**hall call**) with a direction
- Once inside, users select one or more **destination floors**
- Simulation runs in **discrete time steps** via `step()` / `tick()`
- Two request types: `PICK_UP`, `PICK_DOWN` (hall), `DESTINATION` (inside cabin)
- System handles **multiple concurrent** pickup requests
- Invalid requests (out-of-bounds floors) are **rejected**

### Out of Scope
- Weight / passenger capacity limits
- Door open/close mechanics
- Emergency stop
- Dynamic floor or elevator reconfiguration
- UI / rendering layer

---

## 3. Entity Design

```
ElevatorController
│
├── Elevator (×3)
│   ├── int currentFloor
│   ├── Direction direction
│   └── set<Request> pendingRequests
│
└── Request
    ├── int floor
    └── RequestType type
```

---

## 4. Class Design

### Enums

```cpp
enum class Direction {
    UP,
    DOWN,
    IDLE
};

enum class RequestType {
    PICK_UP,       // Hall call going up
    PICK_DOWN,     // Hall call going down
    DESTINATION    // Inside-cabin floor selection
};
```

### Request

```cpp
class Request {
public:
    Request(int floor, RequestType type)
        : floor_(floor), type_(type) {}

    int          getFloor() const { return floor_; }
    RequestType  getType()  const { return type_; }

    // Equality needed for set deduplication
    bool operator==(const Request& other) const {
        return floor_ == other.floor_ && type_ == other.type_;
    }

    bool operator<(const Request& other) const {
        if (floor_ != other.floor_) return floor_ < other.floor_;
        return type_ < other.type_;
    }

private:
    int         floor_;
    RequestType type_;
};
```

### Elevator

```cpp
class Elevator {
public:
    explicit Elevator(int id)
        : id_(id), currentFloor_(0), direction_(Direction::IDLE) {}

    // Called by ElevatorController to assign a request
    bool addRequest(const Request& request) {
        auto [_, inserted] = pendingRequests_.insert(request);
        return inserted; // false if duplicate (already pending)
    }

    void step();                        // Advance one time tick (SCAN logic)
    int       getFloor()     const { return currentFloor_; }
    Direction getDirection() const { return direction_; }
    bool      isIdle()       const { return direction_ == Direction::IDLE; }

private:
    int              id_;
    int              currentFloor_;
    Direction        direction_;
    std::set<Request> pendingRequests_;

    // Returns the nearest pending request (used when IDLE)
    const Request* findNearest() const;

    // True if any pending request lies ahead in `dir`
    bool hasRequestAhead(Direction dir) const;
};
```

### ElevatorController

```cpp
class ElevatorController {
public:
    ElevatorController() {
        for (int i = 0; i < NUM_ELEVATORS; ++i) {
            elevators_.emplace_back(i);
        }
    }

    // External hall call from a floor; type must be PICK_UP or PICK_DOWN
    bool requestElevator(int floor, RequestType type);

    // Advance every elevator by one tick
    void step() {
        for (auto& e : elevators_) {
            e.step();
        }
    }

private:
    static constexpr int NUM_ELEVATORS = 3;
    static constexpr int NUM_FLOORS    = 10;

    std::vector<Elevator> elevators_;

    // Dispatcher
    Elevator* selectBestElevator(const Request& request);

    // Selector helpers
    Elevator* findMovingToward (const Request& request);
    Elevator* findNearestIdle  (int floor);
    Elevator* findNearest      (int floor);
};
```

---

## 5. Core Algorithm — SCAN (Elevator)

The SCAN (disk scheduling) algorithm is used inside each `Elevator::step()`.

```
1. If no pending requests  →  set IDLE, return
2. If IDLE  →  set direction toward the nearest pending request
3. Check current floor:
   a. Hall-call match (floor + direction)  →  remove request, open doors, return
   b. Destination match (floor)            →  remove request, open doors, return
4. If no requests ahead in current direction  →  reverse direction, return
5. Move one floor in current direction
```

### Elevator::step() — Full Implementation

```cpp
void Elevator::step() {
    // 1. Nothing to do
    if (pendingRequests_.empty()) {
        direction_ = Direction::IDLE;
        return;
    }

    // 2. Bootstrap direction when idle
    if (direction_ == Direction::IDLE) {
        const Request* nearest = findNearest();
        direction_ = (nearest->getFloor() > currentFloor_)
                     ? Direction::UP
                     : Direction::DOWN;
    }

    // 3a. Check for a matching hall call at this floor
    RequestType hallType = (direction_ == Direction::UP)
                           ? RequestType::PICK_UP
                           : RequestType::PICK_DOWN;
    Request hallCall(currentFloor_, hallType);

    if (pendingRequests_.count(hallCall)) {
        pendingRequests_.erase(hallCall);
        openDoor(); // out-of-scope stub
        return;
    }

    // 3b. Check for a destination request at this floor
    Request destination(currentFloor_, RequestType::DESTINATION);

    if (pendingRequests_.count(destination)) {
        pendingRequests_.erase(destination);
        openDoor();
        return;
    }

    // 4. No requests ahead → reverse
    if (!hasRequestAhead(direction_)) {
        direction_ = (direction_ == Direction::UP)
                     ? Direction::DOWN
                     : Direction::UP;
        return;
    }

    // 5. Move one floor
    if (direction_ == Direction::UP) {
        ++currentFloor_;
    } else {
        --currentFloor_;
    }
}
```

### hasRequestAhead()

```cpp
bool Elevator::hasRequestAhead(Direction dir) const {
    for (const auto& req : pendingRequests_) {
        if (dir == Direction::UP   && req.getFloor() > currentFloor_) return true;
        if (dir == Direction::DOWN && req.getFloor() < currentFloor_) return true;
    }
    return false;
}
```

### findNearest()

```cpp
const Request* Elevator::findNearest() const {
    const Request* best = nullptr;
    int minDist = INT_MAX;
    for (const auto& req : pendingRequests_) {
        int dist = std::abs(req.getFloor() - currentFloor_);
        if (dist < minDist) {
            minDist = dist;
            best = &req;
        }
    }
    return best;
}
```

---

## 6. Implementation — ElevatorController

### requestElevator()

```cpp
bool ElevatorController::requestElevator(int floor, RequestType type) {
    // Reject inside-cabin request type on the external API
    if (type == RequestType::DESTINATION) {
        throw std::invalid_argument(
            "External hall calls cannot use RequestType::DESTINATION.");
    }

    // Validate floor bounds
    if (floor < 0 || floor >= NUM_FLOORS) {
        return false; // out of bounds → reject silently
    }

    Request request(floor, type);
    Elevator* best = selectBestElevator(request);

    if (best == nullptr) return false;

    return best->addRequest(request);
}
```

### selectBestElevator()

```cpp
Elevator* ElevatorController::selectBestElevator(const Request& request) {
    // Priority 1: elevator already moving toward this request
    Elevator* best = findMovingToward(request);
    if (best) return best;

    // Priority 2: idle elevator closest to the floor
    best = findNearestIdle(request.getFloor());
    if (best) return best;

    // Priority 3: any elevator closest to the floor
    return findNearest(request.getFloor());
}
```

### findMovingToward()

```cpp
Elevator* ElevatorController::findMovingToward(const Request& request) {
    int       floor     = request.getFloor();
    Direction reqDir    = (request.getType() == RequestType::PICK_UP)
                          ? Direction::UP
                          : Direction::DOWN;
    Elevator* nearest   = nullptr;
    int       minDist   = INT_MAX;

    for (auto& e : elevators_) {
        if (e.getDirection() != reqDir) continue;

        // Elevator must not have already passed the target floor
        bool alreadyPassed =
            (reqDir == Direction::UP   && e.getFloor() > floor) ||
            (reqDir == Direction::DOWN && e.getFloor() < floor);

        if (alreadyPassed) continue;

        int dist = std::abs(e.getFloor() - floor);
        if (dist < minDist) {
            minDist = dist;
            nearest = &e;
        }
    }
    return nearest;
}
```

### findNearestIdle() & findNearest()

```cpp
Elevator* ElevatorController::findNearestIdle(int floor) {
    Elevator* nearest = nullptr;
    int       minDist = INT_MAX;

    for (auto& e : elevators_) {
        if (!e.isIdle()) continue;
        int dist = std::abs(e.getFloor() - floor);
        if (dist < minDist) {
            minDist = dist;
            nearest = &e;
        }
    }
    return nearest;
}

Elevator* ElevatorController::findNearest(int floor) {
    Elevator* nearest = nullptr;
    int       minDist = INT_MAX;

    for (auto& e : elevators_) {
        int dist = std::abs(e.getFloor() - floor);
        if (dist < minDist) {
            minDist = dist;
            nearest = &e;
        }
    }
    return nearest;
}
```

---

## 7. Edge Cases

| Scenario | Handling |
|---|---|
| Floor < 0 or ≥ 10 | `requestElevator` returns `false` |
| `DESTINATION` passed as a hall call type | `std::invalid_argument` thrown |
| Request for the current floor | Served immediately on the **next** `step()` call (treated as already-arrived) |
| Duplicate request (same floor + direction already queued) | `std::set` deduplication — silently ignored, returns `false` from `addRequest` |
| All elevators busy, none moving toward request | Falls back to nearest elevator (any direction) |
| Elevator reaches top/bottom with no requests ahead | Reverses direction via `hasRequestAhead` check |
| `step()` called when elevator is IDLE with no requests | No-op, direction stays `IDLE` |
