### 9. Event Design

#### 9.1 Queue Definition

### Notification Event Queue

- Exchange: notification.team.exchange
- RoutingKey: notification.team
- Queue: notification.team.queue

### Team Service -> AI Service Queue

- Exchange: ai.team.exchange
- RoutingKey: ai.team.sprint-generation
- Queue: ai.team.sprint-generation.queue

### AI Service -> Team Service Queue

- Exchange: team.ai.exchange
- RoutingKey: team.ai.sprint-generation-result
- Queue: team.ai.sprint-generation-result.queue

#### 9.2 Event Schema

**9.2.1 Notification Event Schema**

| Field Name | Description | Validation |
| :---- | :---- | :---- |
| event_type | Tên loại sự kiện để Notification Service phân loại logic xử lý. | String. Bắt buộc. Phải thuộc tập giá trị định nghĩa. |
| sender_id | ID của người thực hiện hành động tạo ra sự kiện này. | String (UUID/ObjectID). Bắt buộc. |
| receiver_ids | Danh sách các ID người dùng sẽ nhận được thông báo này. | Array of Strings. Không được rỗng. |
| payload.title | Tiêu đề ngắn gọn của thông báo hiển thị trên UI/Push. | String. Tối đa 100 ký tự. Không được để trống. |
| payload.message | Nội dung chi tiết của thông báo. | String. Tối đa 500 ký tự. |
| payload.correlation_id | ID của đối tượng chính liên quan đến sự kiện. | String (UUID/ObjectID). Bắt buộc. |
| payload.correlation_type | Phân loại đối tượng. | Integer/Int32. Bắt buộc. |
| payload.link | Đường dẫn tương đối để click vào thông báo. | String. Phải bắt đầu bằng /. |
| payload.img_url | Link ảnh đại diện hoặc icon hành động. | String URL. Optional. |
| metadata.is_send_mail | Cờ đánh dấu có cần gửi thêm email hay không. | Boolean. Default false. |

Example JSON:

```json
{
	"event_type": "MEMBER_JOINED",
	"sender_id": "user-uuid-123",
	"receiver_ids": ["owner-uuid-001", "manager-uuid-002"],
	"payload": {
		"title": "Thành viên mới",
		"message": "Nguyễn Văn A đã tham gia nhóm 'Phát triển Backend'",
		"correlation_id": "group-uuid-999",
		"correlation_type": 1,
		"link": "/groups/group-uuid-999/members",
		"img_url": "https://api.com/avatars/user-123.jpg"
	},
	"metadata": {
		"is_send_mail": false
	}
}
```

**9.2.2 Team Service -> AI Service Event Schema**

| Field Name | Description | Validation |
| :---- | :---- | :---- |
| event_type | Loại job gửi sang AI Service. | String. Bắt buộc. Ví dụ: SPRINT_GENERATION_REQUESTED. |
| job_id | ID của job sinh sprint. | String (UUID). Bắt buộc. |
| group_id | ID của group đang yêu cầu generate sprint. | String (UUID). Bắt buộc. |
| sender_id | ID người khởi tạo request. | String (UUID/ObjectID). Bắt buộc. |
| payload.sprint | Thông tin sprint đầu vào. | Object. Bắt buộc. Bao gồm các field con bắt buộc bên dưới. |
| payload.sprint.name | Tên sprint. | String. Bắt buộc. |
| payload.sprint.goal | Mục tiêu sprint. | String. Tùy chọn. |
| payload.sprint.start_date | Ngày bắt đầu sprint. | String date. Bắt buộc. |
| payload.sprint.end_date | Ngày kết thúc sprint. | String date. Bắt buộc. |
| payload.files | Danh sách file metadata để AI download từ R2. | Array. Bắt buộc. Tối đa 3 items. |
| payload.files[].object_key | Key file trên R2. | String. Bắt buộc. |
| payload.files[].size | Kích thước file. | Number. Bắt buộc. Mỗi file <= 4MB, tổng <= 12MB. |
| payload.additional_context | Ngữ cảnh bổ sung cho AI. | String. Optional. |

**9.2.3 AI Service -> Team Service Event Schema**

| Field Name | Description | Validation |
| :---- | :---- | :---- |
| event_type | Loại kết quả trả về từ AI Service. | String. Bắt buộc. Ví dụ: SPRINT_GENERATION_COMPLETED. |
| job_id | ID của job sinh sprint. | String (UUID). Bắt buộc. |
| group_id | ID của group liên quan. | String (UUID). Bắt buộc. |
| sender_id | ID của AI Service hoặc system sender. | String. Bắt buộc. |
| payload.sprint | Thông tin sprint đã reconcile để lưu DB. | Object. Bắt buộc. |
| payload.tasks | Danh sách tasks đã generate. | Array. Bắt buộc. |
| payload.status | Trạng thái xử lý. | String. Bắt buộc. Ví dụ: SUCCESS, FAILED. |
| payload.error | Lỗi nếu có. | Object. Optional. |

#### 9.3 Events Design

| Event Type | Purpose | Sender | Receiver | Require Notification |
| :---- | :---- | :---- | :---- | :---- |
| NOTIFICATION_EVENT | Gửi thông báo cho user qua Notification Service. | Team Service / AI Service / Domain Service | Notification Service | Yes |
| SPRINT_GENERATION_REQUESTED | Team Service gửi job sang AI Service để chạy pipeline generate sprint. | Team Service | AI Service | No |
| SPRINT_GENERATION_COMPLETED | AI Service trả kết quả để Team Service tạo sprint và lưu DB. | AI Service | Team Service | Yes |

