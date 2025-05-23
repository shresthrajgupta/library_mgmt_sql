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

**13) Identify members who currently have overdue books**
```sql
SELECT i.issued_member_id, m.member_name, b.book_title, i.issued_date, CURDATE() - i.issued_date as overdues_days
FROM issued_status as i
INNER JOIN members as m
ON m.member_id = i.issued_member_id
INNER JOIN books as b
ON b.isbn = i.issued_book_isbn
LEFT JOIN return_status as r
ON r.issued_id = i.issued_id
WHERE r.return_date IS NULL
AND (CURDATE() - i.issued_date) > 30
ORDER BY 5;
```

**14) Update the status of a book when it is returned**
```sql
DROP PROCEDURE IF EXISTS add_return_records;

DELIMITER //

CREATE PROCEDURE add_return_records(IN p_return_id VARCHAR(10), IN p_issued_id VARCHAR(10), IN p_book_quality VARCHAR(10))

BEGIN
	DECLARE v_isbn VARCHAR(50);
    DECLARE v_book_name VARCHAR(80);
    
    INSERT INTO return_status(return_id, issued_id, return_date, book_quality)
    VALUES (p_return_id, p_issued_id, CURDATE(), p_book_quality);

    SELECT issued_book_isbn, issued_book_name INTO v_isbn, v_book_name
    FROM issued_status
    WHERE issued_id = p_issued_id;

    UPDATE books
    SET status = 'yes'
    WHERE isbn = v_isbn;

    SELECT CONCAT('Thank you for returning the book: ', v_book_name) AS message;
END //

DELIMITER ;

CALL add_return_records('RS138', 'IS135', 'Good');
CALL add_return_records('RS148', 'IS140', 'Good');

SELECT * FROM return_status;

SELECT * FROM books WHERE book_title LIKE '%Sapiens%';
SELECT * FROM books WHERE book_title LIKE '%Animal%';
```

**15) Branch performance report in a new table `branch_reports`**
```sql
DROP TABLE IF EXISTS branch_reports;

CREATE TABLE branch_reports AS
SELECT b.branch_id, b.manager_id, COUNT(i.issued_id) as count_book_issued, COUNT(r.return_id) as count_book_returned, SUM(bk.rental_price) as total_revenue
FROM issued_status as i
INNER JOIN employees as e
ON e.emp_id = i.issued_emp_id
INNER JOIN branch as b
ON e.branch_id = b.branch_id
LEFT JOIN return_status as r
ON r.issued_id = i.issued_id
INNER JOIN books as bk
ON i.issued_book_isbn = bk.isbn
GROUP BY 1, 2;

SELECT * FROM branch_reports;
```

**16) Create a new table `active_members` containing members who issued at least one book in the past 2 months**
```sql
DROP TABLE IF EXISTS active_members;

CREATE TABLE active_members AS
SELECT * FROM members
WHERE member_id IN (SELECT DISTINCT issued_member_id FROM issued_status WHERE issued_date >= CURDATE() - INTERVAL 2 MONTH);

SELECT * FROM active_members;
```

**17) Find top 3 employees who have processed the highest number of book issues**
```sql
SELECT e.emp_id, e.emp_name, COUNT(*) AS count_books_issued, b.branch_id, b.branch_address
FROM issued_status as i
INNER JOIN employees as e
ON e.emp_id = i.issued_emp_id
INNER JOIN branch as b
ON e.branch_id = b.branch_id
GROUP BY e.emp_id, e.emp_name, b.branch_id, b.branch_address
ORDER BY COUNT(*) DESC
LIMIT 3;
```

**18) Issue a book if it is available**
```sql
DROP PROCEDURE IF EXISTS issue_book;

DELIMITER //

CREATE PROCEDURE issue_book(IN p_issued_id VARCHAR(10), IN p_issued_member_id VARCHAR(30), IN p_issued_book_isbn VARCHAR(30), IN p_issued_emp_id VARCHAR(10))

BEGIN
    DECLARE v_status VARCHAR(10);
    
    -- checking if book is available 'yes'
    SELECT `status` INTO v_status 
    FROM books 
    WHERE isbn = p_issued_book_isbn;

    IF v_status = 'yes' THEN
		INSERT INTO issued_status(issued_id, issued_member_id, issued_date, issued_book_isbn, issued_emp_id)
		VALUES (p_issued_id, p_issued_member_id, CURDATE(), p_issued_book_isbn, p_issued_emp_id);

		UPDATE books SET `status` = 'no'
		WHERE isbn = p_issued_book_isbn;

		SELECT CONCAT('Book records added successfully for book isbn: ', p_issued_book_isbn) AS message;
    ELSE
        SELECT CONCAT('Sorry to inform you the book you have requested is unavailable book_isbn: ', p_issued_book_isbn) AS message;
    END IF;
END //

DELIMITER ;

CALL issue_book('IS155', 'C108', '978-0-553-29698-2', 'E104');
CALL issue_book('IS156', 'C108', '978-0-375-41398-8', 'E104');

SELECT * FROM issued_status;

SELECT * FROM books
WHERE isbn = '978-0-553-29698-2';

SELECT * FROM books
WHERE isbn = '978-0-375-41398-8';
```

***

### Contact
For questions or suggestions, feel free to open an issue or fork this repo and contribute.
