# DynamoDB — Transactions & ACID (Giao Dịch & ACID)

> DynamoDB hỗ trợ ACID transactions (Giao Dịch ACID) kể từ 2018, cho phép thực hiện các thao tác nguyên tử trên nhiều items và bảng. Tuy nhiên có những giới hạn và trade-offs quan trọng cần hiểu.

## 📚 Mục Lục

1. [ACID trong DynamoDB](#acid-trong-dynamodb)
2. [TransactWriteItems — Ghi Nguyên Tử](#transactwriteitems--ghi-nguyên-tử)
3. [TransactGetItems — Đọc Nhất Quán](#transactgetitems--đọc-nhất-quán)
4. [Giới Hạn Transactions](#giới-hạn-transactions)
5. [Chi Phí Transactions](#chi-phí-transactions)
6. [Idempotency — Tính Bất Biến](#idempotency--tính-bất-biến)
7. [Conditional Writes vs Transactions](#conditional-writes-vs-transactions)
8. [Use Cases Thực Tế](#use-cases-thực-tế)
9. [Hạn Chế & Khi Không Nên Dùng](#hạn-chế--khi-không-nên-dùng)
10. [Câu Hỏi Phỏng Vấn](#câu-hỏi-phỏng-vấn)

---

## ACID trong DynamoDB

### ACID là gì?

ACID là viết tắt của bốn thuộc tính đảm bảo tính tin cậy của database transactions:

```
A — Atomicity (Tính Nguyên Tử)
    Toàn bộ transaction thành công hoặc toàn bộ thất bại.
    Không có "thành công một nửa".

C — Consistency (Tính Nhất Quán)
    Database luôn ở trạng thái hợp lệ trước và sau transaction.
    Ví dụ: tổng tiền trong tài khoản không thay đổi sau khi transfer.

I — Isolation (Tính Cô Lập)
    Transactions đồng thời không ảnh hưởng lẫn nhau.
    DynamoDB dùng serializable isolation (cô lập có thể tuần tự hóa).

D — Durability (Tính Bền Vững)
    Sau khi commit thành công, data được lưu vĩnh viễn.
    DynamoDB sao chép qua 3 AZ — đảm bảo durability.
```

### Trước 2018: Không Có Native Transactions

Trước khi DynamoDB hỗ trợ transactions, developer phải implement thủ công:

```python
# ❌ Cách cũ: Không atomic, có thể partial failure (thất bại một phần)
table.update_item(Key={'AccountId': 'acc-001'},
                  UpdateExpression='SET Balance = Balance - :amount',
                  ExpressionAttributeValues={':amount': Decimal('100')})

# Nếu crash ở đây → tiền bị trừ nhưng chưa cộng!
table.update_item(Key={'AccountId': 'acc-002'},
                  UpdateExpression='SET Balance = Balance + :amount',
                  ExpressionAttributeValues={':amount': Decimal('100')})
```

### Từ 2018: TransactWriteItems & TransactGetItems

```python
# ✅ Cách mới: Atomic — cả hai thành công hoặc cả hai thất bại
dynamodb.transact_write(Items=[
    {
        'Update': {
            'TableName': 'Accounts',
            'Key': {'AccountId': {'S': 'acc-001'}},
            'UpdateExpression': 'SET Balance = Balance - :amount',
            'ExpressionAttributeValues': {':amount': {'N': '100'}},
            'ConditionExpression': 'Balance >= :amount'  # Đảm bảo đủ tiền
        }
    },
    {
        'Update': {
            'TableName': 'Accounts',
            'Key': {'AccountId': {'S': 'acc-002'}},
            'UpdateExpression': 'SET Balance = Balance + :amount',
            'ExpressionAttributeValues': {':amount': {'N': '100'}}
        }
    }
])
```

---

## TransactWriteItems — Ghi Nguyên Tử

### Cú Pháp Đầy Đủ

`TransactWriteItems` cho phép thực hiện tối đa **100 actions** trong một transaction, trên **tối đa 10 bảng khác nhau**.

```python
import boto3
from boto3.dynamodb.conditions import Attr

dynamodb = boto3.client('dynamodb')

response = dynamodb.transact_write(Items=[
    # 1. Put — tạo hoặc replace item
    {
        'Put': {
            'TableName': 'Orders',
            'Item': {
                'UserId':    {'S': 'user-001'},
                'OrderId':   {'S': 'ord-001'},
                'Status':    {'S': 'PENDING'},
                'TotalAmount': {'N': '150.00'}
            },
            'ConditionExpression': 'attribute_not_exists(OrderId)'
        }
    },

    # 2. Update — cập nhật attributes
    {
        'Update': {
            'TableName': 'Inventory',
            'Key': {'ProductId': {'S': 'prod-001'}},
            'UpdateExpression': 'SET Stock = Stock - :qty',
            'ExpressionAttributeValues': {':qty': {'N': '1'}, ':zero': {'N': '0'}},
            'ConditionExpression': 'Stock >= :qty'  # Đảm bảo còn hàng
        }
    },

    # 3. Delete — xóa item
    {
        'Delete': {
            'TableName': 'CartItems',
            'Key': {'UserId': {'S': 'user-001'}, 'ProductId': {'S': 'prod-001'}},
        }
    },

    # 4. ConditionCheck — kiểm tra điều kiện mà không thay đổi data
    {
        'ConditionCheck': {
            'TableName': 'Users',
            'Key': {'UserId': {'S': 'user-001'}},
            'ConditionExpression': 'AccountStatus = :active',
            'ExpressionAttributeValues': {':active': {'S': 'ACTIVE'}}
        }
    }
])
```

### Các Actions trong TransactWrite

| Action           | Mô Tả                                                          |
| ---------------- | -------------------------------------------------------------- |
| `Put`            | Tạo hoặc thay thế item (như PutItem nhưng trong transaction)   |
| `Update`         | Cập nhật attributes của item (như UpdateItem)                  |
| `Delete`         | Xóa item (như DeleteItem)                                      |
| `ConditionCheck` | Kiểm tra điều kiện mà không thay đổi data — "read-for-write"   |

### Xử Lý Lỗi TransactWrite

```python
from botocore.exceptions import ClientError

try:
    dynamodb.transact_write(Items=[...])

except ClientError as e:
    error_code = e.response['Error']['Code']

    if error_code == 'TransactionCanceledException':
        # Một hoặc nhiều conditions thất bại
        reasons = e.response['CancellationReasons']
        for i, reason in enumerate(reasons):
            if reason['Code'] != 'None':
                print(f"Action {i} failed: {reason['Code']} - {reason['Message']}")
                # Codes: ConditionalCheckFailed, ItemCollectionSizeLimitExceeded,
                #        ProvisionedThroughputExceeded, ThrottlingError, etc.

    elif error_code == 'TransactionConflictException':
        # Cùng item đang được modify bởi transaction khác
        # Retry sau vài ms
        time.sleep(0.1)
        retry_transaction()

    elif error_code == 'IdempotentParameterMismatchException':
        # ClientRequestToken đã dùng nhưng với parameters khác
        print("Same ClientRequestToken with different parameters!")
```

---

## TransactGetItems — Đọc Nhất Quán

`TransactGetItems` đọc nhiều items từ nhiều bảng với **strongly consistent reads** đảm bảo tất cả items phản ánh state tại cùng một thời điểm.

```python
response = dynamodb.transact_get(TransactItems=[
    {
        'Get': {
            'TableName': 'Accounts',
            'Key': {'AccountId': {'S': 'acc-001'}}
        }
    },
    {
        'Get': {
            'TableName': 'Accounts',
            'Key': {'AccountId': {'S': 'acc-002'}}
        }
    }
])

items = response['Responses']
account_1 = items[0]['Item']  # None nếu không tìm thấy
account_2 = items[1]['Item']

# Cả hai items được đọc tại cùng một điểm nhất quán
```

**Khi nào dùng TransactGet:**
- Cần đọc nhiều items và đảm bảo chúng consistent với nhau
- Ví dụ: đọc số dư của 2 tài khoản trước khi quyết định transfer

---

## Giới Hạn Transactions

```
┌───────────────────────────────────────────────────────────────┐
│                    Transaction Limits                          │
├─────────────────────────────┬─────────────────────────────────┤
│ Số actions tối đa           │ 100 actions mỗi transaction     │
│ Tổng data tối đa            │ 4 MB (tổng size tất cả items)   │
│ Số bảng tối đa              │ Không giới hạn (nhưng thực tế   │
│                             │ bị giới hạn bởi 100 actions)    │
│ Thời gian timeout           │ Không có timeout riêng          │
│                             │ (bị giới hạn bởi Lambda timeout)|
│ Cross-region                │ ❌ Không hỗ trợ                 │
└─────────────────────────────┴─────────────────────────────────┘
```

### Không Hỗ Trợ Cross-Region

```
Transaction KHÔNG THỂ span qua:
❌ Nhiều AWS Regions
❌ DynamoDB Global Tables (các replicas)

→ Chỉ trong cùng Region và cùng account
→ Đây là giới hạn quan trọng nhất của DynamoDB transactions
```

---

## Chi Phí Transactions

Transactions tốn gấp đôi RCU/WCU so với thao tác thông thường:

```
Standard Write:       1 WCU per 1 KB
Transactional Write:  2 WCU per 1 KB  (gấp đôi!)

Standard Read (strongly consistent):  1 RCU per 4 KB
Transactional Read:                   2 RCU per 4 KB  (gấp đôi!)

Lý do: DynamoDB phải thực hiện 2 pha (Two-Phase Commit — Cam Kết Hai Pha):
Pha 1: Prepare (Chuẩn Bị) — lock items, check conditions
Pha 2: Commit (Cam Kết) — apply changes
```

### Ví Dụ Tính Chi Phí

```
Transaction: 3 writes, mỗi item 2 KB

Không dùng transaction:
3 writes × ceil(2KB/1KB) × 1 WCU = 6 WCU

Dùng transaction:
3 writes × ceil(2KB/1KB) × 2 WCU = 12 WCU  (gấp đôi!)
```

---

## Idempotency — Tính Bất Biến

### ClientRequestToken

Để đảm bảo transaction idempotent (an toàn khi retry), dùng `ClientRequestToken`:

```python
import uuid

# Mỗi lần tạo order mới, generate unique token
client_token = str(uuid.uuid4())

# Lần gọi đầu tiên
dynamodb.transact_write(
    TransactItems=[...],
    ClientRequestToken=client_token
)

# Nếu nhận timeout/network error và retry:
dynamodb.transact_write(
    TransactItems=[...],
    ClientRequestToken=client_token  # Cùng token!
    # DynamoDB nhận ra đây là retry của transaction đã thành công
    # → Không execute lại, trả về kết quả từ lần đầu
)
```

**Lưu ý:** Token có hiệu lực **10 phút**. Sau 10 phút, DynamoDB coi đây là transaction mới.

---

## Conditional Writes vs Transactions

### Khi Nào Dùng Gì?

```
Conditional Write (single item):
✅ Chỉ cần atomic operation trên 1 item
✅ Rẻ hơn (tốn 1 WCU, không phải 2)
✅ Không cần rollback logic phức tạp

Ví dụ:
table.put_item(
    Item={...},
    ConditionExpression='attribute_not_exists(OrderId)'
)

Transaction (multiple items):
✅ Cần atomic operation trên nhiều items
✅ Cần đảm bảo "tất cả thành công hoặc tất cả thất bại"
✅ Cần ConditionCheck trên item không liên quan

Ví dụ: Transfer tiền giữa 2 accounts (2 updates phải atomic)
```

### Optimistic Locking (Khóa Lạc Quan) vs Transactions

Trước khi có transactions, pattern phổ biến là Optimistic Locking:

```python
# Optimistic Locking với version attribute
def transfer_with_optimistic_lock(from_acc, to_acc, amount, max_retries=3):
    for attempt in range(max_retries):
        # Đọc cả hai accounts
        acc1 = get_account(from_acc)
        acc2 = get_account(to_acc)

        try:
            # Cập nhật với version check
            update_account(from_acc, acc1['Balance'] - amount, acc1['Version'])
            update_account(to_acc, acc2['Balance'] + amount, acc2['Version'])
            return  # Thành công!
        except ConditionalCheckFailedException:
            # Concurrent update — retry
            continue

    raise Exception("Transfer failed after max retries")

# Vấn đề: Không atomic — nếu update đầu thành công nhưng update thứ hai fail
# → Cần implement compensating transaction (giao dịch bù trừ) thủ công
# → Transaction API giải quyết vấn đề này đơn giản hơn
```

---

## Use Cases Thực Tế

### 1. Đặt Hàng (Order Placement)

```python
def place_order(user_id, product_id, quantity, price):
    """Đặt hàng: tạo order + trừ tồn kho + trừ số dư user."""
    order_id = str(uuid.uuid4())
    total = Decimal(str(price * quantity))

    dynamodb.transact_write(TransactItems=[
        # 1. Tạo order mới
        {
            'Put': {
                'TableName': 'Orders',
                'Item': {
                    'UserId': {'S': user_id},
                    'OrderId': {'S': order_id},
                    'Status': {'S': 'CONFIRMED'},
                    'TotalAmount': {'N': str(total)}
                },
                'ConditionExpression': 'attribute_not_exists(OrderId)'
            }
        },
        # 2. Trừ tồn kho (đảm bảo còn hàng)
        {
            'Update': {
                'TableName': 'Inventory',
                'Key': {'ProductId': {'S': product_id}},
                'UpdateExpression': 'SET Stock = Stock - :qty',
                'ExpressionAttributeValues': {
                    ':qty': {'N': str(quantity)},
                    ':zero': {'N': '0'}
                },
                'ConditionExpression': 'Stock >= :qty'
            }
        },
        # 3. Trừ số dư user (đảm bảo đủ tiền)
        {
            'Update': {
                'TableName': 'Wallets',
                'Key': {'UserId': {'S': user_id}},
                'UpdateExpression': 'SET Balance = Balance - :total',
                'ExpressionAttributeValues': {
                    ':total': {'N': str(total)},
                    ':zero': {'N': '0'}
                },
                'ConditionExpression': 'Balance >= :total'
            }
        }
    ])

    return order_id
```

### 2. Transfer Tiền

```python
def transfer_money(from_account, to_account, amount):
    """Chuyển tiền giữa hai tài khoản — atomic."""
    amount_decimal = Decimal(str(amount))

    dynamodb.transact_write(TransactItems=[
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'AccountId': {'S': from_account}},
                'UpdateExpression': 'SET Balance = Balance - :amount',
                'ExpressionAttributeValues': {
                    ':amount': {'N': str(amount_decimal)},
                    ':zero':   {'N': '0'}
                },
                'ConditionExpression': 'Balance >= :amount AND AccountStatus = :active',
                'ExpressionAttributeValues': {
                    ':amount': {'N': str(amount_decimal)},
                    ':zero':   {'N': '0'},
                    ':active': {'S': 'ACTIVE'}
                }
            }
        },
        {
            'Update': {
                'TableName': 'Accounts',
                'Key': {'AccountId': {'S': to_account}},
                'UpdateExpression': 'SET Balance = Balance + :amount',
                'ExpressionAttributeValues': {
                    ':amount': {'N': str(amount_decimal)}
                },
                'ConditionExpression': 'AccountStatus = :active',
                'ExpressionAttributeValues': {
                    ':amount': {'N': str(amount_decimal)},
                    ':active': {'S': 'ACTIVE'}
                }
            }
        }
    ])
```

### 3. Atomic Counter với Unique Constraint

```python
def claim_unique_username(username, user_id):
    """Đăng ký username duy nhất + tạo profile."""
    dynamodb.transact_write(TransactItems=[
        # 1. Claim username (fail nếu đã tồn tại)
        {
            'Put': {
                'TableName': 'Usernames',
                'Item': {
                    'Username': {'S': username},
                    'UserId':   {'S': user_id},
                    'ClaimedAt': {'S': datetime.utcnow().isoformat()}
                },
                'ConditionExpression': 'attribute_not_exists(Username)'
            }
        },
        # 2. Cập nhật profile với username đã claim
        {
            'Update': {
                'TableName': 'Users',
                'Key': {'UserId': {'S': user_id}},
                'UpdateExpression': 'SET Username = :u',
                'ExpressionAttributeValues': {':u': {'S': username}},
                'ConditionExpression': 'attribute_exists(UserId)'
            }
        }
    ])
```

---

## Hạn Chế & Khi Không Nên Dùng

```
❌ Không nên dùng transactions khi:
1. Chỉ cần operation trên 1 item → dùng Conditional Write (rẻ hơn 50%)
2. Cần span nhiều Regions → DynamoDB transactions không hỗ trợ
3. Cần > 100 items trong một transaction → cần chia nhỏ
4. High-throughput, latency-sensitive path → transactions thêm latency
5. Eventual consistency là đủ → không cần transaction overhead

✅ Nên dùng transactions khi:
1. Cần atomic multi-item updates (transfer tiền, đặt hàng)
2. Cần guarantee "tất cả thành công hoặc không cái nào"
3. Cần ConditionCheck trên nhiều items đồng thời
4. Business logic đòi hỏi ACID semantics
```

---

## Câu Hỏi Phỏng Vấn

**Q: DynamoDB có hỗ trợ ACID transactions không? Khác gì với SQL database?**

A: Có, DynamoDB hỗ trợ ACID transactions qua `TransactWriteItems` và `TransactGetItems`. Tuy nhiên khác SQL: (1) giới hạn 100 items và 4 MB mỗi transaction; (2) chỉ trong cùng Region, không cross-region; (3) tốn gấp đôi RCU/WCU; (4) không có deadlock detection như SQL. Dùng khi cần atomicity trên nhiều items (đặt hàng, transfer tiền), nhưng tránh overuse vì chi phí cao hơn.

**Q: Tại sao DynamoDB transactions tốn gấp đôi RCU/WCU?**

A: DynamoDB transactions dùng Two-Phase Commit (Cam Kết Hai Pha): Phase 1 (Prepare) — lock items và kiểm tra conditions; Phase 2 (Commit) — áp dụng thay đổi. Cả hai pha đều tiêu thụ capacity, nên total gấp đôi thao tác thường. Đây là lý do không nên dùng transactions khi chỉ cần operation trên 1 item — Conditional Write cũng atomic nhưng chỉ tốn 1 WCU.

**Q: ClientRequestToken trong TransactWrite dùng để làm gì?**

A: ClientRequestToken đảm bảo idempotency (tính bất biến) khi retry transactions. Nếu gọi TransactWrite với cùng token trong vòng 10 phút, DynamoDB nhận ra đây là retry và trả về kết quả của lần gọi đầu mà không execute lại. Điều này quan trọng trong distributed systems khi network timeout — có thể retry an toàn mà không lo tạo duplicate records.

---

**Cập Nhật Lần Cuối:** 2026-05-15
**Trạng Thái:** ✅ Hoàn Thành
