# SDE1 Interview Preparation: Python Fullstack Developer

## Target: Senior roles at Meta, Google, Amazon, Microsoft, etc.

---

## Part 1: SDE1 Interview Expectations (Typical)

### Interview Rounds (4-5 rounds)
1. **Coding Round** - DSA (LeetCode Medium/Hard)
2. **System Design** - Scale a feature (30-45 min)
3. **Backend/API Design** - Design APIs for a problem
4. **Frontend** - Basic React component + integration
5. **Behavioral** - Projects, leadership, conflicts

### What They Test

| Round | What | Time | Example |
|-------|------|------|---------|
| **Coding** | DSA, Python optimization, edge cases | 45 min | Implement LRU Cache, design efficient algo |
| **System Design** | Scalability, databases, caching, queues | 45 min | Design Event Scheduler for 1M users |
| **Backend Design** | API design, validation, error handling | 45 min | Design REST API for events with auth |
| **Frontend** | React components, state, API integration | 45 min | Build event form + list with API calls |
| **Behavioral** | Impact, leadership, conflicts, learning | 30 min | Tell about a project you built |

---

## Part 2: Python DSA - Foundation (Non-negotiable)

### Must-Know Algorithms & Patterns

**Must Know (Top 30% of problems):**
```
1. Two Pointers
2. Sliding Window
3. Binary Search
4. DFS/BFS
5. Dynamic Programming (DP)
6. Hash Map
7. Heap/Priority Queue
8. Stack/Queue
9. Linked List
10. Tree Traversals (In/Pre/Post order)
```

**Target:** Solve 100 LeetCode problems
- 60 Easy
- 35 Medium
- 5 Hard

**Focus Areas:**
```
Arrays & Strings:     20%
Trees & Graphs:       25%
Linked Lists:         10%
Hash Maps/Sets:       15%
Stacks & Queues:      10%
Dynamic Programming:  15%
Binary Search:        5%
```

**Study Plan:**
- Week 1-2: Easy problems (3 per day)
- Week 3-4: Medium problems (2 per day)
- Week 5-6: Mix Medium + Hard (1 per day)
- Week 7-8: Mock interviews

**Resources:**
- LeetCode (subscription needed)
- NeetCode (free alternatives)
- GeeksforGeeks

---

## Part 3: System Design for SDE1

### What SDE1 System Design Looks Like

**NOT like SDE2:** You don't need micro-services, Kafka, Cassandra, etc.

**SDE1 = Single service, smart design**

### Typical SDE1 Design Problem

**"Design Event Scheduler for 10M users"**

Requirements:
- Users create events
- Share events with others
- Real-time notifications
- Search events by date
- Handle 100K concurrent users

### Solution Architecture (SDE1 Level)

```
FRONTEND (React)
    |
    └─→ Load Balancer (2-3 servers)
         |
         ├─→ API Server 1 (Python/FastAPI)
         ├─→ API Server 2
         └─→ API Server 3
              |
              ├─→ Cache (Redis)
              ├─→ Database (PostgreSQL)
              ├─→ Message Queue (Kafka/RabbitMQ)
              └─→ Search (Elasticsearch)
```

### Key Design Concepts SDE1 Must Know

**1. Database Design**
```python
# Users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR UNIQUE NOT NULL,
    password_hash VARCHAR NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

# Events table
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    title VARCHAR NOT NULL,
    date DATE NOT NULL,
    time TIME NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    
    # Indexes for query performance
    INDEX (user_id),
    INDEX (date)
);

# Event participants (many-to-many)
CREATE TABLE event_participants (
    event_id INT REFERENCES events(id),
    user_id INT REFERENCES users(id),
    status ENUM('invited', 'accepted', 'declined'),
    PRIMARY KEY (event_id, user_id)
);
```

**Why this design?**
- Normalized (no duplicate data)
- Indexes on frequently queried columns
- Foreign keys maintain integrity
- Many-to-many relationship handled separately

