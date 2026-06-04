# Amazon Bedrock Agents — Tác Nhân AI Tự Chủ

> Bedrock Agents — Tác Nhân AI Tự Chủ là tính năng cho phép Foundation Models tự động lập kế hoạch (plan) và thực hiện chuỗi hành động (multi-step actions) để hoàn thành mục tiêu phức tạp — bằng cách gọi các APIs, truy vấn cơ sở dữ liệu, và tương tác với hệ thống bên ngoài mà không cần lập trình thủ công từng bước.

---

## 🎯 Vấn Đề Agents Giải Quyết

### Giới Hạn Của RAG + Prompt Thông Thường

```
RAG + Prompt thông thường:
User: "Đặt vé máy bay Hà Nội → Sài Gòn ngày 10/7, hạng economy, rẻ nhất"
LLM: [Không thể đặt vé — chỉ trả lời text, không thực hiện action được]

Bedrock Agent:
User: "Đặt vé máy bay Hà Nội → Sài Gòn ngày 10/7, hạng economy, rẻ nhất"
Agent:
  → Bước 1: Gọi search_flights(from="HAN", to="SGN", date="2026-07-10", class="economy")
  → Bước 2: Phân tích kết quả, tìm vé rẻ nhất
  → Bước 3: Gọi book_flight(flight_id="VN123", passenger=user_info)
  → Bước 4: Gọi send_confirmation_email(booking_id="BK001", email=user_email)
  → Trả lời: "Đã đặt vé VN123 khởi hành 07:30, giá 850,000 VND. Xác nhận đã gửi email."
```

---

## 🏗️ Kiến Trúc Bedrock Agent

### Các Thành Phần

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Bedrock Agent                                │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │            Foundation Model (Não Agent)                     │   │
│  │  Hiểu intent → Lập kế hoạch → Quyết định action tiếp theo  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│           │                    │                    │               │
│           ▼                    ▼                    ▼               │
│  ┌──────────────┐  ┌───────────────────┐  ┌──────────────────┐    │
│  │ Action Groups│  │  Knowledge Bases  │  │ Memory (Bộ Nhớ)  │    │
│  │ (Nhóm Hành  │  │  (RAG — Truy Xuất │  │ Session history  │    │
│  │  Động)       │  │  Tài Liệu)        │  │ (Lịch Sử Phiên) │    │
│  │ Lambda / API │  │                   │  │                  │    │
│  └──────────────┘  └───────────────────┘  └──────────────────┘    │
│           │                                                         │
│           ▼                                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Guardrails (Rào Cản Nội Dung)                   │  │
│  │         Content filtering + PII protection                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
         Lambda 1         Lambda 2       External API
       (search DB)    (send email)     (payment gateway)
```

### Orchestration Loop (Vòng Lặp Điều Phối)

Bedrock Agent sử dụng **ReAct** (Reasoning + Acting) pattern:

```
1. User gửi task
         │
         ▼
2. Agent "Suy Nghĩ" (Thought):
   Phân tích yêu cầu, xác định bước tiếp theo
         │
         ▼
3. Agent "Hành Động" (Action):
   Chọn tool và gọi với parameters
         │
         ▼
4. Agent nhận "Quan Sát" (Observation):
   Kết quả từ tool call
         │
         ▼
5. Lặp lại Bước 2-4 cho đến khi đủ thông tin
         │
         ▼
6. Agent trả lời Final Response (Kết Quả Cuối)
```

---

## 🔧 Action Groups (Nhóm Hành Động)

### Action Group Là Gì?

Action Group định nghĩa **tập hợp các actions (hành động)** mà agent có thể thực hiện. Mỗi action group bao gồm:
- **API Schema** (Lược Đồ API): OpenAPI specification mô tả các functions
- **Lambda Function**: Hàm thực thi khi agent chọn action đó

### Tạo Action Group — Ví Dụ Hệ Thống Đặt Hàng

#### Bước 1: Định Nghĩa API Schema (OpenAPI)

```yaml
# order_actions_schema.yaml
openapi: "3.0.0"
info:
  title: Order Management API
  version: "1.0"

