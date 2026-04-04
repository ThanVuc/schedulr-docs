# ✅ COMPACT CONTEXT FOR LLD GENERATION (NO INFORMATION LOSS)

---

# 1. SYSTEM PURPOSE

Hệ thống quản lý công việc theo Group, hỗ trợ:

* Group / Sprint / Work / Checklist / Comment / Notification
* AI Generate Sprint từ planning (text/file)
* AI Chatbot (RAG, read-only)
* Search/filter work theo tên (partial/full match)
* Thông báo realtime khi có event liên quan

Out of scope: LLD, Architecture, Implementation strategy.

---

# 2. ACTORS & ROLES

## Roles

* **Owner**: full quyền, 1/group, không được rời nếu chưa transfer ownership
* **Manager**: quản lý work, sprint, member
* **Member**: CRUD work (trừ delete nếu rule hạn chế)
* **Viewer**: chỉ xem

## Global Rules

* User phải đăng nhập
* Mỗi Group có đúng 1 Owner
* Owner muốn rời phải transfer quyền
* Permission luôn validate theo Group scope

---

# 3. GROUP

## CRUD

* Create → creator thành Owner
* Lưu: name, description, createdAt, createdBy
* Update/Delete theo role
* View list group theo membership

## Member Management

* Invite, remove
* Change role
* Viewer có thể rời group
* Owner không được rời nếu chưa transfer

## Notifications

Trigger khi:

* Join
* Role change
* Remove

---

# 4. SPRINT

## Lifecycle

States: Draft → Active → Completed / Cancelled

* Chỉ Owner/Manager được Activate/Complete/Cancel
* Hard delete chỉ khi Draft
* Không được overlap thời gian trong cùng Group
* Duration ≤ 30 ngày
* StartDate < EndDate

## Work in Sprint

* Add/remove work khi Sprint Draft/Active
* Không add nếu Completed
* Max 250 work
* Khi Sprint Completed → không sửa work

## Metrics

* Progress = % Work Completed
* Velocity tính khi Completed và immutable

## Export

* Owner/Manager export
* Bao gồm overview, work list, workload
* Dữ liệu realtime

## Monitoring

* Warning nếu Draft tồn tại lâu
* Planning guide hỗ trợ scrum

---

# 5. WORK

## Core

* Thuộc 1 Group
* Có thể thuộc Sprint hoặc Backlog (sprintId NULL)
* Default status: Todo
* Workflow:
  Todo → InProgress → InReview → Done
  InReview → InProgress
  (Không bắt buộc approve sang Done)

## Fields

* title (required)
* description (optional)
* status (required)
* assignee (max 1, phải thuộc Group)
* estimate hours (optional)
* story point (optional, positive int)
* dueDate
* createdAt, createdBy
* updatedAt

## Overdue

* dueDate < currentDate AND status ≠ Done
* Là computed attribute

## Constraints

* Không sửa Work nếu Sprint Completed
* Owner/Manager full quyền
* Member: create/view/update
* Viewer: view only

## Delete

* Cascade delete checklist

---

# 6. CHECKLIST

* Work có 0..n checklist items
* Item thuộc duy nhất 1 Work
* Không tồn tại độc lập
* Required: name
* Fields: id, workId, name, isCompleted, createdAt, updatedAt
* Default isCompleted = false
* Chỉ 2 trạng thái: completed/not
* Không có assignee/story point riêng
* Không edit nếu Work Done hoặc Sprint Completed
* Progress Work = % checklist completed
* Chỉ Owner/Manager/Assignee/Creator được edit

---

# 7. BACKLOG

* Là Work có sprintId = NULL
* View theo membership
* Viewer chỉ xem
* Có reorder
* Add/remove sang Sprint

---

# 8. COMMENT

## Rules

* Thuộc 1 Work
* Chỉ member cùng Group được comment
* Required: content
* Fields: creatorId, workId, content, createdAt
* Ordered: oldest first

## Permissions

* Creator: edit/delete own
* Owner/Manager: delete any

## Realtime update supported

---

# 9. NOTIFICATION

## Structure

* type
* correlationId
* createdAt
* isRead (default false)

## Rules

* Chỉ gửi cho member cùng Group
* Actor không nhận notification cho chính hành động của mình
* Chỉ recipient được xem/update
* Không làm thay đổi trạng thái dữ liệu

## Triggers

* Group: join, role change, remove
* Sprint: activate, complete, cancel
* Work:

  * assign / unassign
  * InReview reviewer assigned
  * Done → notify creator
  * Overdue → notify assignee
  * dueDate change → notify assignee
  * delete → notify creator & assignee

---

# 10. AI SPRINT GENERATION

## Access

* Owner/Manager only
* Group scope enforced
* Permission validated trước AI call

## Input

* Manual text OR file upload (.txt, .docx, .pdf, .md)
* Extract content → preview editable
* Required: Sprint Goal, Feature Description, StartDate, EndDate
* Validate: Start < End, duration ≤ 30 days

## AI Processing

AI must:

* Parse thành structured:

  * Sprint name
  * Refined goal (optional)
  * Work list
* Mỗi Work có tối thiểu name
* Generate checklist
* Assign priority: Low/Medium/High
* Không generate story point
* Không auto assign
* Rate limit per user

## Preview & Editing

* Show preview Sprint Draft
* Full edit Sprint/Work/Checklist/Priority
* User confirm trước save
* Có thể cancel

## Persistence

* Atomic transaction:

  1. Parse
  2. Create Sprint (Draft)
  3. Create Works (Todo)
  4. Create Checklist
* Rollback nếu fail
* Không lưu raw AI output (hoặc chỉ lưu dạng draft)
* Must follow all Sprint business rules

## Error Handling

* Nếu parse fail → không tạo Sprint
* Show validation error
* Retry supported

## Security

* Data integrity enforced
* Structured data only
* Audit logging

---

# 11. AI CHATBOT (RAG – FUTURE)

## Access

* Logged-in user
* Chỉ trong Group context
* Permission check trước query

## Capabilities

* Sprint info
* Work info
* Member info
* Progress
* Summary

## Technical Rules

* Phải dùng RAG
* Source: DB (Sprint, Work, Member, Group…)
* Chỉ retrieve data thuộc current Group
* Không truy vấn ngoài scope
* Read-only (không create/update/delete)
* Không trả dữ liệu user không có quyền
* Sanitize & validate input
* Context gửi LLM phải minimal & relevant
* Trả text response
* Nếu không có data → message phù hợp
* Nếu query invalid → error message
* Log toàn bộ query
* Dữ liệu phải latest tại thời điểm query

## Interaction

* Send message
* Receive response
* Show history trong session
* Loading indicator
* Error handling

---

# 12. SYSTEM-WIDE GUARANTEES

* Transaction safety khi create/update quan trọng
* Access control theo Group ở mọi entity
* Data isolation giữa Group
* Audit logging AI actions
* Validation trước persistence
* Real-time update (comment, notification)

---

# TOKEN OPTIMIZATION RESULT

✔ Loại bỏ ~60–70% redundancy
✔ Giữ nguyên toàn bộ:

* State machine
* Validation rules
* Permission matrix
* AI constraints
* Edge cases
* Transaction rules
* Notification triggers
* RAG security model