**2. API Design (REST)**
```
# ENDPOINTS

GET /api/events?date=2026-08-22
  → Fetch events for a date
  → Response: 200 + list of events
  → Cached in Redis (5 min TTL)

GET /api/events/{id}
  → Fetch single event
  → Response: 200 + event details + participants

POST /api/events
  → Create event
  → Validates: title, date, time
  → Response: 201 + created event

PUT /api/events/{id}
  → Update event
  → Check authorization (owner only)
  → Response: 200 + updated event

DELETE /api/events/{id}
  → Delete event
  → Check authorization
  → Response: 204 (No Content)

POST /api/events/{id}/invite
  → Invite user to event
  → Send notification (async via queue)
  → Response: 201

PUT /api/events/{id}/accept
  → Accept event invitation
  → Update status, send notification
  → Response: 200
```

**3. Authentication (JWT)**
```python
# Login flow
POST /api/auth/login
Body: { email, password }
Response: { access_token, refresh_token, expires_in }

# Make request with token
Headers: { Authorization: "Bearer <token>" }

# Token structure (JWT)
Header: { alg: "HS256", type: "JWT" }
Payload: { user_id: 123, email: "user@email.com", exp: 1693132800 }
Signature: (encrypted)
```

**4. Caching Strategy (Redis)**
```python
# Cache expensive queries
GET /api/events?date=2026-08-22
  1. Check Redis cache first
  2. If exists → Return cached (fast)
  3. If miss → Query DB → Cache 5 min → Return

# Cache invalidation
When user creates/updates/deletes event:
  → Delete related cache keys
  → Next request recalculates

# Pattern
cache_key = f"events:{date}"
ttl = 300  # 5 minutes
```

**5. Handling Notifications (Async Queue)**
```python
# Synchronous (BAD - slow)
@app.post("/api/events/{id}/invite")
def invite_user(event_id, user_id):
    # Send email (takes 2 seconds) - USER WAITS
    send_email()
    return { status: "invited" }

# Asynchronous (GOOD - fast)
@app.post("/api/events/{id}/invite")
def invite_user(event_id, user_id):
    # Queue task, return immediately
    queue.enqueue("send_invitation_email", event_id, user_id)
    return { status: "invited" }  # Responds in 100ms

# Worker processes queue in background
worker.process_queue()
    → send_email()
    → update_notification_status()
```

**6. Scalability Decisions**

| Problem | Solution | When |
|---------|----------|------|
| **Slow queries** | Database indexes | When > 100K records |
| **Repeated queries** | Redis cache | When same data requested 10x/sec |
| **Slow operations** | Async queue | When operation takes > 1 sec |
| **Concurrent users** | Load balancer | When > 1000 concurrent |
| **Search complexity** | Elasticsearch | When need full-text search |

---

## Part 4: Backend Implementation (Python/FastAPI)

### SDE1 Backend Patterns

**1. Proper Project Structure**
```
event-scheduler/
├── app/
│   ├── __init__.py
│   ├── main.py              # FastAPI app
│   ├── config.py            # Config, env variables
│   ├── database.py          # DB connection
│   ├── models/              # SQLAlchemy models
│   │   ├── user.py
│   │   └── event.py
│   ├── schemas/             # Pydantic validation
│   │   ├── user.py
│   │   └── event.py
│   ├── routes/              # API endpoints
│   │   ├── auth.py
│   │   ├── events.py
│   │   └── users.py
│   ├── services/            # Business logic
│   │   ├── event_service.py
│   │   └── auth_service.py
│   ├── utils/               # Helper functions
│   │   ├── validators.py
│   │   └── cache.py
│   └── middleware/          # Middleware
│       └── auth.py
├── tests/                   # Unit & integration tests
├── requirements.txt
└── .env
```

