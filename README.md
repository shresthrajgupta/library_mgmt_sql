# Library Management using SQL

## Project overview
This project is a relational database system designed to manage the day-to-day operations of a library. Built using MySQL, it allows efficient handling of books, members, borrowing records, and returns. Key features include tracking book availability, managing member registrations, and monitoring overdue returns. The system ensures data integrity through foreign key constraints and provides a scalable structure for library databases.

***

### Database setup

```sql
CREATE DATABASE library_db;
USE library_db;

-- import branch, employees, members, books, issued_status, return_status tables
```

***

### Operations

**1) Create a new record for a book**
```sql
INSERT INTO books(isbn, book_title, category, rental_price, status, author, publisher)
VALUES('978-1-60129-456-2', 'To Kill a Mockingbird', 'Classic', 6.00, 'yes', 'Harper Lee', 'J.B. Lippincott & Co.');

SELECT * FROM books WHERE isbn = '978-1-60129-456-2';
```

**2) Update the address of an existing member**
```sql
UPDATE members
SET member_address = '125 Oak St'
WHERE member_id = 'C103';

SELECT * FROM members WHERE member_id = 'C103';
```

**3) Delete a record from the `issued_status` table**
```sql
DELETE FROM issued_status
WHERE issued_id = 'IS121';

SELECT * FROM issued_status
WHERE issued_id = 'IS121';
```

**4) Retrieve all books issued by a specific employee**
```sql
SELECT * FROM issued_status
WHERE issued_emp_id = 'E101';
```

**5) List members who have issued more than one book**
```sql
SELECT issued_member_id, COUNT(*) AS issued_books_count
FROM issued_status
GROUP BY 1
HAVING COUNT(*) > 1;
```

**6) CTAS to store the count of each book issued**
```sql
DROP TABLE IF EXISTS book_issued_cnt;

CREATE TABLE book_issued_cnt AS
SELECT books.isbn, books.book_title, COUNT(issued_status.issued_id) AS issue_count
FROM issued_status
INNER JOIN books
ON issued_status.issued_book_isbn = books.isbn
GROUP BY 1, 2;

SELECT * FROM book_issued_cnt;
```

**7) Retrieve all books that belong to a specific category**
```sql
SELECT * FROM books
WHERE category = 'Fantasy';
```

**8) Find the total rental income for each category**
```sql
SELECT b.category, COUNT(*) AS total_count_borrowed, SUM(b.rental_price) AS total_rent_profit
FROM issued_status as i
INNER JOIN books as b
ON b.isbn = i.issued_book_isbn
GROUP BY 1;
```

**9) List members who registered within the last 180 days from the date 2022-05-01**
```sql
SELECT * FROM members
WHERE DATEDIFF('2022-05-01', reg_date) <= 180 AND DATEDIFF('2022-05-01', reg_date) >= 0;
```

**10) List employees along with their branch manager's name and branch details**
```sql
SELECT e1.emp_id, e1.emp_name, e1.position, e1.salary, b.*, e2.emp_name as manager
FROM employees as e1
INNER JOIN branch as b
ON e1.branch_id = b.branch_id    
INNER JOIN employees as e2
ON e2.emp_id = b.manager_id;
```

**11) Create a table `expensive_books` containing books with a rental price above 7**
```sql
DROP TABLE IF EXISTS expensive_books;

CREATE TABLE expensive_books AS
SELECT * FROM books
WHERE rental_price > 7.00;

SELECT * FROM expensive_books;
```

**12) Retrieve the list of books that have not yet been returned**
```sql
SELECT * FROM issued_status as i
LEFT JOIN return_status as r
ON r.issued_id = i.issued_id
WHERE r.return_id IS NULL;
```
