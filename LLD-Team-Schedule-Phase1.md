# 1. Overview

## 1.1 Goal

Define backend implementation design for Team Schedule Management module.

Includes:

* Application data model (Entity, Aggregate)
* Database schema, indexes, migration
* REST API + gRPC interfaces
* Business logic, validation, lifecycle
* Component responsibility & boundaries
* Message Queue + WebSocket event design

Objectives:

* Ensure consistent implementation
* Eliminate requirement ambiguity
* Define clear ownership & boundaries
* Serve as dev, review, maintenance reference
* Ensure scalability, maintainability, architecture compliance

---

## 1.2 In Scope

### Functional

Group management
Member management
Sprint management
Work (task) management
Backlog management
Work assignment
Work status management
Realtime update (WebSocket)
Event publishing (Message Queue)

### Technical

Application data model
DTO (API + internal)
Database schema, index, migration
REST API design
gRPC internal interfaces
Business logic + validation
Component interaction
Event model (MQ + WS)
Error mapping

---

## 1.3 Out of Scope

UI/UX
Frontend implementation
Infrastructure setup (K8s, CI/CD, deployment)
Monitoring, logging, observability setup
Authentication mechanism (JWT/OAuth implementation)
Authorization policy definition (RBAC config)
AI planning / chatbot
Realtime configuration
Timezone configuration

Focus: backend design and service communication.

---

# 2. Application Data Model

Defines domain model independent from database. Used for business logic and aggregate boundaries.

---

## 2.1 Entities

---

### User

```
id: UUID PK
email: string UNIQUE NOT NULL
status: enum(UserStatus) NOT NULL
created_at: datetime NOT NULL
```

Rules:

* email unique globally
* INACTIVE user cannot perform operations

---

### Group

```
id: UUID PK
name: string NOT NULL
description: string NULL
owner_id: UUID FK users NOT NULL
created_at: datetime NOT NULL
updated_at: datetime NOT NULL
deleted_at: datetime NULL (soft delete)
```

Rules:

* exactly 1 owner
* owner must be member
* deleted group cannot be restored

---

### GroupMember

```
id: UUID PK
group_id: UUID FK groups NOT NULL
user_id: UUID FK users NOT NULL
role: enum(GroupRole) NOT NULL
joined_at: datetime NOT NULL
```

Rules:

* unique (group_id, user_id)
* max 10 members per group
* role required

---

### Invite

```
id: UUID PK
group_id: UUID FK groups NOT NULL
token: string UNIQUE NOT NULL
role: enum(GroupRole) NOT NULL
email: string NULL
expires_at: datetime NOT NULL
created_by: UUID FK users NOT NULL
created_at: datetime NOT NULL
```

Rules:

* expired invite invalid

---

### Sprint

```
id: UUID PK
group_id: UUID FK groups NOT NULL
name: string NOT NULL
goal: string NULL
start_date: date NOT NULL
end_date: date NOT NULL
status: enum(SprintStatus) NOT NULL
velocity_work: int NULL
velocity_estimate: float NULL
created_at: datetime NOT NULL
updated_at: datetime NOT NULL
```

Rules:

* start_date < end_date
* duration ≤ 30 days
* only 1 ACTIVE sprint per group
* editable only in DRAFT
* max 250 works per sprint
* cannot modify after COMPLETED

---

### Work

```
id: UUID PK
group_id: UUID FK groups NOT NULL
sprint_id: UUID FK sprints NULL
name: string NOT NULL
description: string NULL
status: enum(WorkStatus) NOT NULL
assignee_id: UUID FK users NULL
creator_id: UUID FK users NOT NULL
estimate_hours: float NULL (>0)
story_point: int NULL (>0, fibonacci)
priority: enum(WorkPriority) NULL
due_date: date NULL
created_at: datetime NOT NULL
updated_at: datetime NOT NULL
```

Rules:

* must belong to 1 group
* belongs to max 1 sprint
* assignee must be group member
* cannot modify if sprint COMPLETED

NULL sprint_id = backlog

---

### ChecklistItem

```
id: UUID PK
work_id: UUID FK works NOT NULL
name: string NOT NULL
is_completed: bool NOT NULL
created_at: datetime NOT NULL
updated_at: datetime NOT NULL
```

Rules:

* belongs to 1 work
* cascade delete on work delete

---

### Comment

```
id: UUID PK
work_id: UUID FK works NOT NULL
creator_id: UUID FK users NOT NULL
content: string NOT NULL
created_at: datetime NOT NULL
updated_at: datetime NOT NULL
```

Rules:

* creator can edit
* creator, manager, owner can delete

---

