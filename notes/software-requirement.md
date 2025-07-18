# NOTES ABOUT SOFTWARE REQUIREMENTs SPECIFICATION

### Types of Diagram 💖

#### Context Diagram 🏀

**Định nghĩa**: Context Diagram là sự thể hiện sự tương tác giữa các actor (hình vuông) tương tác với hệ thống của chúng ta (hình tròn) thông qua các mũi tên thể hiện sự ra vào của dữ liệu (data must be a noun!!! ⛔)
![Context Diagram](./images/context-diagram.png)



---

#### Use Case Diagram 🍄

**Định nghĩa**: Model the stakeholders of a system as well as the goals they want to achieve through interacting with the system.
Quy tắt đọc: khi nói đến mối quan hệ include or extend thì chúng ta sẽ có 1 thằng use case là ==base==

- Include: Ví dụ ta có ăn cơm (base)--include--> thanh toán. Thì ở mối quan hện này khi ăn cơm xảy ra thì thanh toán sẽ chắc chắn xảy ra không quan trọng thứ tự thực hiện, như ta có thể thanh toán tiền rồi ăn cơm hoặc là ăn cơm rồi mới thanh toán tiền. Trong trường hợp của include thì cái base là phía không có mũi tên
- Extend: Ví dụ ta có ăn cơm (base) <--extend-- ăn cơm thêm. Thì trong trường hợp này ăn cơm thêm có thể có hoặc không và mũi tên phải chỉ về hướng base use case.

**Use Case Diagram's Components**
- **Use Case** is a function that users can do to achieve something in the system
- **Association** is a interaction with use case, like who uses what functions or use cases
- **Actor**: roles --> can be draw with rectangle
- **System** defines the system boundary
- **Include** relates to the included use case to indicate inserted behavior
- **Extend** is a relationship between two use cases in which one is a variation of another


![alt text](./images/usecase-diagram.png)

Use case diagram symbols and notation

- **Use cases**: Horizontally shaped ovals that represent the different uses that a user might have.
- **Actors**: Stick figures that represent the people actually employing the use cases.
- **Associations**: A line between actors and use cases.
- **System boundary boxes**: A box that sets a system scope to use cases. All use cases outside the box would be considered outside the scope of that system.
- **Include Relationship**: The Include Relationship indicates that a use case includes the functionality of another use case.
  ![Include Relationship](./images/include-relationship.png)
- **Extend Relationship**: illustrates that a use case can be extended by another use case under specific conditions.
  ![Extend Relationship](./images/extend-relationship.png)
- **Generalization Relationship**: The Generalization Relationship establishes an "is-a" connection between two use cases, indicating that one use case is a specialized version of another.
![alt text](./images/generalization-relationship.png)


___

