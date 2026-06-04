# Virtualization — Ảo Hóa Danh Sách Dài

> **Virtualization** (Ảo Hóa) hay **Windowing** (Cửa Sổ Hiển Thị) là kỹ thuật chỉ render những phần tử **đang hiển thị trong viewport** (vùng nhìn thấy của màn hình). Thay vì tạo 10,000 DOM nodes cho 10,000 phần tử, bạn chỉ tạo ~20-30 nodes và thay thế nội dung khi người dùng scroll.

---

## 📌 Mục Lục

1. [Vấn Đề Với Danh Sách Dài](#1-vấn-đề-với-danh-sách-dài)
2. [Nguyên Lý Hoạt Động](#2-nguyên-lý-hoạt-động)
3. [react-window — Thư Viện Cơ Bản](#3-react-window--thư-viện-cơ-bản)
4. [@tanstack/virtual — Giải Pháp Hiện Đại](#4-tanstackvirtual--giải-pháp-hiện-đại)
5. [Kết Hợp Với Infinite Scroll](#5-kết-hợp-với-infinite-scroll)
6. [Virtual Grid — Lưới Ảo](#6-virtual-grid--lưới-ảo)
7. [Khi Nào Nên Dùng Virtualization](#7-khi-nào-nên-dùng-virtualization)
8. [Câu Hỏi Phỏng Vấn](#8-câu-hỏi-phỏng-vấn)

---

## 1. Vấn Đề Với Danh Sách Dài

### Tại Sao Danh Sách Dài Gây Vấn Đề?

```
Danh sách 10,000 phần tử KHÔNG virtualize:

Bước 1: React tạo 10,000 React elements (object JavaScript)
Bước 2: React tạo 10,000 DOM nodes thực (div, li, img...)
Bước 3: Trình duyệt tính toán layout cho 10,000 nodes → 500ms+
Bước 4: Trình duyệt paint (vẽ) 10,000 nodes → 200ms+
Bước 5: Scroll → browser phải recalculate style/layout lại

→ Initial render: 700ms-2s
→ Scroll: giật lag, FPS (Frames Per Second — Khung Hình Mỗi Giây) < 30
→ Bộ nhớ: 100-300MB chỉ cho DOM nodes
```

### Demo Vấn Đề

```jsx
// ❌ Không virtualize — render tất cả 10,000 phần tử
function SlowProductList({ products }) { // products.length = 10,000
  return (
    <ul>
      {products.map(product => (
        // 10,000 DOM nodes được tạo cùng lúc
        <li key={product.id}>
          <img src={product.thumbnail} alt={product.name} />
          <div>
            <h3>{product.name}</h3>
            <p>{product.description}</p>
            <span>${product.price}</span>
          </div>
        </li>
      ))}
    </ul>
  );
}
// Mỗi <li> có 4 DOM nodes → tổng 40,000 DOM nodes!
// Performance Panel trong Chrome DevTools sẽ thấy: 2-3s "Recalculate Style"
```

### Measurement — Đo Lường Baseline

```javascript
// Đo lường bằng Performance API
performance.mark("list-start");
// Render danh sách...
performance.mark("list-end");
performance.measure("list-render", "list-start", "list-end");

const measure = performance.getEntriesByName("list-render")[0];
console.log(`Render time: ${measure.duration.toFixed(2)}ms`);
```

---

## 2. Nguyên Lý Hoạt Động

### Cơ Chế Windowing

```
Viewport (màn hình): 800px cao
Item height: 50px mỗi phần tử
Visible items: 800/50 = 16 phần tử hiển thị

Virtualization chỉ render:
- 16 phần tử đang hiển thị
- + N phần tử đệm phía trên (overscan)
- + N phần tử đệm phía dưới (overscan)
= ~20-30 phần tử thực sự trong DOM

Khi scroll xuống:
- Các phần tử rời khỏi viewport phía trên → tái sử dụng (recycle)
- Điền nội dung mới của phần tử phía dưới vào
- Container giữ nguyên kích thước "giả" bằng totalHeight

"Container giả":
┌─────────────────────┐  ←─ top: 0
│   (khoảng trắng)    │     height: scrolled_items * item_height
│   (không có DOM)    │
│─────────────────────│  ←─ Bắt đầu vùng render thực
│   Item 150          │
│   Item 151          │     ~20-30 DOM nodes thực sự
│   Item 152          │
│   ...               │
│   Item 170          │
│─────────────────────│  ←─ Kết thúc vùng render thực
│   (khoảng trắng)    │
│   (không có DOM)    │     height: remaining_items * item_height
└─────────────────────┘  ←─ bottom: totalHeight
```

---

## 3. react-window — Thư Viện Cơ Bản

**react-window** là thư viện virtualization nhẹ và phổ biến nhất.

```bash
npm install react-window
```

### FixedSizeList — Danh Sách Chiều Cao Cố Định

```jsx
import { FixedSizeList } from "react-window";

// Mỗi phần tử phải có cùng chiều cao (height)
const ITEM_HEIGHT = 80; // pixel
const VISIBLE_HEIGHT = 600; // chiều cao container

// Row component — phải nhận style prop và gắn vào container ngoài cùng
function ProductRow({ index, style, data }) {
  const product = data.products[index];

  return (
    // style BẮT BUỘC phải có — chứa top/height để định vị đúng chỗ
    <div style={style} className="product-row">
      <img src={product.thumbnail} alt={product.name} width={60} height={60} />
      <div>
        <h3>{product.name}</h3>
        <p>${product.price}</p>
      </div>
    </div>
  );
}

function VirtualProductList({ products }) {
  // itemData giúp truyền dữ liệu xuống Row mà không tạo closure mới mỗi render
  const itemData = useMemo(() => ({ products }), [products]);

  return (
    <FixedSizeList
      height={VISIBLE_HEIGHT} // Chiều cao container (viewport)
      itemCount={products.length} // Tổng số phần tử
      itemSize={ITEM_HEIGHT} // Chiều cao mỗi phần tử (cố định)
      itemData={itemData} // Dữ liệu truyền xuống Row
      width="100%" // Chiều rộng container
      overscanCount={3} // Render thêm 3 items ngoài viewport để scroll mượt hơn
    >
      {ProductRow}
    </FixedSizeList>
  );
}

// Kết quả: 10,000 sản phẩm nhưng chỉ ~20 DOM nodes
```

### VariableSizeList — Danh Sách Chiều Cao Khác Nhau

```jsx
import { VariableSizeList } from "react-window";

// Mỗi phần tử có thể có chiều cao khác nhau
function ChatMessage({ index, style, data }) {
  const message = data.messages[index];
  const isLong = message.text.length > 200;

  return (
    <div style={style}>
      <div className={`message ${message.sender}`}>
        <strong>{message.senderName}</strong>
        <p>{message.text}</p>
        {message.attachments?.map(att => (
          <img key={att.id} src={att.url} alt={att.name} />
        ))}
      </div>
    </div>
  );
}

function ChatList({ messages }) {
  // getItemSize phải trả về chiều cao CHÍNH XÁC cho mỗi index
  // Đây là phần khó nhất của VariableSizeList
  const getItemSize = useCallback((index) => {
    const message = messages[index];
    const baseHeight = 60;
    const textLines = Math.ceil(message.text.length / 50);
    const attachmentHeight = (message.attachments?.length || 0) * 100;
    return baseHeight + textLines * 20 + attachmentHeight;
  }, [messages]);

  const listRef = useRef(null);

  // Reset cached sizes khi messages thay đổi
  useEffect(() => {
    listRef.current?.resetAfterIndex(0);
  }, [messages]);

  return (
    <VariableSizeList
      ref={listRef}
      height={500}
      itemCount={messages.length}
      itemSize={getItemSize}
      itemData={{ messages }}
      width="100%"
    >
      {ChatMessage}
    </VariableSizeList>
  );
}
```

### react-window-infinite-loader — Kết Hợp Infinite Scroll

```bash
npm install react-window-infinite-loader
```

```jsx
import { FixedSizeList } from "react-window";
import InfiniteLoader from "react-window-infinite-loader";
import { useInfiniteQuery } from "@tanstack/react-query";

function InfiniteProductList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ["products"],
    queryFn: ({ pageParam = 0 }) =>
      fetchProducts({ offset: pageParam, limit: 20 }),
    getNextPageParam: (lastPage) => lastPage.nextOffset,
  });

  const allProducts = data?.pages.flatMap(page => page.products) ?? [];
  // Thêm 1 phần tử "giả" nếu còn trang tiếp → dùng để hiển thị loading
  const itemCount = hasNextPage ? allProducts.length + 1 : allProducts.length;

  const isItemLoaded = (index) => !hasNextPage || index < allProducts.length;

  const loadMoreItems = isFetchingNextPage
    ? () => {} // Đang fetch → không cần gọi lại
    : () => fetchNextPage();

  function Item({ index, style }) {
    if (!isItemLoaded(index)) {
      return <div style={style}>Đang tải thêm...</div>;
    }
    const product = allProducts[index];
    return (
      <div style={style}>
        <span>{product.name}</span>
        <span>${product.price}</span>
      </div>
    );
  }

  return (
    <InfiniteLoader
      isItemLoaded={isItemLoaded}
      itemCount={itemCount}
      loadMoreItems={loadMoreItems}
      threshold={5} // Bắt đầu tải khi còn 5 items trước khi hết
    >
      {({ onItemsRendered, ref }) => (
        <FixedSizeList
          ref={ref}
          height={600}
          itemCount={itemCount}
          itemSize={80}
          onItemsRendered={onItemsRendered}
          width="100%"
        >
          {Item}
        </FixedSizeList>
      )}
    </InfiniteLoader>
  );
}
```

---

## 4. @tanstack/virtual — Giải Pháp Hiện Đại

**@tanstack/virtual** (TanStack Virtual) là thư viện virtualization hiện đại hơn, headless — không có UI mặc định, linh hoạt hơn react-window.

```bash
npm install @tanstack/react-virtual
```

### Danh Sách Cơ Bản

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualList({ products }) {
  // parentRef: tham chiếu đến scroll container
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: products.length, // Tổng số phần tử
    getScrollElement: () => parentRef.current, // Scroll container
    estimateSize: () => 80, // Ước tính chiều cao mỗi phần tử (pixel)
    overscan: 5, // Render thêm 5 phần tử ngoài viewport
  });

  return (
    // Scroll container — phải có overflow-y: auto và chiều cao cố định
    <div
      ref={parentRef}
      style={{ height: "600px", overflow: "auto" }}
    >
      {/* Container "giả" — chiều cao = tổng chiều cao tất cả phần tử */}
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
        {/* Chỉ render các phần tử đang "thấy" */}
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            style={{
              position: "absolute",
              top: 0,
              left: 0,
              width: "100%",
              // transform thay vì top để tối ưu GPU rendering
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            <ProductCard product={products[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Dynamic Size — Chiều Cao Tự Động Đo

TanStack Virtual hỗ trợ **dynamic size measurement** (đo kích thước động) — không cần biết trước chiều cao:

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";

function DynamicHeightList({ messages }) {
  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: messages.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 100, // Ước tính ban đầu
    // measureElement: tự động đo chiều cao thực sau khi render
    measureElement:
      typeof window !== "undefined" && navigator.userAgent.indexOf("Firefox") === -1
        ? element => element?.getBoundingClientRect().height
        : undefined,
  });

  return (
    <div ref={parentRef} style={{ height: "500px", overflow: "auto" }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
        {virtualizer.getVirtualItems().map(virtualItem => (
          <div
            key={virtualItem.key}
            // data-index: TanStack Virtual dùng để đo kích thước
            data-index={virtualItem.index}
            ref={virtualizer.measureElement} // Tự động đo sau khi render
            style={{
              position: "absolute",
              top: 0,
              left: 0,
              width: "100%",
              transform: `translateY(${virtualItem.start}px)`,
            }}
          >
            {/* Nội dung có chiều cao không cố định */}
            <ChatMessage message={messages[virtualItem.index]} />
          </div>
        ))}
      </div>
    </div>
  );
}
```

### Scroll To Index — Cuộn Đến Phần Tử Cụ Thể

```jsx
function VirtualListWithNavigation({ items }) {
  const parentRef = useRef(null);
  const [searchResult, setSearchResult] = useState(null);

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 60,
  });

  const scrollToItem = (index) => {
    virtualizer.scrollToIndex(index, {
      align: "center", // "start" | "center" | "end" | "auto"
      behavior: "smooth", // "auto" | "smooth"
    });
  };

  const handleSearch = (query) => {
    const index = items.findIndex(item =>
      item.name.toLowerCase().includes(query.toLowerCase())
    );
    if (index !== -1) {
      setSearchResult(index);
      scrollToItem(index);
    }
  };

  return (
    <div>
      <SearchBar onSearch={handleSearch} />
      <div ref={parentRef} style={{ height: "600px", overflow: "auto" }}>
        <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
          {virtualizer.getVirtualItems().map(virtualItem => (
            <div
              key={virtualItem.key}
              style={{
                position: "absolute",
                top: 0,
                width: "100%",
                transform: `translateY(${virtualItem.start}px)`,
                // Highlight kết quả tìm kiếm
                background: virtualItem.index === searchResult ? "#fef3c7" : undefined,
              }}
            >
              <ItemRow item={items[virtualItem.index]} />
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

---

## 5. Kết Hợp Với Infinite Scroll

### @tanstack/virtual + @tanstack/react-query

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";
import { useInfiniteQuery } from "@tanstack/react-query";

const PAGE_SIZE = 20;

function InfiniteVirtualList() {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
  } = useInfiniteQuery({
    queryKey: ["users"],
    queryFn: ({ pageParam = 0 }) =>
      fetchUsers({ offset: pageParam, limit: PAGE_SIZE }),
    getNextPageParam: (lastPage, allPages) => {
      const nextOffset = allPages.length * PAGE_SIZE;
      return nextOffset < lastPage.total ? nextOffset : undefined;
    },
  });

  const allUsers = data?.pages.flatMap(page => page.users) ?? [];
  const totalCount = data?.pages[0]?.total ?? 0;

  const parentRef = useRef(null);

  const virtualizer = useVirtualizer({
    count: hasNextPage ? allUsers.length + 1 : allUsers.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 72,
    overscan: 5,
  });

  // Khi scroll gần cuối → tải trang tiếp theo
  useEffect(() => {
    const lastItem = virtualizer.getVirtualItems().at(-1);
    if (!lastItem) return;

    if (
      lastItem.index >= allUsers.length - 1 &&
      hasNextPage &&
      !isFetchingNextPage
    ) {
      fetchNextPage();
    }
  }, [
    virtualizer.getVirtualItems(),
    hasNextPage,
    isFetchingNextPage,
    fetchNextPage,
    allUsers.length,
  ]);

  return (
    <div>
      <p>Tổng: {totalCount} người dùng | Đã tải: {allUsers.length}</p>
      <div ref={parentRef} style={{ height: "600px", overflow: "auto" }}>
        <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
          {virtualizer.getVirtualItems().map(virtualItem => {
            const isLoaderRow = virtualItem.index > allUsers.length - 1;

            return (
              <div
                key={virtualItem.key}
                style={{
                  position: "absolute",
                  top: 0,
                  width: "100%",
                  height: `${virtualItem.size}px`,
                  transform: `translateY(${virtualItem.start}px)`,
                }}
              >
                {isLoaderRow ? (
                  <div>Đang tải thêm...</div>
                ) : (
                  <UserRow user={allUsers[virtualItem.index]} />
                )}
              </div>
            );
          })}
        </div>
      </div>
    </div>
  );
}
```

---

## 6. Virtual Grid — Lưới Ảo

Virtualize dữ liệu dạng lưới (grid) — ví dụ: thư viện ảnh, sản phẩm:

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";

function VirtualImageGrid({ images, columnCount = 3 }) {
  const parentRef = useRef(null);

  const rowCount = Math.ceil(images.length / columnCount);
  const ITEM_HEIGHT = 200; // pixel
  const ITEM_GAP = 8;

  const rowVirtualizer = useVirtualizer({
    count: rowCount,
    getScrollElement: () => parentRef.current,
    estimateSize: () => ITEM_HEIGHT + ITEM_GAP,
    overscan: 2,
  });

  return (
    <div ref={parentRef} style={{ height: "600px", overflow: "auto" }}>
      <div
        style={{
          height: `${rowVirtualizer.getTotalSize()}px`,
          position: "relative",
        }}
      >
        {rowVirtualizer.getVirtualItems().map(virtualRow => {
          // Tính các phần tử trong hàng này
          const startIndex = virtualRow.index * columnCount;
          const rowImages = images.slice(startIndex, startIndex + columnCount);

          return (
            <div
              key={virtualRow.key}
              style={{
                position: "absolute",
                top: 0,
                left: 0,
                width: "100%",
                display: "grid",
                gridTemplateColumns: `repeat(${columnCount}, 1fr)`,
                gap: ITEM_GAP,
                transform: `translateY(${virtualRow.start}px)`,
                height: ITEM_HEIGHT,
              }}
            >
              {rowImages.map(image => (
                <div key={image.id} style={{ height: ITEM_HEIGHT }}>
                  <img
                    src={image.thumbnail}
                    alt={image.title}
                    loading="lazy" // Native lazy loading cho ảnh
                    style={{ width: "100%", height: "100%", objectFit: "cover" }}
                  />
                </div>
              ))}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## 7. Khi Nào Nên Dùng Virtualization

### ✅ Nên Dùng Khi

```
Số lượng phần tử: > 100-500 (tùy độ phức tạp mỗi phần tử)
Phần tử có: image, animation, nhiều nested elements
Môi trường: mobile devices (CPU/GPU yếu hơn)
Triệu chứng: scroll lag, initial render > 200ms, memory cao
```

### ❌ Không Nên Dùng Khi

```
Số lượng phần tử: < 100 (thường không đáng)
Danh sách tĩnh: không scroll (hiển thị toàn bộ trong màn hình)
Phần tử đơn giản: chỉ là text, không có media
Pagination (phân trang): đã giới hạn số phần tử mỗi trang (VD: 10-20)
```

### So Sánh Các Thư Viện

| Tiêu Chí | react-window | @tanstack/virtual | react-virtuoso |
| -------- | ------------ | ----------------- | -------------- |
| **Bundle size** (kích thước) | ~6KB | ~10KB | ~20KB |
| **API** | Đơn giản, dễ học | Linh hoạt, headless | Đơn giản, feature-rich |
| **Dynamic heights** (chiều cao động) | Khó | ✅ Tốt | ✅ Tốt |
| **Horizontal** (chiều ngang) | ✅ | ✅ | ✅ |
| **Grid** (lưới) | ✅ FixedSizeGrid | Tự implement | ✅ |
| **Infinite scroll** | Cần addon | Tự implement | ✅ Tích hợp sẵn |
| **TypeScript** | Tốt | ✅ Xuất sắc | Tốt |
| **Khuyến nghị** | Projects đơn giản | Projects cần linh hoạt | Projects cần nhiều tính năng |

---

## 8. Câu Hỏi Phỏng Vấn

### Câu Hỏi Cơ Bản

**Q: Virtualization là gì và giải quyết vấn đề gì?**

A: Virtualization (Ảo Hóa) chỉ render DOM nodes cho các phần tử đang hiển thị trong viewport, thay vì tạo DOM nodes cho tất cả phần tử. Giải quyết vấn đề performance khi render danh sách dài (>500 items): giảm initial render time, giảm memory usage, cải thiện scroll performance.

---

**Q: react-window và @tanstack/virtual khác nhau như thế nào?**

A:
- **react-window**: API đơn giản hơn, bundle nhỏ hơn (~6KB), tốt cho trường hợp cơ bản. Hỗ trợ fixed và variable size.
- **@tanstack/virtual**: Headless — không có UI mặc định, linh hoạt hơn, hỗ trợ dynamic size measurement tốt hơn, TypeScript tốt hơn. Phù hợp khi cần tùy chỉnh cao.

---

**Q: Khi nào nên dùng pagination thay vì virtualization?**

A:
- **Pagination** (Phân Trang): Phù hợp khi người dùng cần biết tổng số trang, muốn bookmark trang cụ thể, SEO quan trọng, danh sách có thứ tự quan trọng
- **Virtualization**: Phù hợp khi danh sách liên tục (feed, chat), người dùng scroll tự nhiên, không cần biết trang cụ thể, UX mượt mà quan trọng hơn
- **Kết hợp cả hai**: Virtual list + infinite scroll = tải theo trang nhưng hiển thị liền mạch

---

### Câu Hỏi Nâng Cao

**Q: Làm thế nào để virtualize list có phần tử chiều cao thay đổi theo nội dung?**

A: Có ba cách:
1. **VariableSizeList (react-window)**: Tính toán trước chiều cao với `getItemSize(index)` — cần biết trước hoặc ước tính
2. **@tanstack/virtual + measureElement**: Đo kích thước thực sau khi render — chính xác nhất nhưng có delay 1 frame
3. **react-virtuoso**: Tự động đo kích thước, đơn giản nhất cho variable heights

---

**Q: Tại sao cần `overscan` và nên đặt giá trị bao nhiêu?**

A: **Overscan** render thêm N phần tử ngoài viewport để:
- Tránh "flash" — phần tử trắng khi scroll nhanh
- Chuẩn bị sẵn DOM nodes cho scroll tiếp theo
- Mặc định: 1-3 là đủ cho hầu hết trường hợp. Tăng lên 5-10 nếu scroll nhanh gây flash, nhưng quá cao sẽ mất lợi ích của virtualization.

---

## 🔗 Điều Hướng

- **Trước đó:** [2-code-splitting.md](./2-code-splitting.md) — Code Splitting với React.lazy
- **Tiếp theo:** [4-concurrent-features.md](./4-concurrent-features.md) — Concurrent Mode, startTransition
- **Liên quan:** [5-data-fetching/2-tanstack-query-advanced.md](../05-data-fetching/2-tanstack-query-advanced.md) — Infinite queries
- **Chỉ mục:** [INDEX.md](../INDEX.md)