**9.3.1 Notification Event**

Use the schema in section 9.2.1.

Recommended usage for AI Sprint Generation:

- Success: title = "Sprint mới đã sẵn sàng", message = "Sprint '[Sprint Name]' đã được tạo thành công."
- Error: title = "Sprint generation failed", message = "Không thể tạo sprint. Vui lòng thử lại sau."

**Recommended Payload Model**

| Field Name | Type | Required | Description |
| :---- | :---- | :---- | :---- |
| event_type | string | Yes | Loại sự kiện notification. |
| sender_id | string | Yes | ID người tạo sự kiện. |
| receiver_ids | string[] | Yes | Danh sách user nhận thông báo. |
| payload.title | string | Yes | Tiêu đề ngắn của notification. |
| payload.message | string | Yes | Nội dung notification. |
| payload.correlation_id | string | Yes | ID đối tượng gốc liên quan. |
| payload.correlation_type | int32 | Yes | Loại đối tượng gốc. |
| payload.link | string | Yes | Link điều hướng tương đối. |
| payload.img_url | string | No | Ảnh đại diện hoặc icon. |
| metadata.is_send_mail | bool | No | Cờ gửi email bổ sung. |

**9.3.2 Team Service -> AI Service Event**

Purpose:

- Team Service publish job payload to AI Service queue.
- AI Service consumes job, downloads files from R2, and runs pipeline.

Recommended payload:

**Recommended Payload Model**

| Field Name | Type | Required | Description |
| :---- | :---- | :---- | :---- |
| event_type | string | Yes | Loại event gửi sang AI Service. |
| job_id | string (UUID) | Yes | ID của job generation. |
| group_id | string (UUID) | Yes | ID của group đang generate sprint. |
| sender_id | string | Yes | ID user khởi tạo request. |
| payload.sprint | object | Yes | Sprint input object. |
| payload.sprint.name | string | Yes | Tên sprint. |
| payload.sprint.goal | string | No | Mục tiêu sprint. |
| payload.sprint.start_date | string (date) | Yes | Ngày bắt đầu sprint. |
| payload.sprint.end_date | string (date) | Yes | Ngày kết thúc sprint. |
| payload.files | object[] | Yes | Danh sách file metadata. |
| payload.files[].object_key | string | Yes | Key file trên R2. |
| payload.files[].size | number | Yes | Kích thước file, tối đa 4MB mỗi file. |
| payload.additional_context | string | No | Ngữ cảnh bổ sung cho AI. |

```json
{
	"event_type": "SPRINT_GENERATION_REQUESTED",
	"job_id": "job-uuid-123",
	"group_id": "group-uuid-999",
	"sender_id": "user-uuid-001",
	"payload": {
		"sprint": {
			"name": "Sprint 24",
			"goal": "Hoan thanh AI sprint generation",
			"start_date": "2026-04-01",
			"end_date": "2026-04-14"
		},
		"files": [
			{
				"object_key": "uploads/group-uuid-999/doc-1.md",
				"size": 1024000
			}
		],
		"additional_context": "Uu tien tasks cho backend va validation"
	}
}
```

**9.3.3 AI Service -> Team Service Event**

Purpose:

- AI Service returns structured sprint data and generated tasks.
- Team Service persists sprint and tasks to DB.
- Team Service may publish notification event after save succeeds or fails.

Recommended payload:

**Recommended Payload Model**

| Field Name | Type | Required | Description |
| :---- | :---- | :---- | :---- |
| event_type | string | Yes | Loại event trả kết quả từ AI. |
| job_id | string (UUID) | Yes | ID của job generation. |
| group_id | string (UUID) | Yes | ID của group liên quan. |
| sender_id | string | Yes | ID của AI Service hoặc system sender. |
| payload.status | string | Yes | Trạng thái xử lý, ví dụ SUCCESS hoặc FAILED. |
| payload.sprint | object | Yes | Sprint data đã generate để lưu DB. |
| payload.tasks | object[] | Yes | Danh sách tasks đã generate. |
| payload.tasks[].name | string | Yes | Tên task. |
| payload.tasks[].description | string | Yes | Mô tả task. |
| payload.tasks[].priority | string \| null | No | Độ ưu tiên của task. |
| payload.tasks[].story_point | number \| null | No | Story point của task. |
| payload.tasks[].due_date | string \| null | No | Ngày đến hạn nếu có. |
| payload.error | object | No | Thông tin lỗi nếu pipeline thất bại. |

```json
{
	"event_type": "SPRINT_GENERATION_COMPLETED",
	"job_id": "job-uuid-123",
	"group_id": "group-uuid-999",
	"sender_id": "ai-service",
	"payload": {
		"status": "SUCCESS",
		"sprint": {
			"name": "Sprint 24",
			"goal": "Hoan thanh AI sprint generation",
			"start_date": "2026-04-01",
			"end_date": "2026-04-14"
		},
		"tasks": [
			{
				"name": "Implement Extraction Agent",
				"description": "Process one file per AI request",
				"priority": "HIGH",
				"story_point": 5,
				"due_date": null
			}
		]
	}
}
```

**9.4 Operation Flow**

1. User starts AI Sprint Generation in Frontend.
2. Team Service validates request and publishes SPRINT_GENERATION_REQUESTED.
3. AI Service consumes the job, downloads files from R2, and runs the pipeline.
4. AI Service publishes SPRINT_GENERATION_COMPLETED back to Team Service.
5. Team Service saves sprint and tasks to DB.
6. Team Service publishes NOTIFICATION_EVENT for success or failure.

