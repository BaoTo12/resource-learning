# NOTES ABOUT SOFTWARE REQUIREMENTs SPECIFICATION

### Types of Diagram 💖

#### Content Diagram 🏀

**Định nghĩa**: Context Diagram là sự thể hiện sự tương tác giữa các actor (hình vuông) tương tác với hệ thống của chúng ta (hình tròn) thông qua các mũi tên thể hiện sự ra vào của dữ liệu (data must be a noun!!! ⛔)
![Context Diagram](./images/context-diagram.png)

---

#### Use Case Diagram 🍄

**Định nghĩa**: Model the stakeholders of a system as well as the goals they want to achieve through interacting with the system.
Quy tắt đọc

- Include: đọc từ gốc mũi tên, **ví dụ** thằng Manage meal Subscription là thằng chính còn login là thằng phụ
- Extend: đọc thằng chính ở ngọn, **ví dụ** view menu là thằng chính còn order a meal là thằng phụ

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