paths:
  /check-inventory:
    get:
      operationId: checkInventory
      summary: Kiểm tra tồn kho sản phẩm
      parameters:
        - name: product_id
          in: query
          required: true
          schema:
            type: string
          description: Mã sản phẩm cần kiểm tra
        - name: quantity
          in: query
          required: true
          schema:
            type: integer
          description: Số lượng cần kiểm tra
      responses:
        "200":
          description: Kết quả kiểm tra tồn kho
          content:
            application/json:
              schema:
                type: object
                properties:
                  available:
                    type: boolean
                  stock_count:
                    type: integer
                  warehouse_location:
                    type: string

  /create-order:
    post:
      operationId: createOrder
      summary: Tạo đơn hàng mới
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                product_id:
                  type: string
                quantity:
                  type: integer
                customer_id:
                  type: string
                shipping_address:
                  type: string
              required:
                - product_id
                - quantity
                - customer_id
      responses:
        "200":
          description: Đơn hàng được tạo thành công
          content:
            application/json:
              schema:
                type: object
                properties:
                  order_id:
                    type: string
                  estimated_delivery:
                    type: string
                  total_price:
                    type: number

  /get-order-status:
    get:
      operationId: getOrderStatus
      summary: Lấy trạng thái đơn hàng
      parameters:
        - name: order_id
          in: query
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Trạng thái đơn hàng
```

#### Bước 2: Tạo Lambda Function Xử Lý Actions

```python
import json
import boto3

def lambda_handler(event, context):
    """
    Lambda function xử lý tất cả actions từ Bedrock Agent.
    Agent gửi event với format chuẩn của Bedrock.
    """
    action_group = event["actionGroup"]
    api_path = event["apiPath"]
    http_method = event["httpMethod"]
    parameters = event.get("parameters", [])
    request_body = event.get("requestBody", {})

    # Chuyển parameters list thành dict
    params = {p["name"]: p["value"] for p in parameters}

    # Route đến handler phù hợp
    if api_path == "/check-inventory" and http_method == "GET":
        result = handle_check_inventory(params)
    elif api_path == "/create-order" and http_method == "POST":
        body = json.loads(request_body.get("content", {}).get("application/json", {}).get("body", "{}"))
        result = handle_create_order(body)
    elif api_path == "/get-order-status" and http_method == "GET":
        result = handle_get_order_status(params)
    else:
        result = {"error": f"Unknown action: {api_path}"}

    # Format response theo chuẩn Bedrock Agent
    return {
        "messageVersion": "1.0",
        "response": {
            "actionGroup": action_group,
            "apiPath": api_path,
            "httpMethod": http_method,
            "httpStatusCode": 200,
            "responseBody": {
                "application/json": {
                    "body": json.dumps(result)
                }
            }
        }
    }


def handle_check_inventory(params: dict) -> dict:
    """Kiểm tra tồn kho trong DynamoDB."""
    dynamodb = boto3.resource("dynamodb")
    table = dynamodb.Table("products")

    product_id = params["product_id"]
    required_qty = int(params["quantity"])

    item = table.get_item(Key={"product_id": product_id}).get("Item", {})
    stock = item.get("stock_count", 0)

    return {
        "available": stock >= required_qty,
        "stock_count": stock,
        "warehouse_location": item.get("warehouse", "Hà Nội")
    }


def handle_create_order(body: dict) -> dict:
    """Tạo đơn hàng mới trong DynamoDB."""
    import uuid
    from datetime import datetime, timedelta

    dynamodb = boto3.resource("dynamodb")
    orders_table = dynamodb.Table("orders")

    order_id = f"ORD-{uuid.uuid4().hex[:8].upper()}"
    delivery_date = (datetime.now() + timedelta(days=3)).strftime("%Y-%m-%d")

    order = {
        "order_id": order_id,
        "product_id": body["product_id"],
        "quantity": body["quantity"],
        "customer_id": body["customer_id"],
        "shipping_address": body.get("shipping_address", ""),
        "status": "CONFIRMED",
        "created_at": datetime.now().isoformat()
    }

    orders_table.put_item(Item=order)

    return {
        "order_id": order_id,
        "estimated_delivery": delivery_date,
        "total_price": body["quantity"] * 150000  # Giả định 150,000 VND/unit
    }
```

#### Bước 3: Tạo Agent Qua Python SDK

```python
import boto3
import json
import time

bedrock_agent_client = boto3.client("bedrock-agent", region_name="us-east-1")

