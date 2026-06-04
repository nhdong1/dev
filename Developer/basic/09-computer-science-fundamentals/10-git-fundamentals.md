# Git Fundamentals - Interview Questions

## Question 1: Git lưu trữ dữ liệu theo mô hình nào?

**Answer:**

- Git là content-addressable store dựa trên hash.
- Các object chính: blob, tree, commit, tag.
- Commit trỏ tới tree snapshot và commit cha.
- Vì vậy Git thiên về snapshot hơn là diff thuần túy.

---

## Question 2: Working tree, staging area, local repo khác nhau ra sao?

**Answer:**

- Working tree: file bạn đang chỉnh sửa.
- Staging area (index): tập thay đổi chuẩn bị commit.
- Local repo: lịch sử commit đã ghi nhận.
- Hiểu ba vùng này giúp thao tác add/reset/commit chính xác.

---

## Question 3: `git merge` và `git rebase` khác nhau như thế nào?

**Answer:**

- Merge giữ nguyên lịch sử và tạo merge commit.
- Rebase viết lại lịch sử bằng cách đặt commit lên base mới.
- Rebase giúp lịch sử tuyến tính, dễ đọc log.
- Không rebase branch đã chia sẻ công khai nếu team chưa thống nhất.

---

## Question 4: Fast-forward merge là gì?

**Answer:**

- Xảy ra khi target branch chưa có commit mới kể từ điểm tách.
- Git chỉ cần di chuyển con trỏ branch, không tạo merge commit.
- Lịch sử gọn hơn nhưng mất dấu mốc hợp nhất rõ ràng.
- Nhiều team dùng `--no-ff` để giữ context merge PR.

---

## Question 5: Cherry-pick dùng trong tình huống nào?

**Answer:**

- Lấy một hoặc vài commit cụ thể sang branch khác.
- Hữu ích khi backport fix sang release branch.
- Cẩn thận xung đột và duplicate commit logic.
- Nên ghi chú rõ trong message để dễ truy vết.

---

## Question 6: Reflog giúp cứu tình huống gì?

**Answer:**

- Reflog lưu lịch sử di chuyển HEAD/local refs.
- Dùng để tìm commit bị "mất" sau reset/rebase sai.
- Là công cụ cứu hộ quan trọng khi thao tác lịch sử nhầm.
- Reflog cục bộ và có thời hạn, cần xử lý sớm khi gặp sự cố.

---

## Question 7: Conflict thường xuất hiện khi nào?

**Answer:**

- Khi hai nhánh sửa cùng vùng nội dung theo cách không tương thích.
- Phổ biến lúc merge, rebase, cherry-pick.
- Giảm conflict bằng chia nhỏ PR và sync nhánh thường xuyên.
- Sau khi resolve, cần chạy test để xác nhận hành vi đúng.

---

## Question 8: `.gitignore` có giới hạn gì?

**Answer:**

- Chỉ áp dụng cho file chưa được track.
- File đã track thì thêm vào ignore vẫn tiếp tục theo dõi.
- Cần `git rm --cached` nếu muốn ngừng track.
- Nên chuẩn hóa ignore cho build artifacts và secrets.

---

## Question 9: Tag và Branch khác nhau?

**Answer:**

- Branch là con trỏ di động theo commit mới.
- Tag (thường annotated tag) đánh dấu mốc cố định.
- Dùng tag cho phiên bản release, rollback, audit.
- SemVer + tag nhất quán giúp vận hành và truy vết dễ hơn.

---

## Question 10: Conventional Commits mang lại lợi ích gì?

**Answer:**

- Chuẩn hóa message commit theo loại thay đổi.
- Dễ tạo changelog tự động và semantic release.
- Tăng tính rõ ràng cho review và lịch sử.
- Quan trọng là team thống nhất và tuân thủ nhất quán.

---

## Question 11: Squash merge nên dùng khi nào?

**Answer:**

- Khi muốn gộp nhiều commit nhỏ WIP thành một commit sạch.
- Lịch sử main gọn và tập trung vào ý nghĩa thay đổi.
- Mất chi tiết commit trung gian ở nhánh feature.
- Phù hợp team ưu tiên lịch sử đơn giản, dễ đọc.

---

## Question 12: Force push có rủi ro gì?

**Answer:**

- Có thể ghi đè lịch sử branch remote.
- Dễ làm đồng đội mất commit nếu phối hợp không chặt.
- Nếu cần, ưu tiên `--force-with-lease` để an toàn hơn.
- Hạn chế force push lên branch protected.

---

## Question 13: Git hooks giúp tự động hóa gì?

**Answer:**

- Chạy lint, test, format trước commit/push.
- Giảm lỗi cơ bản lọt vào CI.
- Hook local tiện lợi nhưng không thay thế server-side checks.
- Cần giữ hook nhanh để không ảnh hưởng trải nghiệm dev.

---

## Question 14: Mono-repo và Multi-repo ảnh hưởng workflow Git ra sao?

**Answer:**

- Mono-repo dễ đồng bộ thay đổi cross-service, nhưng repo lớn.
- Multi-repo tách quyền và vòng đời rõ, nhưng khó quản lý phụ thuộc.
- Git strategy (branching, CI, code owners) phải phù hợp mô hình repo.
- Không có mô hình tuyệt đối tốt, chỉ có phù hợp ngữ cảnh tổ chức.

---

## Question 15: Best practices cho pull request chất lượng?

**Answer:**

- PR nhỏ, mục tiêu rõ, mô tả bối cảnh và tác động.
- Có checklist test và rủi ro rollout.
- Tách refactor và feature để reviewer dễ đánh giá.
- Phản hồi review nhanh, giữ trao đổi kỹ thuật tập trung và tôn trọng.