**2. FastAPI Skeleton (SDE1 Quality)**
```python
# main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from app.routes import auth, events
from app.config import settings

app = FastAPI(title="Event Scheduler API")

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Routes
app.include_router(auth.router, prefix="/api/auth", tags=["auth"])
app.include_router(events.router, prefix="/api/events", tags=["events"])

@app.get("/health")
def health_check():
    return {"status": "healthy"}
```

**3. Database Models (SQLAlchemy)**
```python
# models/user.py
from sqlalchemy import Column, Integer, String, DateTime
from datetime import datetime
from app.database import Base

class User(Base):
    __tablename__ = "users"
    
    id = Column(Integer, primary_key=True)
    email = Column(String, unique=True, index=True)
    password_hash = Column(String)
    created_at = Column(DateTime, default=datetime.utcnow)

# models/event.py
class Event(Base):
    __tablename__ = "events"
    
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey("users.id"), index=True)
    title = Column(String, index=True)
    date = Column(Date, index=True)
    time = Column(Time)
    created_at = Column(DateTime, default=datetime.utcnow)
```

**4. Validation (Pydantic)**
```python
# schemas/event.py
from pydantic import BaseModel, Field, validator
from datetime import date, time

class EventCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=100)
    date: date
    time: time
    description: Optional[str] = None
    
    @validator('date')
    def date_not_past(cls, v):
        if v < date.today():
            raise ValueError('Date must be today or later')
        return v

class EventResponse(EventCreate):
    id: int
    user_id: int
    created_at: datetime
    
    class Config:
        from_attributes = True  # SQLAlchemy model → schema
```

**5. Error Handling (SDE1 Pattern)**
```python
# utils/exceptions.py
class EventNotFound(Exception):
    pass

class UnauthorizedError(Exception):
    pass

class ValidationError(Exception):
    pass

# routes/events.py
from fastapi import HTTPException, status

@app.get("/api/events/{event_id}")
async def get_event(event_id: int):
    event = await db.query(Event).filter(Event.id == event_id).first()
    
    if not event:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail="Event not found"
        )
    
    return event
```

**6. Authentication (JWT)**
```python
# utils/auth.py
from passlib.context import CryptContext
from jose import JWTError, jwt
from datetime import datetime, timedelta

pwd_context = CryptContext(schemes=["bcrypt"])

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain, hashed) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user_id: int, expires_in: int = 3600):
    payload = {
        "user_id": user_id,
        "exp": datetime.utcnow() + timedelta(seconds=expires_in)
    }
    return jwt.encode(payload, settings.SECRET_KEY, algorithm="HS256")

# routes/auth.py
@app.post("/api/auth/login")
async def login(email: str, password: str):
    user = await db.query(User).filter(User.email == email).first()
    
    if not user or not verify_password(password, user.password_hash):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    token = create_access_token(user.id)
    return { "access_token": token, "token_type": "bearer" }

# Dependency for protected routes
async def get_current_user(token: str = Header(...)):
    try:
        payload = jwt.decode(token, settings.SECRET_KEY, algorithms=["HS256"])
        user_id = payload.get("user_id")
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    user = await db.query(User).filter(User.id == user_id).first()
    return user

@app.post("/api/events")
async def create_event(event: EventCreate, current_user = Depends(get_current_user)):
    # Protected route - only logged-in users
    new_event = Event(**event.dict(), user_id=current_user.id)
    db.add(new_event)
    db.commit()
    return new_event
```

---

## Part 5: Frontend (React) - SDE1 Requirements

### You DON'T need to be a React expert for SDE1, but need to:

**Must Know (45 min build):**
```
✓ Components (functional)
✓ Hooks (useState, useEffect)
✓ Event handling
✓ Form submission
✓ API integration (fetch)
✓ Error handling
✓ Loading states
✓ Basic styling (CSS)
```

**DON'T need:**
```
✗ Redux / Complex state management
✗ Advanced animations
✗ Mobile optimization
✗ Advanced TypeScript
✗ Testing (unit tests)
```