def create_order_agent(
    agent_name: str,
    lambda_arn: str,
    role_arn: str,
    schema_s3_uri: str
) -> str:
    """Tạo Bedrock Agent cho hệ thống đặt hàng."""

    # Bước 1: Tạo Agent
    agent_response = bedrock_agent_client.create_agent(
        agentName=agent_name,
        agentResourceRoleArn=role_arn,
        foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
        instruction="""Bạn là trợ lý đặt hàng thông minh cho cửa hàng ABC.
        Nhiệm vụ của bạn:
        1. Kiểm tra tồn kho trước khi nhận đơn
        2. Tạo đơn hàng khi tồn kho đủ
        3. Thông báo tình trạng đơn hàng cho khách

        Luôn xác nhận với khách hàng trước khi thực hiện đặt hàng.
        Trả lời bằng tiếng Việt.""",
        idleSessionTTLInSeconds=1800  # Session timeout sau 30 phút không hoạt động
    )
    agent_id = agent_response["agent"]["agentId"]

    # Bước 2: Tạo Action Group
    bedrock_agent_client.create_agent_action_group(
        agentId=agent_id,
        agentVersion="DRAFT",
        actionGroupName="OrderManagement",
        description="Quản lý đơn hàng: kiểm tra tồn kho, tạo đơn, theo dõi",
        actionGroupExecutor={
            "lambda": lambda_arn
        },
        apiSchema={
            "s3": {
                "s3BucketName": "my-schemas-bucket",
                "s3ObjectKey": "order_actions_schema.yaml"
            }
        }
    )

    # Bước 3: Prepare Agent (Chuẩn Bị Agent — cần thiết sau mỗi thay đổi)
    bedrock_agent_client.prepare_agent(agentId=agent_id)
    time.sleep(5)  # Chờ prepare hoàn thành

    # Bước 4: Tạo Alias (Phiên Bản Stable Để Production)
    alias_response = bedrock_agent_client.create_agent_alias(
        agentId=agent_id,
        agentAliasName="production-v1"
    )

    return agent_id
```

---

## 💬 Gọi Agent Từ Application

```python
import boto3
import uuid

bedrock_agent_runtime = boto3.client("bedrock-agent-runtime", region_name="us-east-1")

def invoke_agent(
    agent_id: str,
    agent_alias_id: str,
    user_message: str,
    session_id: str = None
) -> str:
    """
    Gọi Bedrock Agent và lấy response.

    Args:
        session_id: Dùng cùng session_id để giữ conversation history trong session
    """
    if session_id is None:
        session_id = str(uuid.uuid4())

    response = bedrock_agent_runtime.invoke_agent(
        agentId=agent_id,
        agentAliasId=agent_alias_id,
        sessionId=session_id,
        inputText=user_message,
        enableTrace=True  # Bật trace để debug (xem reasoning steps)
    )

    # Đọc streaming response
    full_response = ""
    trace_steps = []

    for event in response["completion"]:
        if "chunk" in event:
            chunk_bytes = event["chunk"]["bytes"]
            full_response += chunk_bytes.decode("utf-8")
        elif "trace" in event:
            # Trace cho thấy reasoning steps của agent
            trace = event["trace"]["trace"]
            if "orchestrationTrace" in trace:
                orch = trace["orchestrationTrace"]
                if "rationale" in orch:
                    # Model's thinking — suy nghĩ của model
                    trace_steps.append(f"💭 {orch['rationale']['text']}")
                elif "invocationInput" in orch:
                    # Tool being called — tool đang được gọi
                    inv = orch["invocationInput"]
                    if "actionGroupInvocationInput" in inv:
                        action = inv["actionGroupInvocationInput"]
                        trace_steps.append(f"⚡ Calling {action['apiPath']}")

    # In trace để debug (trong production thì log vào CloudWatch)
    for step in trace_steps:
        print(step)

    return full_response


# Ví dụ sử dụng
AGENT_ID = "ABCDEFGHIJ"
AGENT_ALIAS_ID = "TSTALIASID"

session = str(uuid.uuid4())

# Turn 1 (Lượt 1)
response1 = invoke_agent(
    AGENT_ID, AGENT_ALIAS_ID,
    "Tôi muốn đặt 2 chiếc áo thun mã SP001, giao đến 123 Đinh Tiên Hoàng HN",
    session
)
print(f"Agent: {response1}")

