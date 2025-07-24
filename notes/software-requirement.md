# NOTES ABOUT SOFTWARE REQUIREMENTs SPECIFICATION

## Types of Diagram 💖

### Context Diagram 🏀

**Định nghĩa**: Context Diagram là sự thể hiện sự tương tác giữa các actor (hình vuông) tương tác với hệ thống của chúng ta (hình tròn) thông qua các mũi tên thể hiện sự ra vào của dữ liệu (data must be a noun!!! ⛔)
![Context Diagram](./images/context-diagram.png)

---

### Use Case Diagram 🍄

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

---

### Entity relationship diagrams (ERD)

ERD helps us understand the connection between various "entities" that make up a system.
ERD's Components

- Entity
- Attributes

**Relationship Between Entities in ERDs**

- Zero ![alt text](./images/zero-rel.png)
- One ![alt text](./images/one-rel.png)
- Many ![alt text](./images/many-rel.png)
- Zero or Many ![alt text](./images/zero-or-many-rel.png)
- One or Many ![alt text](./images/one-or-many-rel.png)
- One and only one ![alt text](./images/one-only-on-rel.png)

## Use Case Specification

![Use Case Specification](./images/use-case-specificatoin.png)

**Trigger**
Trigger identifies the conditions, or events that initiate the use case. Think of it as the opening scene of your story - what happens in the real world that makes someone decide they need to use your system?

- Triggers can be ==time-based==, for ex the registration deadline approaches
- Triggers can be ==event-based==, for ex a student submits a grade appeal
- Triggers can be ==need-based==, for ex the department head receives notice that a new course has been approve

**Description**
Description briefly explains what the use case accomplishes from a business perspective. Write it as if you're explaining to a non-technical stakeholder why this functionality matters.
**Template** : The **<Use_Case_Name>** use case represents the process performed by **<Actor_Name>** to **<DO_SOMETHING>**. ....
**Template** : The **<Use_Case_Name>** enter the **System name** from either cooperate intranet or external Internet. 
**Pre-conditions**
Preconditions define the state that must exist before the use case can begin successfully. Think of them as the prerequisite conditions that create a foundation for the interaction.

- ==User-related preconditions== ensure the right person is attempting the right action, for ex authentication, authorization
- ==System-related preconditions== ensure the technology infrastructure is ready, for ex the system must be operational, the database must be accessible, and any required external systems must be available
- ==Data-related preconditions== ensure the necessary information exists

Ký hiệu: PRE-<**Number**>
For example, PRE-1: The barcode reader, printer is connected and functioning correctly

**Post-conditions**
Post-conditions describe how the world has changed after the use case completes, whether successfully or unsuccessfully. Think of them as the "after" state that you could observe and verify.

- ==Successful completion==, identify what new information exists in the system, what relationships have been established or modified, and what new capabilities are now available to users.
- ==Failure scenarios==, consider what partial changes might have occurred and what cleanup might be necessary.

Ký hiệu: POST-<**Number**>
For example, POST-1: The order is successfully created and stored in the system

```
Success:
1. New exam slot is created and stored in the FUE360 database
2. Exam slot appears in system listings for all authorized users
3. Exam slot is available for student check-in and check-out operations
4. System audit log records the creation activity

Failure:
1. No exam slot is created in the system
2. Database remains unchanged
3. Error is logged for administrative review
```

**Normal Flow**
The step-by-step description of how the interaction unfolds when everything goes smoothly.

- Start with the user's initial action and then alternate between user actions and system responses. Each step should be concrete and observable - something you could test or demonstrate. Avoid internal system details that the user can't see, and focus on the externally visible behavior that creates value.
- The scenario should follow a natural rhythm of interaction. Users need time to read information, make decisions, and enter data. Systems need time to process requests, validate information, and prepare responses.
- Consider the cognitive load on the user at each step. Are you asking them to provide information they naturally have available? Are you giving them feedback at appropriate moments? Are you breaking complex tasks into manageable chunks? The main scenario should feel intuitive to someone actually performing the work.
  **Template**