## 2.2 Enums

UserStatus:

```
ACTIVE
INACTIVE
```

GroupRole:

```
OWNER
MANAGER
MEMBER
VIEWER
```

SprintStatus:

```
DRAFT
ACTIVE
COMPLETED
CANCELLED
```

WorkStatus:

```
TODO
IN_PROGRESS
IN_REVIEW
DONE
```

WorkPriority:

```
LOW
MEDIUM
HIGH
```

---

## 2.3 Aggregate Boundaries

Group Aggregate:

* Group
* GroupMember
* Invite
* Sprint (root reference)

Sprint Aggregate:

* Sprint
* Work (within sprint)

Work Aggregate:

* Work
* ChecklistItem
* Comment

---

## 2.4 Transaction Boundaries

Transaction required when modifying multiple entities.

Create Work:

```
Work
ChecklistItem (optional)
```

Assign Work:

```
Work
validate GroupMember
```

Update Work Status:

```
Work
```

Delete Work:

```
Work
ChecklistItem (cascade)
Comment (cascade)
```

Complete Sprint:

```
Sprint
Work (move incomplete → backlog)
update velocity
```

Join Group:

```
GroupMember
Invite (invalidate)
```

Remove Member:

```
GroupMember
Work (unassign)
```

---

# 3. DTO Definition

Defines API request/response models.

---

## 3.1 Request DTO

CreateGroupRequest

```
name: string REQUIRED
description: string OPTIONAL
```

UpdateGroupRequest

```
name: optional
description: optional

constraint: ≥1 field required
```

InviteMemberRequest

```
role REQUIRED
email OPTIONAL
```

UpdateMemberRoleRequest

```
userId REQUIRED
role REQUIRED
```

RemoveMemberRequest

```
userId REQUIRED
```

CreateSprintRequest

```
name REQUIRED
goal OPTIONAL
startDate REQUIRED
endDate REQUIRED
```

UpdateSprintRequest

```
name optional
goal optional
startDate optional
endDate optional

constraint: sprint.status == DRAFT
```

CreateWorkRequest

```
name REQUIRED
description optional
assigneeId optional
estimateHours optional
storyPoint optional
priority optional
dueDate optional
```

UpdateWorkRequest

```
name optional
description optional
assigneeId optional/null
estimateHours optional
storyPoint optional
priority optional
dueDate optional
status optional
sprintId optional
```

CreateChecklistItemRequest

```
name REQUIRED
```

UpdateChecklistItemRequest

```
name optional
isCompleted optional

constraint ≥1 field required
```

CreateCommentRequest

```
content REQUIRED
```

UpdateCommentRequest

```
content REQUIRED
```

---

## 3.2 Response DTO

GroupResponse

```
id
name
description
ownerId
createdAt
updatedAt
```

SprintResponse

```
id
groupId
name
goal
startDate
endDate
status
totalWork
completedWork
progressPercent
createdAt
updatedAt
```

WorkResponse

```
id
groupId
sprintId
name
description
status
priority
assigneeId
estimateHours
storyPoint
dueDate
isOverdue (computed)
createdAt
updatedAt
```

ChecklistItemResponse

```
id
workId
name
isCompleted
createdAt
updatedAt
```

CommentResponse

```
id
workId
creatorId
content
createdAt
updatedAt
```

---

# 4. Database Schema

Relational schema optimized for:

* permission filtering
* sprint queries
* backlog queries
* assignment queries
* realtime updates

---

## Tables

users
groups
group_members
invites
sprints
works
checklist_items
comments

---

## Key Constraints

FK integrity enforced

Cascade delete:

```
checklist_items → works
comments → works
```

Unique constraints:

```
users.email
group_members(group_id, user_id)
invites.token
```

Partial indexes:

```
active sprint per group
backlog works (sprint_id NULL)
non-deleted groups
```

---

## Critical Indexes

Permission filtering:

```
group_members(group_id, user_id)
works(group_id)
```

Sprint queries:

```
sprints(group_id, status)
works(sprint_id)
```

Backlog queries:

```
works(group_id WHERE sprint_id IS NULL)
```

Assignment:

```
works(assignee_id)
```

Realtime ordering:

```
comments(work_id, created_at)
```

Overdue detection:

```
works(due_date)
```

---

## Migration Strategy

Single migration file:

```
000001_init_schema.sql
```

Execution order:

```
users
groups
group_members
invites
sprints
works
checklist_items
comments
indexes
```

Execution mode:

```
BEGIN;
CREATE TABLE ...
CREATE INDEX ...
COMMIT;
```

Atomic migration required.

---

# END OF COMPRESSED LLD