# Turn 2 — Agent nhớ context từ Turn 1 (Lượt 2 — Agent nhớ ngữ cảnh từ Lượt 1)
response2 = invoke_agent(
    AGENT_ID, AGENT_ALIAS_ID,
    "Xác nhận đặt hàng",
    session  # Cùng session_id — agent nhớ đơn hàng vừa thảo luận
)
print(f"Agent: {response2}")
```

---

## 🧠 Memory (Bộ Nhớ Agent)

Bedrock Agents hỗ trợ hai loại memory:

### 1. Session Memory (Bộ Nhớ Phiên)

Trong cùng một session (phiên), agent tự động nhớ toàn bộ conversation history:

```
Session 1 (Phiên 1):
  User: "Đặt SP001, 2 cái"
  Agent: [Kiểm tra tồn kho, tạo đơn ORD-ABC123]
  User: "Thêm 1 cái nữa đi"
  Agent: [Nhớ SP001 từ lượt trước, cập nhật đơn]
```

### 2. Long-term Memory (Bộ Nhớ Dài Hạn)

Cho phép agent nhớ thông tin across sessions (qua nhiều phiên khác nhau):

```python
# Bật long-term memory khi tạo agent
bedrock_agent_client.create_agent(
    ...
    memoryConfiguration={
        "enabledMemoryTypes": ["SESSION_SUMMARY"],  # Tóm tắt phiên trước
        "storageDays": 30  # Giữ memory 30 ngày
    }
)

# Khi invoke: agent tự động retrieve (lấy lại) memory của user này
response = bedrock_agent_runtime.invoke_agent(
    ...
    memoryId="user-123",  # ID định danh user, để match memory
)
```

---

## 🛡️ Guardrails cho Agents

```python
# Áp dụng Guardrails để kiểm soát agent behavior
bedrock_agent_client.update_agent(
    agentId=agent_id,
    ...
    guardrailConfiguration={
        "guardrailIdentifier": "my-guardrail-id",
        "guardrailVersion": "1"
    }
)

# Guardrails sẽ:
# - Chặn agent thảo luận về competitors
# - Ẩn số thẻ tín dụng trong output
# - Từ chối các yêu cầu ngoài phạm vi (VD: hỏi về chính trị)
```

---

## 📊 Inline Agents (Agent Nội Tuyến)

**Inline Agents** — Agent Nội Tuyến là cách tạo agent theo cách động (at runtime) mà không cần tạo persistent agent:

```python
def invoke_inline_agent(user_message: str, customer_id: str) -> str:
    """
    Tạo và gọi agent một cách động — không cần pre-create trong console.
    Hữu ích khi mỗi customer cần agent với context riêng.
    """
    response = bedrock_agent_runtime.invoke_inline_agent(
        foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
        instruction=f"""Bạn là trợ lý cá nhân cho khách hàng #{customer_id}.
                       Giúp khách hàng với các vấn đề đơn hàng và sản phẩm.""",
        sessionId=f"session-{customer_id}",
        inputText=user_message,
        actionGroups=[
            {
                "actionGroupName": "CustomerOrders",
                "actionGroupExecutor": {
                    "lambda": "arn:aws:lambda:us-east-1:...:function:order-handler"
                },
                "apiSchema": {
                    "payload": open("schema.yaml").read()
                }
            }
        ]
    )

    full_response = ""
    for event in response["completion"]:
        if "chunk" in event:
            full_response += event["chunk"]["bytes"].decode("utf-8")

    return full_response
```

---

## 🔄 Multi-Agent Collaboration (Cộng Tác Đa Tác Nhân)

Bedrock hỗ trợ **multi-agent patterns** — nhiều agents cộng tác với nhau:

```
Supervisor Agent (Tác Nhân Điều Phối):
  - Nhận request phức tạp
  - Phân tích và giao việc cho sub-agents

Sub-Agent 1: OrderAgent
  - Xử lý đặt hàng, kiểm tra tồn kho

Sub-Agent 2: SupportAgent
  - Xử lý khiếu nại, trả lời FAQ từ Knowledge Base

Sub-Agent 3: AnalyticsAgent
  - Phân tích dữ liệu bán hàng, tạo báo cáo