```
1.	The cashier initiates the “Create an order” use case by selecting the menu “Create new order” on the system
2.	The cashier asks the customer if he/she would like to record his/her purchase for membership benefits (by asking the customer the phone number)
3.	The system activates the barcode reader to allow scanning products [see 1-AF]
4.	The cashier scans each product’s barcode using the barcode reader [see 1-AF]
5.	For each scanned product, the system identifies the product details: id, name, price, quantity, promotion (if any) and adds the item into the order
6.	The system calculates the total order amount at whole
7.	After scanning all of the desired products, the casher asks the customer for the preferred payment method (cash, wallet, card) [see 2-AF]
8.	The customer chooses the “Cash” payment option [???phiếu quà tặng thay tiền mặt]
9.	The cashier confirms the payment and finalizes the order
10.	The system updates the inventory accordingly, generates/prints the receipt/bill and returns to the customer, update membership benefits (if any)

```

**Alternative flow**
Alternative flows handle the complications that arise when the ideal scenario encounters real-world messiness.

- Think systematically about what could vary at each step of the main scenario. Users might enter invalid data, make different choices, or change their minds. Systems might detect business rule violations, encounter performance issues, or lose connectivity to external services.
- For each alternative flow, specify where it branches from the main scenario, what triggers the alternative path, how the system responds, and where the flow either rejoins the main path or exits the use case entirely.

**Template**

```
1-AF: The barcode reader fails to scan a product/doesn’t work correctly
a.	The casher manually enters the product details including the product code and quantity via GUI [??? Nhập sai mã s/p]
b.	For each manually entered product, the system identifies the product details: id, name, price, quantity, promotion (if any) and adds the item into the order
c.	Return to STEP 6 (NF) of normal flow
2-AF: The customer chooses the “wallet” payment option
a.	The system checks the customer’s wallet balance to ensure it covers the total order amount
b.	If the wallet balance is sufficient, the system deducts the amount from the customer’s wallet [??? Tiền trong ví ko đủ]
c.	Return to STEP 9 (NF) of normal flow
?-AF:


```

**Exceptions**
While alternative flows handle variations in the normal process, exceptions represent truly disruptive conditions that can occur at any point during the use case execution.
**Template**: <**Number**>-EF. At any time, the app (or something) cannot communicate with the server/core system (or something) (due to network malfunction/technical issues), the system displays an error message. The <**Actor_Name**> calls the technical support for supporting purpose
For example 1-EF: At any time, the app cannot communicate with the server/core system (due to network malfunction/technical issues), the system displays an error message. The cashier calls the technical support for supporting purpose
DB Unavailable: System cannot persist record if database is down → show “Service Unavailable.”
Excel Parse Error: Uploaded file corrupt/invalid format → prompt re‑upload.



**Priority**
Priority reflects the relative importance of this use case within your overall system development effort.
Three levels: ==High (Medium, Low)==

**Frequency of Use**
This section captures how often you expect this use case to be executed, which has profound implications for system design, performance requirements, and user interface considerations.

- High frequency
- Medium frequency
- Low frequency

**Business Rules**
Business rules represent the policies, regulations, constraints, and logical conditions that your system must enforce.

- Business rules can be ==constraining==, like "Exam slots cannot overlap in the same room" or "Students cannot check into an exam more than 30 minutes before the scheduled start time."
- They can be ==derivational==, like "Total exam duration equals scheduled end time minus scheduled start time" or "Exam capacity equals room capacity minus 10% buffer for accessibility needs."

**Template**: BR-<**Number**>
For example
BR-1: The system must support different payment method (cash, wallet, card…)
BR-2: The payment process has an option to record the customer payment by using physical voucher
BR-3: Using phone number to record/keep track the customer membership info

**Other Information**
This catch-all section captures important information that doesn't fit neatly into other categories but still influences how the use case should be implemented or understood

**Assumptions**
Assumptions represent the conditions or facts that you're taking for granted when writing the use case. These are the things you believe to be true but haven't explicitly verified or that represent decisions made elsewhere in the project that affect this particular use case.

---
![use-case-specification-example1](./images/use-case-specification-example1.png)

---
![use-case-specification-example2](./images/use-case-specification-example2.png)

---


##### Template to write non-functional requirements
###### Security 
- The **System_Name** must authenticate all users within 2 seconds using **Something** through the **System_Name**'s SSO system, and must automatically lock user accounts for 15 minutes after 3 consecutive failed login attempts within a 5-minute period.
- The **System_Name**  must encrypt all **Actor_Name** data and personal information using SHA-256 encryption algorithm, m...
###### Performance 
- The **System_Name**  must support concurrent **Action_Name** operations for at least 1000 **Actor_Name**  simultaneously across **System_Name**, with system response time not exceeding 3 seconds for any user transaction under normal operating load conditions.