### SDE1 React Component Example

```javascript
import React, { useState, useEffect } from 'react';

function EventScheduler() {
  const [events, setEvents] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [formData, setFormData] = useState({
    title: '',
    date: '',
    time: ''
  });

  // Fetch events on mount
  useEffect(() => {
    fetchEvents();
  }, []);

  const fetchEvents = async () => {
    setLoading(true);
    try {
      const response = await fetch('/api/events', {
        headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
      });
      if (!response.ok) throw new Error('Failed to fetch');
      const data = await response.json();
      setEvents(data);
      setError(null);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };

  const handleCreateEvent = async (e) => {
    e.preventDefault();
    try {
      const response = await fetch('/api/events', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${localStorage.getItem('token')}`
        },
        body: JSON.stringify(formData)
      });
      if (!response.ok) throw new Error('Failed to create event');
      setFormData({ title: '', date: '', time: '' });
      fetchEvents();  // Refresh list
    } catch (err) {
      setError(err.message);
    }
  };

  return (
    <div>
      <h1>Event Scheduler</h1>
      
      {error && <div className="error">{error}</div>}
      {loading && <div>Loading...</div>}

      <form onSubmit={handleCreateEvent}>
        <input
          type="text"
          placeholder="Event title"
          value={formData.title}
          onChange={(e) => setFormData({ ...formData, title: e.target.value })}
          required
        />
        <input
          type="date"
          value={formData.date}
          onChange={(e) => setFormData({ ...formData, date: e.target.value })}
          required
        />
        <input
          type="time"
          value={formData.time}
          onChange={(e) => setFormData({ ...formData, time: e.target.value })}
          required
        />
        <button type="submit">Create Event</button>
      </form>

      <div className="events-list">
        {events.map((event) => (
          <div key={event.id} className="event-card">
            <h3>{event.title}</h3>
            <p>{event.date} at {event.time}</p>
          </div>
        ))}
      </div>
    </div>
  );
}

export default EventScheduler;
```

---

## Part 6: Testing (SDE1 Must-Know)

### Unit Tests (Python)
```python
# tests/test_events.py
import pytest
from app.models import Event, User
from app.schemas import EventCreate

@pytest.fixture
def client():
    # Setup test database
    return TestClient(app)

@pytest.fixture
def auth_headers(client):
    # Create test user, login, return auth headers
    pass

def test_create_event_success(client, auth_headers):
    event_data = {
        "title": "SE practical",
        "date": "2026-08-22",
        "time": "11:00"
    }
    response = client.post(
        "/api/events",
        json=event_data,
        headers=auth_headers
    )
    assert response.status_code == 201
    assert response.json()["title"] == "SE practical"

def test_create_event_unauthorized(client):
    response = client.post("/api/events", json={...})
    assert response.status_code == 401

def test_get_nonexistent_event(client, auth_headers):
    response = client.get("/api/events/999", headers=auth_headers)
    assert response.status_code == 404
