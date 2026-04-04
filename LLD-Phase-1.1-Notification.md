### 9\. Event Design

#### 9.1 Queue Definition

- Exchange: notification.team.exchange  
- RoutingKey: notification.team  
- Queue: notification.team.queue

#### 9.1 Event Schema

| Field Name | Description | Validation |
| :---- | :---- | :---- |
| **event\_type** | Tên loại sự kiện (để Notification Service phân loại logic xử lý). | String. Bắt buộc. Phải thuộc tập giá trị định nghĩa (VD: WORK\_STATUS\_CHANGED, MEMBER\_JOINED,...). |
| **sender\_id** | ID của người thực hiện hành động tạo ra sự kiện này. | String (UUID/ObjectID). Bắt buộc. |
| **receiver\_ids** | Danh sách các ID người dùng sẽ nhận được thông báo này. | Array of Strings. Không được rỗng (phải có ít nhất 1 người nhận). |
| **payload.title** | Tiêu đề ngắn gọn của thông báo hiển thị trên UI/Push. | String. Tối đa 100 ký tự. Không được để trống. |
| **payload.message** | Nội dung chi tiết của thông báo. | String. Tối đa 500 ký tự. Nên chứa thông tin biến động (tên task, trạng thái mới...). |
| **payload.correlation\_id** | ID của đối tượng chính liên quan đến sự kiện (Group/Sprint/Work ID). | String (UUID/ObjectID). Bắt buộc để truy vết dữ liệu gốc. |
| **payload.correlation\_type** | Phân loại đối tượng (1: Group, 2: Sprint, 3: Work). | Integer/Int32. Bắt buộc. Dùng để điều hướng logic và icon hiển thị. |
| **payload.link** | Đường dẫn tương đối để người dùng click vào thông báo sẽ đi đến đúng trang. | String. Phải bắt đầu bằng /. VD: /groups/{id}/works/{id}. |
| **payload.img\_url** | Link ảnh đại diện (thường là avatar người gửi hoặc icon hành động). | String (URL format). Tùy chọn (Optional). |
| **metadata.is\_send\_mail** | Cờ đánh dấu sự kiện này có cần gửi thêm Email hay không. | Boolean. Mặc định là false. |

Example Json:  
{  
  "event\_type": "MEMBER\_JOINED",  
  "sender\_id": "user-uuid-123",  
  "receiver\_ids": \["owner-uuid-001", "manager-uuid-002"\],  
  "payload": {  
    "title": "Thành viên mới",  
    "message": "Nguyễn Văn A đã tham gia nhóm 'Phát triển Backend'",  
    "correlation\_id": "group-uuid-999",  
    "correlation\_type": 1,  
    "link": "/groups/group-uuid-999/members",  
    "img\_url": "https://api.com/avatars/user-123.jpg"  
  },  
  "metadata": {  
    "is\_send\_mail": false  
  }  
}

#### 9.2 Event Design

| Event Type | Display Title (Tiêu đề) | Display Message (Nội dung gợi ý) | Receivers | Require Send Mail |
| :---- | :---- | :---- | :---- | :---- |
| **GROUP\_CREATED** | Tạo nhóm thành công | "Nhóm \[Group Name\] đã được tạo thành công." | Owner | **No** |
| **MEMBER\_JOINED** | Thành viên mới | "\[User Name\] đã tham gia vào nhóm \[Group Name\]." | Owner, Managers, Members | **No** |
| **MEMBER\_REMOVED** | Thay đổi nhân sự | "\[User Name\] không còn là thành viên của nhóm \[Group Name\]." | Owner, Managers, Người bị xóa | **Yes** |
| **SPRINT\_CREATED** | Sprint mới đã sẵn sàng | "Một Sprint mới '\[Sprint Name\]' đã được tạo (Draft)." | Owner, Managers | **No** |
| **SPRINT\_ACTIVATED** | Sprint đã bắt đầu | "Sprint '\[Sprint Name\]' đã chính thức bắt đầu. Hãy kiểm tra task của bạn." | Tất cả thành viên | **Yes** |
| **SPRINT\_COMPLETED** | Kết thúc Sprint | "Sprint '\[Sprint Name\]' đã hoàn thành. Hãy kiểm tra báo cáo tiến độ." | Tất cả thành viên | **Yes** |
| **SPRINT\_CANCELLED** | Hủy bỏ Sprint | "Sprint '\[Sprint Name\]' đã bị hủy." | Tất cả thành viên | **No** |
| **WORK\_CREATED** | Công việc mới | "Công việc '\[Work Title\]' vừa được khởi tạo." | Owner, Managers | **No** |
| **WORK\_ASSIGNED** | Giao việc cho bạn | "Bạn đã được giao thực hiện công việc '\[Work Title\]'." | Người được giao (Assignee) | **Yes** |
| **WORK\_STATUS\_CHANGED** | Cập nhật trạng thái | "Công việc '\[Work Title\]' đã chuyển sang trạng thái \[Status\]." | Owner, Managers, Assignee | **No** |
| **WORK\_COMMENTED** | Bình luận mới | "\[User Name\] đã bình luận trong '\[Work Title\]': \[Content\]" | Assignee, Người thảo luận | **No** |

#### 9.3 Operation

Khi một Work được chuyển trạng thái:

1. **Work Service** cập nhật Database.  
2. **Work Service** đẩy sự kiện WORK\_STATUS\_CHANGED vào Exchange của Message Queue.  
3. **Notification Service** nhận tin và gửi Push cho người quản lý.  
4. **Real-time Service** (WebSocket) nhận tin và đẩy trạng thái mới lên màn hình của các thành viên khác đang xem bảng Kanban mà không cần tải lại trang.