```

```python
# Supervisor Agent gọi Sub-Agent như một Action
def create_supervisor_agent():
    """Supervisor agent điều phối các sub-agents."""
    bedrock_agent_client.create_agent(
        agentName="SupervisorAgent",
        foundationModel="anthropic.claude-3-5-sonnet-20241022-v2:0",
        instruction="""Bạn là supervisor điều phối hệ thống AI doanh nghiệp.
        Phân tích yêu cầu và chuyển cho sub-agent phù hợp:
        - Vấn đề đặt hàng → OrderAgent
        - Câu hỏi hỗ trợ → SupportAgent
        - Yêu cầu báo cáo → AnalyticsAgent""",
        ...
    )
```

---

## 💡 Best Practices (Thực Hành Tốt Nhất)

### Thiết Kế Agent Instruction

```
✅ Tốt:
"Bạn là trợ lý đặt hàng cho Shop ABC.
Nhiệm vụ: Giúp khách đặt hàng, kiểm tra đơn hàng.
Giới hạn: Chỉ xử lý orders, không tư vấn sản phẩm.
Khi không chắc: Hỏi lại thay vì đoán.
Ngôn ngữ: Tiếng Việt, lịch sự, xưng 'Shop ABC'."

❌ Kém:
"Bạn là AI trợ lý. Hãy giúp đỡ người dùng."
```

### Xử Lý Lỗi Trong Lambda

```python
def lambda_handler(event, context):
    try:
        result = process_action(event)
        return format_success_response(event, result)
    except ValidationError as e:
        # Lỗi validation — agent có thể retry với parameters khác
        return format_error_response(event, f"Dữ liệu không hợp lệ: {e}", 400)
    except ResourceNotFoundError as e:
        # Không tìm thấy resource — agent cần hỏi lại user
        return format_error_response(event, f"Không tìm thấy: {e}", 404)
    except Exception as e:
        # Lỗi không mong đợi — log và trả về lỗi chung
        print(f"Unexpected error: {e}")  # Sẽ vào CloudWatch
        return format_error_response(event, "Lỗi hệ thống, vui lòng thử lại", 500)
```

### Khi Nào Dùng Bedrock Agents

| Dùng Agents Khi | Không Cần Agents Khi |
|---|---|
| Cần thực hiện actions (gọi API, ghi DB) | Chỉ cần trả lời text |
| Task phức tạp nhiều bước | Một bước đơn giản |
| Cần lập kế hoạch động | Workflow cố định |
| Cần tool selection (chọn tool phù hợp) | Luôn dùng cùng một tool |
| Multi-turn conversation với state | Stateless Q&A |

---

## ❓ Câu Hỏi Phỏng Vấn Thường Gặp

**Q: Bedrock Agent khác gì với RAG đơn thuần?**

> RAG chỉ retrieve thông tin rồi generate text. Bedrock Agent có thể **thực hiện actions** (gọi API, ghi database, gửi email), **lập kế hoạch nhiều bước**, và **thích nghi** dựa trên kết quả từng bước. Agent phù hợp với các task cần tương tác với hệ thống bên ngoài.

**Q: Action Group và Lambda tương tác thế nào?**

> Action Group định nghĩa schema (OpenAPI) mô tả các functions có thể gọi. Khi agent quyết định gọi một function, Bedrock tự động gọi Lambda với event chứa action group name, API path, parameters. Lambda xử lý và trả kết quả về cho agent tiếp tục reasoning.

**Q: Làm thế nào để debug agent khi nó đưa ra quyết định sai?**

> Bật `enableTrace=True` khi invoke. Trace log cho thấy từng bước reasoning (suy nghĩ), action được chọn, parameters, và observation (kết quả). Xem trace để hiểu tại sao agent chọn action đó, từ đó cải thiện instruction hoặc action group schema.

**Q: Long-term memory trong agent hoạt động thế nào?**

> Cuối mỗi session, Bedrock tự động tóm tắt conversation và lưu vào memory store (DynamoDB nội bộ). Trong session tiếp theo với cùng `memoryId`, agent retrieve (lấy lại) tóm tắt đó và inject vào context, giúp agent "nhớ" user preferences và lịch sử tương tác.

---

**Cập Nhật Lần Cuối:** 2026-06-03
**Phiên Bản:** 1.0
