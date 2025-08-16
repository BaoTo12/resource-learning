
#### Bức tranh tổng thể (mental model)
```c#
[target]  <-- depends on --  [prerequisite files/targets]
   |
   +-- recipe (các lệnh shell để tạo [target] từ [prereqs])
```

> make so sánh timestamp: nếu bất kỳ prereq mới hơn target (hoặc target chưa có), nó chạy recipe. Bạn chỉ cần mô tả đồ thị phụ thuộc và cách dựng nút

#### Cú pháp
```c#
# Variables
APP := myapp            # := = "simply expanded" (evaluate ngay) 
SRC := $(wildcard *.c)  # =  = "recursively expanded" (evaluate khi dùng)

# Rule cơ bản
app: main.o util.o      # target: prerequisites
	$(CC) $^ -o $@      # recipe (TAB ở đầu dòng!)

# Pattern rule (sinh nhiều rule)
%.o: %.c
	$(CC) -c $< -o $@   # $< = first prereq, $@ = target

.PHONY: clean
clean:
	rm -f app *.o

```

#### Biến & phép gán
- = (recursive) giữ nguyên biểu thức đến lúc sử dụng mới expand—chậm & dễ “đánh đố”.
- := (simple) expand ngay lúc gán—dự đoán được, thường dùng trong dự án thật.
- ?= chỉ gán nếu chưa có giá trị (tiện cho override qua CLI: make FOO=bar).
- += nối thêm.