```

---

## Part 7: Interview Preparation Strategy

### Timeline (12 weeks to SDE1)

**Week 1-4: DSA Foundation**
- LeetCode: 60 Easy problems (3/day)
- Focus: Arrays, Strings, HashMaps
- Target: Get comfortable with Python

**Week 5-8: DSA + Backend Concepts**
- LeetCode: 35 Medium problems (2/day)
- Build: Event Scheduler API (backend only)
- Focus: Database design, API patterns, auth

**Week 9-10: System Design**
- Design 3-4 systems at SDE1 level
- Study: Caching, Databases, Queues
- Practice: Design Event Scheduler for 10M users

**Week 11: Full Integration**
- Add React frontend to your API
- Write tests (unit + integration)
- Setup Docker + deployment

**Week 12: Interview Prep**
- Mock interviews (Pramp, InterviewBit)
- Practice behavioral
- Review weak areas

### What to Build (Portfolio Project)

**Event Scheduler (Full Stack)**
- Backend: FastAPI + PostgreSQL + Redis
- Frontend: React
- Features:
  - User auth (JWT)
  - Create/edit/delete events
  - Share events (invite users)
  - Real-time notifications (WebSocket or polling)
  - Search by date
  - Responsive design
- Deployment: Docker + AWS/Heroku
- Tests: 80%+ coverage

**Why this project:**
- Shows full stack understanding
- Tests all SDE1 concepts
- Realistic at scale
- Interview talking point

### Mock Interview Questions (Typical)

**System Design:**
- "Design event scheduler for 10M users"
- "How would you handle 100K concurrent users?"
- "How would you implement real-time notifications?"

**Backend:**
- "Design the database schema"
- "What indexes would you add?"
- "How would you handle race conditions?"

**Frontend:**
- "Build a form to create events"
- "How would you handle loading/error states?"
- "How would you optimize performance?"

**Behavioral:**
- "Tell me about a difficult problem you solved"
- "How did you handle a conflict with a team member?"
- "What did you learn building Event Scheduler?"

---

## Part 8: Key Concepts Summary (SDE1 Must Master)

| Concept | SDE1 Level | Learn From |
|---------|-----------|-----------|
| **DSA** | LeetCode Medium | LeetCode, NeetCode |
| **System Design** | Single service architecture | System Design Interview book |
| **Database** | SQL, indexing, relationships | PostgreSQL docs, SQL Bolt |
| **API Design** | REST, validation, auth | FastAPI docs |
| **Backend** | FastAPI, async, error handling | FastAPI tutorial |
| **Frontend** | React basics, state, API | React.dev docs |
| **Testing** | Unit tests, integration tests | Pytest docs |
| **Deployment** | Docker, basic CI/CD | Docker docs, AWS basics |

---

## Part 9: Resources (Python SDE1)

### DSA
- **LeetCode** (150+ Medium problems)
- **NeetCode** (free)
- **AlgoExpert** (Colt Steele's course)

### System Design
- **"Designing Data-Intensive Applications"** (Kleppmann) - Chapters 1-5
- **"System Design Interview"** (Alex Xu) - Vol 1

### Backend (FastAPI)
- **Official FastAPI docs:** https://fastapi.tiangolo.com
- **Real Python - FastAPI tutorial**
- **Miguel Grinberg - FastAPI course**

### Frontend (React)
- **React.dev** - Official docs
- **Scrimba React Course**

### Databases
- **SQL Bolt** - Interactive SQL tutorial
- **PostgreSQL official docs**
- **Real Python - SQLAlchemy ORM**

### Interview Prep
- **Pramp** - Free mock interviews
- **InterviewBit** - Coding + system design
- **LeetCode Premium** - Mock interviews

---

## 12-Week Study Schedule

```
WEEK 1-4: DSA FOUNDATION
├── Week 1: Arrays & Strings (15 problems)
├── Week 2: HashMaps & Sets (15 problems)
├── Week 3: Stacks & Queues (15 problems)
└── Week 4: Trees & Graphs (15 problems)

WEEK 5-8: DSA + BACKEND
├── Week 5: Medium DP (10 problems) + Start backend project
├── Week 6: Medium Mixed (10 problems) + Database design
├── Week 7: Medium Hard (10 problems) + API endpoints
└── Week 8: Hard (5 problems) + Auth & validation

WEEK 9-10: SYSTEM DESIGN
├── Week 9: Design patterns, caching, databases
└── Week 10: Design 4 systems (30 min each)

WEEK 11: FULL STACK
├── Build: React frontend
├── Integrate: Frontend + Backend
├── Add: Tests (70%+ coverage)
└── Deploy: Docker + Cloud

WEEK 12: INTERVIEW PREP
├── 3 mock interviews (Pramp)
├── R
