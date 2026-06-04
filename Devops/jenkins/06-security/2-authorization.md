# 2 — Authorization: Phân Quyền và Kiểm Soát Truy Cập

> Authorization (phân quyền) xác định "Bạn được làm gì?" sau khi đã xác thực thành công. Jenkins cung cấp nhiều Authorization Strategy (chiến lược phân quyền) từ đơn giản đến tinh vi — từ "mọi người được làm mọi thứ" đến RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò) chi tiết theo từng job.

## Mục Lục

1. [Tổng Quan Authorization Strategy](#1-tổng-quan-authorization-strategy)
2. [Anyone Can Do Anything](#2-anyone-can-do-anything)
3. [Logged-in Users Can Do Anything](#3-logged-in-users-can-do-anything)
4. [Matrix-based Security](#4-matrix-based-security)
5. [Project-based Matrix Authorization](#5-project-based-matrix-authorization)
6. [Role Strategy Plugin (RBAC)](#6-role-strategy-plugin-rbac)
7. [Folder-based Authorization](#7-folder-based-authorization)
8. [So Sánh Các Strategy](#8-so-sánh-các-strategy)
9. [Cấu Hình Thực Tế](#9-cấu-hình-thực-tế)
10. [Troubleshooting Authorization](#10-troubleshooting-authorization)

---

## 1. Tổng Quan Authorization Strategy

**Authorization Strategy** (chiến lược phân quyền) xác định cách Jenkins kiểm tra xem người dùng đã xác thực có được phép thực hiện một hành động cụ thể không.

```
Request: alice muốn TRIGGER build cho job "payments-service"
                      ↓
        Authorization Strategy kiểm tra:
        "alice có quyền Job/Build trên payments-service không?"
                      ↓
          ┌─ CÓ → Cho phép, build được kích hoạt
          └─ KHÔNG → HTTP 403 Forbidden
```

### Các Quyền (Permissions) Cơ Bản

Jenkins định nghĩa permission theo nhóm:

| Nhóm Quyền   | Quyền Cụ Thể              | Mô Tả                                                      |
| ------------ | ------------------------- | ---------------------------------------------------------- |
| **Overall**  | Administer                | Toàn quyền cấu hình Jenkins, cực kỳ nguy hiểm nếu lạm dụng |
|              | Read                      | Xem giao diện Jenkins (cần thiết để login)                 |
|              | RunScripts                | Chạy Groovy script trong Script Console                    |
| **Job**      | Build                     | Kích hoạt build thủ công                                   |
|              | Cancel                    | Dừng build đang chạy                                       |
|              | Configure                 | Sửa cấu hình job                                           |
|              | Create                    | Tạo job mới                                                |
|              | Delete                    | Xóa job                                                    |
|              | Discover                  | "Redirect" khi URL job tồn tại nhưng không có quyền xem    |
|              | Move                      | Di chuyển job sang folder khác                             |
|              | Read                      | Xem job và lịch sử build                                   |
|              | Workspace                 | Xem nội dung workspace của job                             |
| **Run**      | Delete                    | Xóa bản ghi build cũ                                       |
|              | Replay                    | Chạy lại pipeline với script sửa đổi                       |
|              | Update                    | Sửa mô tả build                                            |
| **View**     | Configure, Create, Delete, Read | Quản lý View (cách sắp xếp hiển thị job)            |
| **Credentials** | Create, Delete, ManageDomains, Update, View | Quản lý Credentials                  |
| **Agent**    | Build, Configure, Connect, Create, Delete, Disconnect | Quản lý Agent node       |

---

## 2. Anyone Can Do Anything

**"Anyone Can Do Anything"** (Ai Cũng Làm Được Mọi Thứ) — chế độ không có bảo mật, mọi người (kể cả người dùng ẩn danh) đều có quyền admin.

```
Manage Jenkins → Security → Authorization
→ "Anyone can do anything"
```

### Khi Nào Dùng

- Chỉ dùng trong môi trường **hoàn toàn isolated** (cô lập): localhost, Docker Compose lab không expose ra ngoài
- **Tuyệt đối không dùng trong production**

```yaml
# JCasC
jenkins:
  authorizationStrategy: unsecured
```

---

## 3. Logged-in Users Can Do Anything

**"Logged-in Users Can Do Anything"** (Người Dùng Đã Đăng Nhập Có Thể Làm Mọi Thứ) — user phải đăng nhập, nhưng sau khi đăng nhập có toàn quyền.

```
Manage Jenkins → Security → Authorization
→ "Logged-in users can do anything"
→ ☐ Allow anonymous read access (tắt để bảo mật hơn)
```

### Điểm Chú Ý

- Mọi user hợp lệ đều là admin Jenkins → **nguy hiểm cho team lớn**
- Phù hợp cho team nhỏ (<5 người) tin tưởng nhau hoàn toàn
- Không hỗ trợ phân quyền chi tiết

```yaml
# JCasC
jenkins:
  authorizationStrategy:
    loggedInUsersCanDoAnything:
      allowAnonymousRead: false
```

---

## 4. Matrix-based Security

**Matrix-based Security** (Bảo Mật Dựa Trên Ma Trận) cho phép gán quyền chi tiết cho từng user và group theo bảng ma trận permission × user/group.

### Cấu Hình qua UI

```
Manage Jenkins → Security → Authorization → Matrix-based security

          | Overall         | Job                          | ...
          | Admin | Read    | Build | Cancel | Configure | ...
---------------------------------------------------------------------
admin     |  ✓   |  ✓    |  ✓   |  ✓    |    ✓     | ...
alice     |      |  ✓    |  ✓   |  ✓    |          | ...
bob       |      |  ✓    |  ✓   |       |          | ...
authenticated|   |  ✓    |      |       |          | ...
anonymous |      |       |      |       |          | ...
```

### Cấu Hình JCasC

```yaml
jenkins:
  authorizationStrategy:
    globalMatrix:
      permissions:
        # Admin có toàn quyền
        - "Overall/Administer:admin"
        # DevOps team có quyền build và view
        - "Overall/Read:authenticated"
        - "Job/Build:devops-team"
        - "Job/Cancel:devops-team"
        - "Job/Read:devops-team"
        - "Run/Replay:devops-team"
        # Developer chỉ xem
        - "Job/Read:developers"
        - "Job/Build:developers"
        # Anonymous không có quyền gì (mặc định)
```

### Dùng Group Thay Vì User Cụ Thể

Khi tích hợp LDAP/AD, Jenkins nhận group membership và có thể gán quyền cho group:

```yaml
jenkins:
  authorizationStrategy:
    globalMatrix:
      permissions:
        - "Overall/Administer:jenkins-admins"          # LDAP group
        - "Overall/Read:jenkins-users"                 # LDAP group
        - "Job/Build:jenkins-developers"               # LDAP group
        - "Credentials/View:jenkins-developers"
        - "Job/Configure:jenkins-senior-devs"
```

### Hạn Chế Của Matrix-based Security

- Mọi quyền áp dụng **toàn bộ Jenkins** — không phân biệt job này hay job kia
- Không thể cấp quyền build job A nhưng không cho build job B
- Giải pháp: dùng **Project-based Matrix** hoặc **Role Strategy Plugin**

---

## 5. Project-based Matrix Authorization

**Project-based Matrix Authorization Strategy** (Phân Quyền Dựa Trên Ma Trận Theo Project) mở rộng Matrix-based Security bằng cách cho phép gán quyền khác nhau cho từng job/project cụ thể.

### Cấu Hình

```
Manage Jenkins → Security → Authorization
→ "Project-based Matrix Authorization Strategy"
```

**Sau đó trong mỗi Job:**

```
Job → Configure → Enable project-based security

          | Job                          |
          | Build | Cancel | Configure | Read |
-------------------------------------------------
alice     |  ✓   |  ✓    |    ✓     |  ✓  |
bob       |  ✓   |       |          |  ✓  |
charlie   |      |       |          |  ✓  | (chỉ xem)
```

### Ví Dụ Thực Tế

```
payments-service job:
  → alice (payments lead):  Job/Build, Job/Configure, Job/Read
  → payments-team (group):  Job/Build, Job/Read
  → devops-team (group):    Job/Build, Job/Configure, Job/Read

inventory-service job:
  → bob (inventory lead):   Job/Build, Job/Configure, Job/Read
  → inventory-team (group): Job/Build, Job/Read
  → payments-team:          (không có quyền gì — không thể thấy job này)
```

### Nhược Điểm

- Cấu hình phân tán — mỗi job phải set riêng → khó quản lý khi có hàng trăm job
- Không có giao diện tập trung để xem "user X có quyền gì trên tất cả jobs"

---

## 6. Role Strategy Plugin (RBAC)

**Role Strategy Plugin** (Plugin Chiến Lược Vai Trò) là giải pháp RBAC (Role-Based Access Control — Kiểm Soát Truy Cập Dựa Trên Vai Trò) mạnh mẽ nhất cho Jenkins, cho phép tạo Role (vai trò) và gán user/group vào Role.

### Plugin Cần Thiết

```
Role-based Authorization Strategy (role-strategy)
```

### Kiến Trúc Role Strategy

```
     GLOBAL ROLES                    PROJECT ROLES
  (Áp dụng toàn Jenkins)        (Áp dụng theo pattern job)

  ┌─────────────────┐            ┌─────────────────────────┐
  │  admin          │            │  payments-developer      │
  │  - Overall/Admin│            │  Pattern: payments-.*   │
  │                 │            │  - Job/Build, Job/Read  │
  ├─────────────────┤            ├─────────────────────────┤
  │  developer      │            │  infra-engineer          │
  │  - Overall/Read │            │  Pattern: infra/.*      │
  │  - Job/Read     │            │  - Job/Build, Configure │
  │  - Job/Build    │            └─────────────────────────┘
  │                 │
  ├─────────────────┤            NODE ROLES
  │  viewer         │        (Áp dụng cho Agent node)
  │  - Overall/Read │
  │  - Job/Read     │            ┌─────────────────────────┐
  └─────────────────┘            │  linux-builder           │
                                 │  - Agent/Build          │
  Assign:                        └─────────────────────────┘
  alice   → admin
  bob     → developer
  charlie → viewer
  payments-team → payments-developer (project role)
```

### Cấu Hình qua UI

```
Manage Jenkins → Manage and Assign Roles

1. Manage Roles:
   Global Roles:
   ┌──────────────┬────────────────────────────────────────────┐
   │ Role Name    │ Permissions                                │
   ├──────────────┼────────────────────────────────────────────┤
   │ admin        │ Overall/Administer (tick toàn bộ)          │
   │ developer    │ Overall/Read, Job/Read, Job/Build, Run/*   │
   │ viewer       │ Overall/Read, Job/Read                     │
   │ job-creator  │ Overall/Read, Job/Create, Job/Read         │
   └──────────────┴────────────────────────────────────────────┘

   Project Roles:
   ┌──────────────────┬───────────────────┬────────────────────┐
   │ Role Name        │ Pattern (regex)   │ Permissions        │
   ├──────────────────┼───────────────────┼────────────────────┤
   │ payments-dev     │ payments.*        │ Job/Build, Read    │
   │ infra-dev        │ infra/.*          │ Job/Build, Config  │
   │ mobile-dev       │ (android|ios)/.*  │ Job/Build, Read    │
   └──────────────────┴───────────────────┴────────────────────┘

2. Assign Roles:
   Global Roles:
   User/Group     → Role
   alice          → admin
   bob            → developer
   charlie        → viewer
   jenkins-admins → admin      (LDAP group)
   jenkins-devs   → developer  (LDAP group)

   Project Roles:
   User/Group       → Role
   payments-team    → payments-dev   (LDAP group)
   infra-team       → infra-dev      (LDAP group)
```

### Cấu Hình JCasC

```yaml
jenkins:
  authorizationStrategy:
    roleBased:
      roles:
        global:
          - name: "admin"
            description: "Jenkins administrators"
            permissions:
              - "Overall/Administer"
            assignments:
              - "alice"
              - "jenkins-admins"    # LDAP group
          - name: "developer"
            description: "Developers — build and view"
            permissions:
              - "Overall/Read"
              - "Job/Build"
              - "Job/Cancel"
              - "Job/Read"
              - "Run/Replay"
              - "Run/Update"
              - "View/Read"
            assignments:
              - "jenkins-developers"
          - name: "viewer"
            description: "Read-only access"
            permissions:
              - "Overall/Read"
              - "Job/Read"
              - "View/Read"
            assignments:
              - "jenkins-viewers"
        items:
          - name: "payments-developer"
            description: "Access to payments-* jobs"
            pattern: "payments.*"
            permissions:
              - "Job/Build"
              - "Job/Cancel"
              - "Job/Read"
              - "Run/Replay"
            assignments:
              - "payments-team"
          - name: "infra-engineer"
            description: "Full access to infra jobs"
            pattern: "infra/.*"
            permissions:
              - "Job/Build"
              - "Job/Configure"
              - "Job/Read"
              - "Job/Cancel"
            assignments:
              - "infra-team"
        agents:
          - name: "linux-agent-user"
            description: "Can use Linux build agents"
            pattern: "linux-.*"
            permissions:
              - "Agent/Build"
            assignments:
              - "jenkins-developers"
```

### Best Practices Với Role Strategy

1. **Nguyên tắc Least Privilege** (quyền tối thiểu): chỉ cấp đúng quyền cần thiết
2. **Dùng group, không dùng individual user**: gán LDAP group vào role thay vì từng user cụ thể
3. **Tách Global Role và Project Role**: Global Role chỉ chứa `Overall/Read` cho developer, còn quyền job cụ thể để ở Project Role
4. **Đặt tên role có ý nghĩa**: `payments-developer` tốt hơn `role1`
5. **Định kỳ audit**: kiểm tra ai đang có role gì ít nhất mỗi quý

---

## 7. Folder-based Authorization

Khi dùng **Folders Plugin** (plugin thư mục), có thể kế thừa hoặc ghi đè quyền theo cấu trúc thư mục:

```
Jenkins/
├── payments/        (Folder)
│   ├── Inherits from parent: developers có Job/Read
│   ├── Override: payments-team có Job/Build, Job/Configure
│   ├── payments-api/
│   ├── payments-worker/
│   └── payments-gateway/
│
├── infra/           (Folder)
│   ├── infra-team: Job/Build, Job/Configure, Job/Read
│   ├── kubernetes/
│   └── terraform/
│
└── mobile/          (Folder)
    ├── mobile-team: Job/Build, Job/Read
    ├── android/
    └── ios/
```

### Cấu Hình Folder Security

```
Folder → Configure → Enable project-based security (với Project-based Matrix)
hoặc
Folder → Configure → Child Item Permissions (với Role Strategy + Folders)
```

---

## 8. So Sánh Các Strategy

| Strategy                       | Phức Tạp | Granularity | Scale  | Khuyến Nghị Dùng            |
| ------------------------------ | -------- | ----------- | ------ | --------------------------- |
| Anyone Can Do Anything         | Rất thấp | Không có    | N/A    | Chỉ cho lab isolated        |
| Logged-in Users Can Do Anything| Thấp     | Không có    | Nhỏ   | Team nhỏ, trust 100%        |
| Matrix-based Security          | Trung bình| Global only | Nhỏ-vừa| Team <20 người, ít job     |
| Project-based Matrix           | Cao      | Per-job     | Vừa   | Khi cần phân quyền per-job |
| Role Strategy Plugin           | Trung bình| Role+Pattern| Lớn   | **Khuyến nghị cho production**|

### Quyết Định Nhanh

```
Số user < 5 và trust nhau hoàn toàn?
  → Logged-in Users Can Do Anything

Cần phân quyền nhưng tất cả user có quyền giống nhau?
  → Matrix-based Security

Cần phân quyền khác nhau theo nhóm job/project?
  → Role Strategy Plugin (best practice)

Cần phân quyền theo từng job riêng lẻ (edge case)?
  → Project-based Matrix Authorization
```

---

## 9. Cấu Hình Thực Tế

### Setup Role Strategy Từ Đầu

```groovy
// Groovy script — khởi tạo Role Strategy cơ bản
import com.michelin.cio.hudson.plugins.rolestrategy.*
import com.michelin.cio.hudson.plugins.rolestrategy.Role
import org.jenkinsci.plugins.rolestrategy.*
import jenkins.model.*
import hudson.security.*

def jenkins = Jenkins.getInstance()

// 1. Tạo Role Strategy
def strategy = new RoleBasedAuthorizationStrategy()

// 2. Tạo Global Admin Role
Set<Permission> adminPermissions = new HashSet<>()
adminPermissions.add(Jenkins.ADMINISTER)
Role adminRole = new Role("admin", adminPermissions)

// 3. Tạo Developer Role
Set<Permission> developerPermissions = new HashSet<>()
developerPermissions.add(hudson.model.Item.READ)
developerPermissions.add(hudson.model.Item.BUILD)
developerPermissions.add(Jenkins.READ)
Role developerRole = new Role("developer", developerPermissions)

// 4. Assign roles
strategy.addRole(RoleBasedAuthorizationStrategy.GLOBAL, adminRole)
strategy.addRole(RoleBasedAuthorizationStrategy.GLOBAL, developerRole)
strategy.assignRole(RoleBasedAuthorizationStrategy.GLOBAL, adminRole, "admin")
strategy.assignRole(RoleBasedAuthorizationStrategy.GLOBAL, developerRole, "jenkins-developers")

// 5. Apply
jenkins.setAuthorizationStrategy(strategy)
jenkins.save()
println "Role Strategy configured"
```

### Kiểm Tra Quyền Của User Cụ Thể

```
Manage Jenkins → Manage and Assign Roles → Role Strategy Macros
→ "Who has permission X on item Y?"
```

Hoặc qua API:

```bash
# Kiểm tra permissions của user alice
curl -u admin:apitoken \
  "https://jenkins.example.com/whoAmI/api/json?pretty=true"

# Xem authorities (roles) của user hiện tại
curl -u alice:alicetoken \
  "https://jenkins.example.com/whoAmI/api/json?pretty=true"
```

---

## 10. Troubleshooting Authorization

### Lỗi HTTP 403 — "alice is missing the Job/Build permission"

```
Nguyên nhân:
1. alice chưa được gán role có Job/Build
2. alice được gán role đúng nhưng pattern job không khớp
3. Job có project-based security override cấm alice

Kiểm tra:
Manage Jenkins → Manage and Assign Roles
→ Xem alice được gán role gì
→ Xem role đó có Job/Build không
→ Xem pattern của Project Role có khớp tên job không
```

### Lỗi — "User không thấy job mình được cấp quyền"

```
Nguyên nhân: thiếu Job/Discover hoặc Overall/Read

Sửa: thêm vào role:
- Overall/Read    (xem Jenkins UI)
- Job/Read        (xem danh sách job và build history)
- Job/Discover    (redirect đúng khi URL job hợp lệ)
```

### Audit Quyền Hiện Tại

```bash
# Export toàn bộ cấu hình Role Strategy qua JCasC
curl -u admin:token \
  "https://jenkins.example.com/configuration-as-code/export" \
  | grep -A 100 "authorizationStrategy"
```

### Ghi Log Authorization Decision

```
Manage Jenkins → System Log → Add new log recorder
Logger: hudson.security.AuthorizationStrategy
Level:  FINE
```

---

## Tóm Tắt Nhanh

| Câu Hỏi                                                | Câu Trả Lời                                                  |
| ------------------------------------------------------- | ------------------------------------------------------------ |
| Plugin RBAC tốt nhất cho Jenkins?                      | Role Strategy Plugin                                          |
| Project Role pattern dùng gì?                          | Regular expression (regex) khớp với tên job đầy đủ           |
| User cần quyền gì tối thiểu để đăng nhập?              | Overall/Read                                                  |
| Làm sao hide job với user không có quyền?              | Không gán Job/Read — job sẽ không hiển thị trong danh sách   |
| Matrix-based vs Role Strategy — khác điểm gì chính?   | Role Strategy hỗ trợ pattern per-job và group role tốt hơn   |

---

**Xem tiếp:** [3-credentials.md](3-credentials.md) — Quản lý secret và thông tin xác thực
