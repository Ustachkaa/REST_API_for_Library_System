# Library Management System - REST API Design

## 1. System Overview

This REST API is designed for a comprehensive library management system that allows librarians to manage books, users to borrow and return materials, track due dates, manage reservations, and handle fines. The system includes automated email notifications for due dates and overdue items.

## 2. Entities and Relationships

### Core Entities

#### **Book**
- `book_id` (UUID, Primary Key)
- `isbn` (String, Unique, Indexed)
- `title` (String, Required)
- `authors` (Array of Strings)
- `publisher` (String)
- `publication_year` (Integer)
- `category_id` (UUID, Foreign Key → Category)
- `total_copies` (Integer, >= 0)
- `available_copies` (Integer, >= 0)
- `description` (Text)
- `cover_image_url` (String)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

#### **User**
- `user_id` (UUID, Primary Key)
- `email` (String, Unique, Required, Indexed)
- `first_name` (String, Required)
- `last_name` (String, Required)
- `phone` (String)
- `address` (Object: street, city, state, zip)
- `membership_type` (Enum: standard, premium, student)
- `membership_expires` (Date)
- `total_fines` (Decimal, >= 0)
- `is_active` (Boolean)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

#### **Loan**
- `loan_id` (UUID, Primary Key)
- `book_id` (UUID, Foreign Key → Book, Indexed)
- `user_id` (UUID, Foreign Key → User, Indexed)
- `checkout_date` (Timestamp, Required)
- `due_date` (Timestamp, Required, Indexed)
- `return_date` (Timestamp, Nullable)
- `status` (Enum: active, returned, overdue)
- `renewal_count` (Integer, Default: 0)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

#### **Reservation**
- `reservation_id` (UUID, Primary Key)
- `book_id` (UUID, Foreign Key → Book, Indexed)
- `user_id` (UUID, Foreign Key → User, Indexed)
- `reservation_date` (Timestamp, Required)
- `expiry_date` (Timestamp, Required, Indexed)
- `status` (Enum: pending, fulfilled, expired, cancelled)
- `notified_at` (Timestamp, Nullable)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

#### **Fine**
- `fine_id` (UUID, Primary Key)
- `loan_id` (UUID, Foreign Key → Loan, Indexed)
- `user_id` (UUID, Foreign Key → User, Indexed)
- `amount` (Decimal, Required, >= 0)
- `reason` (String: overdue, damage, loss)
- `status` (Enum: pending, paid, waived)
- `issued_date` (Timestamp, Required)
- `paid_date` (Timestamp, Nullable)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

#### **Category**
- `category_id` (UUID, Primary Key)
- `name` (String, Unique, Required)
- `description` (Text)
- `parent_category_id` (UUID, Foreign Key → Category, Nullable)
- `created_at` (Timestamp)
- `updated_at` (Timestamp)

#### **Notification**
- `notification_id` (UUID, Primary Key)
- `user_id` (UUID, Foreign Key → User, Indexed)
- `type` (Enum: due_reminder, overdue, reservation_available, fine_notice)
- `title` (String, Required)
- `message` (Text, Required)
- `status` (Enum: pending, sent, failed)
- `scheduled_for` (Timestamp, Indexed)
- `sent_at` (Timestamp, Nullable)
- `created_at` (Timestamp)

### Entity Relationships

- **Book → Category**: Many-to-One (A book belongs to one category)
- **User → Loan**: One-to-Many (A user can have multiple loans)
- **Book → Loan**: One-to-Many (A book can have multiple loan records)
- **User → Reservation**: One-to-Many (A user can reserve multiple books)
- **Book → Reservation**: One-to-Many (A book can have multiple reservations)
- **Loan → Fine**: One-to-One or One-to-Many (A loan can generate multiple fines)
- **User → Fine**: One-to-Many (A user can accumulate multiple fines)
- **User → Notification**: One-to-Many (A user receives multiple notifications)

## 3. API Operations and Endpoints

### Authentication & Authorization
- **POST** `/api/v1/auth/login` - User authentication
- **POST** `/api/v1/auth/logout` - End user session
- **POST** `/api/v1/auth/refresh` - Refresh access token
- **GET** `/api/v1/auth/me` - Get current user profile

### Books Collection

#### List Books
```
GET /api/v1/books
```
**Query Parameters:**
- `page` (integer, default: 1) - Page number
- `limit` (integer, default: 20, max: 100) - Items per page
- `search` (string) - Search in title, authors, ISBN
- `category_id` (UUID) - Filter by category
- `author` (string) - Filter by author
- `available_only` (boolean) - Show only available books
- `sort` (string, options: title, publication_year, created_at) - Sort field
- `order` (string, options: asc, desc, default: asc) - Sort order

**Response:** `200 OK`
```json
{
  "data": [
    {
      "book_id": "uuid",
      "isbn": "978-3-16-148410-0",
      "title": "The Great Gatsby",
      "authors": ["F. Scott Fitzgerald"],
      "publisher": "Scribner",
      "publication_year": 1925,
      "category": {
        "category_id": "uuid",
        "name": "Classic Literature"
      },
      "total_copies": 5,
      "available_copies": 3,
      "cover_image_url": "https://cdn.library.com/covers/123.jpg"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 156,
    "total_pages": 8
  },
  "links": {
    "self": "/api/v1/books?page=1",
    "next": "/api/v1/books?page=2",
    "last": "/api/v1/books?page=8"
  }
}
```

#### Get Book Details
```
GET /api/v1/books/{book_id}
```
**Response:** `200 OK`
```json
{
  "book_id": "uuid",
  "isbn": "978-3-16-148410-0",
  "title": "The Great Gatsby",
  "authors": ["F. Scott Fitzgerald"],
  "publisher": "Scribner",
  "publication_year": 1925,
  "category": {
    "category_id": "uuid",
    "name": "Classic Literature"
  },
  "total_copies": 5,
  "available_copies": 3,
  "description": "A novel set in the Jazz Age...",
  "cover_image_url": "https://cdn.library.com/covers/123.jpg",
  "current_loans": 2,
  "current_reservations": 1,
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-20T14:30:00Z"
}
```

#### Create Book (Admin Only)
```
POST /api/v1/books
```
**Request Body:**
```json
{
  "isbn": "978-3-16-148410-0",
  "title": "The Great Gatsby",
  "authors": ["F. Scott Fitzgerald"],
  "publisher": "Scribner",
  "publication_year": 1925,
  "category_id": "uuid",
  "total_copies": 5,
  "description": "A novel set in the Jazz Age...",
  "cover_image_url": "https://cdn.library.com/covers/123.jpg"
}
```
**Response:** `201 Created`

#### Update Book (Admin Only)
```
PATCH /api/v1/books/{book_id}
```
**Request Body:** (Partial update allowed)
```json
{
  "total_copies": 7,
  "description": "Updated description..."
}
```
**Response:** `200 OK`

#### Delete Book (Admin Only)
```
DELETE /api/v1/books/{book_id}
```
**Response:** `204 No Content`

### Users Collection

#### List Users (Admin Only)
```
GET /api/v1/users
```
**Query Parameters:**
- `page`, `limit` - Pagination
- `search` (string) - Search by name or email
- `membership_type` (enum) - Filter by membership
- `has_overdue` (boolean) - Users with overdue loans
- `has_fines` (boolean) - Users with outstanding fines

**Response:** `200 OK` (Similar pagination structure as books)

#### Get User Profile
```
GET /api/v1/users/{user_id}
```
**Response:** `200 OK`
```json
{
  "user_id": "uuid",
  "email": "john.doe@example.com",
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+1-555-0123",
  "address": {
    "street": "123 Main St",
    "city": "Springfield",
    "state": "IL",
    "zip": "62701"
  },
  "membership_type": "standard",
  "membership_expires": "2025-12-31",
  "total_fines": 5.50,
  "is_active": true,
  "statistics": {
    "total_loans": 45,
    "active_loans": 2,
    "overdue_loans": 0,
    "total_reservations": 3
  }
}
```

#### Create User
```
POST /api/v1/users
```
**Request Body:**
```json
{
  "email": "john.doe@example.com",
  "password": "SecureP@ss123",
  "first_name": "John",
  "last_name": "Doe",
  "phone": "+1-555-0123",
  "address": {
    "street": "123 Main St",
    "city": "Springfield",
    "state": "IL",
    "zip": "62701"
  },
  "membership_type": "standard"
}
```
**Response:** `201 Created`

#### Update User
```
PATCH /api/v1/users/{user_id}
```
**Response:** `200 OK`

#### Deactivate User (Admin Only)
```
DELETE /api/v1/users/{user_id}
```
**Response:** `204 No Content`

### Loans Collection

#### List Loans
```
GET /api/v1/loans
```
**Query Parameters:**
- `page`, `limit` - Pagination
- `user_id` (UUID) - Filter by user
- `book_id` (UUID) - Filter by book
- `status` (enum: active, returned, overdue) - Filter by status
- `due_before` (date) - Due before specific date
- `due_after` (date) - Due after specific date
- `sort` (checkout_date, due_date, return_date)

**Response:** `200 OK`
```json
{
  "data": [
    {
      "loan_id": "uuid",
      "book": {
        "book_id": "uuid",
        "title": "The Great Gatsby",
        "isbn": "978-3-16-148410-0"
      },
      "user": {
        "user_id": "uuid",
        "name": "John Doe",
        "email": "john.doe@example.com"
      },
      "checkout_date": "2025-01-15T10:00:00Z",
      "due_date": "2025-02-05T23:59:59Z",
      "return_date": null,
      "status": "active",
      "renewal_count": 0,
      "days_until_due": 8,
      "is_renewable": true
    }
  ],
  "pagination": { "..." }
}
```

#### Checkout Book
```
POST /api/v1/loans
```
**Request Body:**
```json
{
  "book_id": "uuid",
  "user_id": "uuid"
}
```
**Business Rules:**
- User must be active and have no excessive fines
- Book must be available
- User cannot exceed loan limit (e.g., 5 active loans)
- Automatically sets due_date (e.g., 21 days from checkout)

**Response:** `201 Created`
```json
{
  "loan_id": "uuid",
  "book_id": "uuid",
  "user_id": "uuid",
  "checkout_date": "2025-01-28T14:30:00Z",
  "due_date": "2025-02-18T23:59:59Z",
  "status": "active",
  "message": "Book checked out successfully. Due date: February 18, 2025"
}
```

#### Return Book
```
POST /api/v1/loans/{loan_id}/return
```
**Request Body:** (Optional)
```json
{
  "condition_notes": "Book in good condition"
}
```
**Response:** `200 OK`
```json
{
  "loan_id": "uuid",
  "return_date": "2025-02-10T11:00:00Z",
  "status": "returned",
  "days_overdue": 0,
  "fine_applied": 0.00,
  "message": "Book returned successfully"
}
```

#### Renew Loan
```
POST /api/v1/loans/{loan_id}/renew
```
**Business Rules:**
- Cannot renew if overdue
- Maximum renewals: 2 times
- Cannot renew if book has reservations
- Extends due_date by standard period (e.g., 21 days)

**Response:** `200 OK`
```json
{
  "loan_id": "uuid",
  "new_due_date": "2025-03-11T23:59:59Z",
  "renewal_count": 1,
  "message": "Loan renewed successfully. New due date: March 11, 2025"
}
```

### Reservations Collection

#### List Reservations
```
GET /api/v1/reservations
```
**Query Parameters:**
- `page`, `limit` - Pagination
- `user_id` (UUID) - Filter by user
- `book_id` (UUID) - Filter by book
- `status` (enum: pending, fulfilled, expired, cancelled)

**Response:** `200 OK`

#### Create Reservation
```
POST /api/v1/reservations
```
**Request Body:**
```json
{
  "book_id": "uuid",
  "user_id": "uuid"
}
```
**Business Rules:**
- Book must have 0 available copies (or reservation wouldn't be needed)
- User cannot reserve same book twice
- Reservation expires after 7 days if not fulfilled
- User added to queue (FIFO)

**Response:** `201 Created`
```json
{
  "reservation_id": "uuid",
  "book_id": "uuid",
  "user_id": "uuid",
  "reservation_date": "2025-01-28T14:30:00Z",
  "expiry_date": "2025-02-04T14:30:00Z",
  "status": "pending",
  "queue_position": 3,
  "message": "Reservation created. You are #3 in queue."
}
```

#### Cancel Reservation
```
DELETE /api/v1/reservations/{reservation_id}
```
**Response:** `204 No Content`

### Fines Collection

#### List Fines
```
GET /api/v1/fines
```
**Query Parameters:**
- `page`, `limit` - Pagination
- `user_id` (UUID) - Filter by user
- `status` (enum: pending, paid, waived)
- `min_amount`, `max_amount` (decimal) - Amount range

**Response:** `200 OK`

#### Get User's Fines
```
GET /api/v1/users/{user_id}/fines
```
**Response:** `200 OK`
```json
{
  "user_id": "uuid",
  "total_fines": 15.50,
  "fines": [
    {
      "fine_id": "uuid",
      "loan_id": "uuid",
      "book_title": "The Great Gatsby",
      "amount": 10.50,
      "reason": "overdue",
      "status": "pending",
      "issued_date": "2025-01-20T00:00:00Z",
      "days_overdue": 5
    },
    {
      "fine_id": "uuid",
      "loan_id": "uuid",
      "book_title": "1984",
      "amount": 5.00,
      "reason": "damage",
      "status": "pending",
      "issued_date": "2025-01-25T00:00:00Z"
    }
  ]
}
```

#### Pay Fine
```
POST /api/v1/fines/{fine_id}/pay
```
**Request Body:**
```json
{
  "payment_method": "credit_card",
  "payment_reference": "TXN123456"
}
```
**Response:** `200 OK`

#### Waive Fine (Admin Only)
```
POST /api/v1/fines/{fine_id}/waive
```
**Request Body:**
```json
{
  "reason": "First-time offender, waiving as courtesy"
}
```
**Response:** `200 OK`

### Categories Collection

#### List Categories
```
GET /api/v1/categories
```
**Query Parameters:**
- `parent_id` (UUID, optional) - Get subcategories
- `include_children` (boolean) - Include full tree

**Response:** `200 OK`
```json
{
  "data": [
    {
      "category_id": "uuid",
      "name": "Fiction",
      "description": "Fiction books",
      "parent_category_id": null,
      "book_count": 450,
      "subcategories": [
        {
          "category_id": "uuid",
          "name": "Classic Literature",
          "book_count": 120
        }
      ]
    }
  ]
}
```

#### Create Category (Admin Only)
```
POST /api/v1/categories
```

#### Update Category (Admin Only)
```
PATCH /api/v1/categories/{category_id}
```

#### Delete Category (Admin Only)
```
DELETE /api/v1/categories/{category_id}
```

### Notifications Collection

#### List User Notifications
```
GET /api/v1/users/{user_id}/notifications
```
**Query Parameters:**
- `page`, `limit` - Pagination
- `type` (enum) - Filter by notification type
- `status` (enum: pending, sent, failed)
- `unread_only` (boolean)

**Response:** `200 OK`

#### Mark Notification as Read
```
PATCH /api/v1/notifications/{notification_id}/read
```
**Response:** `200 OK`

### Reports & Analytics (Admin Only)

#### Get Library Statistics
```
GET /api/v1/reports/statistics
```
**Response:** `200 OK`
```json
{
  "total_books": 15000,
  "total_users": 3500,
  "active_loans": 890,
  "overdue_loans": 45,
  "total_reservations": 123,
  "total_fines_outstanding": 2450.75,
  "books_added_this_month": 150,
  "new_users_this_month": 45
}
```

#### Get Overdue Report
```
GET /api/v1/reports/overdue
```
**Query Parameters:**
- `days_overdue` (integer) - Minimum days overdue
- `format` (enum: json, csv) - Export format

**Response:** `200 OK`

#### Get Popular Books Report
```
GET /api/v1/reports/popular-books
```
**Query Parameters:**
- `period` (enum: week, month, year)
- `limit` (integer, default: 10)

**Response:** `200 OK`

## 4. Scheduling and Alerting Systems

### Automated Background Jobs

#### Daily Overdue Check (Runs at 1:00 AM daily)
**Process:**
1. Query all loans where `return_date IS NULL AND due_date < NOW() AND status != 'overdue'`
2. Update loan status to 'overdue'
3. Calculate fine amount (e.g., $0.50 per day)
4. Create fine record in database
5. Create notification for user
6. Send email notification

#### Due Date Reminders (Runs at 9:00 AM daily)
**Process:**
1. Query loans due in 3 days: `return_date IS NULL AND due_date BETWEEN NOW() AND NOW() + INTERVAL '3 days'`
2. Create notifications for each loan
3. Send email reminders with book details and due date

#### Reservation Fulfillment (Triggered on book return)
**Process:**
1. When book is returned, check for pending reservations
2. Find oldest pending reservation (FIFO)
3. Update reservation status to 'fulfilled'
4. Set book aside (decrement available_copies but don't allow checkout by others)
5. Create notification for user
6. Send email: "Your reserved book is now available"
7. Set expiry: User has 3 days to pick up

#### Reservation Expiry Check (Runs at 2:00 AM daily)
**Process:**
1. Query reservations where `status = 'fulfilled' AND expiry_date < NOW()`
2. Update status to 'expired'
3. Increment available_copies
4. Process next reservation in queue if any

#### Weekly Fine Reminders (Runs Sunday at 10:00 AM)
**Process:**
1. Query users with `total_fines > 0 AND status = 'pending'`
2. Group fines by user
3. Send summary email with total amount due
4. Include payment instructions

#### Monthly Membership Expiry (Runs 1st of month at 8:00 AM)
**Process:**
1. Query users where `membership_expires < NOW() + INTERVAL '30 days'`
2. Send renewal reminder emails
3. Users expiring within 7 days get urgent notification

### Notification Templates

#### Due Date Reminder
```
Subject: Book Due in 3 Days - "{book_title}"

Hi {user_name},

This is a friendly reminder that the following book is due soon:

Book: {book_title}
Due Date: {due_date}
Days Remaining: 3

Please return the book by the due date to avoid late fees.
You can also renew this book online if you need more time.

[Renew Now Button]

Thank you,
{library_name}
```

#### Overdue Notice
```
Subject: Overdue Book - "{book_title}"

Hi {user_name},

The following book is now overdue:

Book: {book_title}
Due Date: {due_date}
Days Overdue: {days_overdue}
Current Fine: ${fine_amount}

Please return this book as soon as possible. 
Fines accrue at $0.50 per day.

[View My Account Button]

Thank you,
{library_name}
```

#### Reservation Available
```
Subject: Your Reserved Book is Ready!

Hi {user_name},

Good news! The book you reserved is now available:

Book: {book_title}
Available Until: {expiry_date}

Please visit the library within 3 days to check out this book.
After {expiry_date}, it will be offered to the next person in line.

[View Reservation Button]

Thank you,
{library_name}
```

## 5. API Design Principles

### HTTP Methods Usage
- **GET** - Retrieve resources (safe, idempotent)
- **POST** - Create new resources or trigger actions
- **PATCH** - Partial update of resources
- **PUT** - Full replacement of resources (not used in this API)
- **DELETE** - Remove resources

### Status Codes
- **200 OK** - Successful GET, PATCH, or action
- **201 Created** - Successful POST creating new resource
- **204 No Content** - Successful DELETE
- **400 Bad Request** - Invalid input data
- **401 Unauthorized** - Missing or invalid authentication
- **403 Forbidden** - Authenticated but lacking permissions
- **404 Not Found** - Resource doesn't exist
- **409 Conflict** - Business rule violation (e.g., book already checked out)
- **422 Unprocessable Entity** - Valid syntax but business logic error
- **429 Too Many Requests** - Rate limit exceeded
- **500 Internal Server Error** - Unexpected server error

### Error Response Format
```json
{
  "error": {
    "code": "BOOK_NOT_AVAILABLE",
    "message": "This book is currently checked out and has no available copies",
    "details": {
      "book_id": "uuid",
      "available_copies": 0,
      "expected_return_date": "2025-02-15T23:59:59Z",
      "reservation_available": true
    },
    "timestamp": "2025-01-28T14:30:00Z",
    "request_id": "req-12345"
  }
}
```

### Authentication
- JWT-based authentication
- Access tokens expire in 1 hour
- Refresh tokens expire in 30 days
- Include in header: `Authorization: Bearer {token}`

### Rate Limiting
- 100 requests per minute per user
- 1000 requests per minute per API key (for integrations)
- Headers included in response:
  - `X-RateLimit-Limit: 100`
  - `X-RateLimit-Remaining: 95`
  - `X-RateLimit-Reset: 1643385600`

### Pagination
- Default page size: 20 items
- Maximum page size: 100 items
- Include pagination metadata in response
- Provide HATEOAS links (self, next, prev, first, last)

### Filtering and Sorting
- Use query parameters for filtering
- Support multiple filters combined with AND logic
- Use `sort` parameter with field name
- Use `order` parameter with `asc` or `desc`

### Versioning
- API version in URL: `/api/v1/`
- Major version changes for breaking changes
- Minor version changes backward compatible

### Caching
- Use ETags for conditional requests
- Include `Cache-Control` headers
- Support `If-None-Match` for 304 responses

### Security
- HTTPS required for all endpoints
- Input validation on all parameters
- SQL injection prevention through parameterized queries
- XSS prevention through output encoding
- CSRF tokens for state-changing operations
- Rate limiting to prevent abuse

## 6. Database Indexes

### Critical Indexes for Performance
```sql
-- Books
CREATE INDEX idx_books_isbn ON books(isbn);
CREATE INDEX idx_books_category ON books(category_id);
CREATE INDEX idx_books_title ON books(title);

-- Users
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_membership ON users(membership_expires);

-- Loans
CREATE INDEX idx_loans_user ON loans(user_id);
CREATE INDEX idx_loans_book ON loans(book_id);
CREATE INDEX idx_loans_due_date ON loans(due_date);
CREATE INDEX idx_loans_status ON loans(status);
CREATE INDEX idx_loans_user_status ON loans(user_id, status);

-- Reservations
CREATE INDEX idx_reservations_book ON reservations(book_id);
CREATE INDEX idx_reservations_user ON reservations(user_id);
CREATE INDEX idx_reservations_expiry ON reservations(expiry_date);
CREATE INDEX idx_reservations_status ON reservations(status);

-- Fines
CREATE INDEX idx_fines_user ON fines(user_id);
CREATE INDEX idx_fines_loan ON fines(loan_id);
CREATE INDEX idx_fines_status ON fines(status);

-- Notifications
CREATE INDEX idx_notifications_user ON notifications(user_id);
CREATE INDEX idx_notifications_scheduled ON notifications(scheduled_for);
```

## 7. Example API Workflows

### Workflow 1: User Borrows a Book

1. **User searches for book**
   ```
   GET /api/v1/books?search=gatsby
   ```

2. **User views book details**
   ```
   GET /api/v1/books/{book_id}
   ```

3. **User checks out book** (or librarian does on behalf)
   ```
   POST /api/v1/loans
   {
     "book_id": "uuid",
     "user_id": "uuid"
   }
   ```

4. **System automatically:**
   - Decrements `available_copies`
   - Creates loan record with due date
   - Schedules reminder notification for 3 days before due
   - Returns confirmation

### Workflow 2: Book Becomes Overdue

1. **Daily job runs at 1:00 AM**
   - Identifies overdue loans
   - Calculates fines

2. **System creates fine record**
   ```sql
   INSERT INTO fines (loan_id, user_id, amount, reason, status)
   VALUES (loan_id, user_id, days_overdue * 0.50, 'overdue', 'pending')
   ```

3. **System creates notification**
   ```sql
   INSERT INTO notifications (user_id, type, title, message, scheduled_for)
   VALUES (user_id, 'overdue', 'Overdue Book', message_text, NOW())
   ```

4. **System sends email**
   - Uses notification template
   - Includes fine amount and return instructions

### Workflow 3: Book Return with Reservation Queue

1. **Librarian processes return**
   ```
   POST /api/v1/loans/{loan_id}/return
   ```

2. **System checks for reservations**
   ```sql
   SELECT * FROM reservations 
   WHERE book_id = ? AND status = 'pending'
   ORDER BY reservation_date ASC
   LIMIT 1
   ```

3. **If reservation exists:**
   - Update reservation status to 'fulfilled'
   - Set aside book (don't increment available_copies)
   - Create notification
   - Send "Book Available" email
   - Set 3-day pickup expiry

4. **If no reservation:**
   - Increment `available_copies`
   - Book becomes available for checkout

### Workflow 4: Reservation Expiry and Queue Processing

1. **Daily job runs at 2:00 AM**
   - Finds expired reservations

2. **For each expired reservation:**
   - Update status to 'expired'
   - Increment `available_copies`

3. **Check for next reservation in queue**
   ```sql
   SELECT * FROM reservations 
   WHERE book_id = ? AND status = 'pending'
   ORDER BY reservation_date ASC
   LIMIT 1
   ```

4. **Process next reservation** (same as Workflow 3, step 3)

## 8. Testing and Quality Assurance

### Functional Requirements Testing
- ✅ All CRUD operations work correctly
- ✅ Business rules are enforced (loan limits, renewal restrictions, etc.)
- ✅ Proper error handling and validation
- ✅ Authentication and authorization work correctly
- ✅ Pagination, filtering, and sorting work correctly

### Non-Functional Requirements Testing
- ✅ API response time < 200ms for simple queries
- ✅ API response time < 500ms for complex queries with joins
- ✅ System handles 1000 concurrent users
- ✅ Background jobs complete within time window
- ✅ Database queries use indexes efficiently
- ✅ Rate limiting works correctly
- ✅ Caching reduces database load

### Edge Cases and Error Scenarios
- ✅ User tries to checkout unavailable book
- ✅ User tries to renew overdue loan
- ✅ User tries to reserve already available book
- ✅ Concurrent checkout attempts for last copy
- ✅ Invalid authentication tokens
- ✅ Malformed request bodies
- ✅ Database connection failures

### Security Testing
- ✅ SQL injection prevention
- ✅ XSS prevention
- ✅ CSRF protection
- ✅ Authentication bypass attempts
- ✅ Authorization bypass attempts
- ✅ Rate limit enforcement

## 9. Documentation and API Description

### API Documentation Format
The API uses **OpenAPI 3.0** specification for documentation.

### Documentation Includes:
- Complete endpoint descriptions
- Request/response examples
- Parameter definitions with types and constraints
- Authentication requirements
- Error response examples
- Rate limiting information
- Pagination details

### Interactive Documentation
- Swagger UI available at `/api/v1/docs`
- Allows testing endpoints directly
- Auto-generated from OpenAPI spec

### SDK and Client Libraries
- JavaScript/TypeScript client
- Python client
- Java client
- Each includes:
  - Typed interfaces
  - Authentication handling
  - Error handling
  - Retry logic

## 10. Meaningful HTTP Status Codes

### Success Codes (2xx)
- **200 OK**: GET requests, PATCH updates, successful actions
- **201 Created**: POST requests creating new resources
  - Includes `Location` header with new resource URL
- **204 No Content**: Successful DELETE operations

### Client Error Codes (4xx)
- **400 Bad Request**: Malformed request, invalid JSON
- **401 Unauthorized**: Missing or invalid authentication token
- **403 Forbidden**: Valid authentication but insufficient permissions
  - Example: Regular user trying to access admin endpoint
- **404 Not Found**: Resource doesn't exist
  - Book ID not found, User ID not found
- **409 Conflict**: Business rule violation
  - Book already checked out by this user
  - Cannot delete category with books
- **422 Unprocessable Entity**: Valid request but business logic error
  - User has too many overdue books
  - Book has reservations, cannot be renewed
  - Fine amount exceeds threshold
- **429 Too Many Requests**: Rate limit exceeded
  - Includes `Retry-After` header

### Server Error Codes (5xx)
- **500 Internal Server Error**: Unexpected server error
- **503 Service Unavailable**: Database down, maintenance mode

## 11. Rich Data Model Application

### Complex Queries Supported

#### Get User's Complete Library Activity
```
GET /api/v1/users/{user_id}/activity
```
Returns:
- Current loans with days until due
- Loan history
- Current reservations with queue position
- Outstanding fines with details
- Notification history

#### Search Books with Advanced Filters
```
GET /api/v1/books?category_id=uuid&author=Fitzgerald&publication_year_min=1920&publication_year_max=1930&available_only=true
```

#### Get Overdue Loans with User and Book Details
```
GET /api/v1/loans?status=overdue&sort=due_date&order=asc
```
Returns enriched data with:
- Full book details
- User contact information
- Days overdue
- Current fine amount
- Previous overdue count

### Data Consistency

#### Transaction Management
- Book checkout is atomic: decrement available_copies + create loan
- Book return with reservations: update loan + update reservation + send notification
- Fine payment: update fine status + update user total_fines

#### Referential Integrity
- Foreign key constraints enforced
- Cascade deletes where appropriate
- Soft deletes for user and book records

## 12. Authentication Methods

### JWT-Based Authentication

#### Login Process
```
POST /api/v1/auth/login
{
  "email": "john.doe@example.com",
  "password": "password123"
}
```
**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "user": {
    "user_id": "uuid",
    "email": "john.doe@example.com",
    "name": "John Doe",
    "role": "user"
  }
}
```

#### Token Refresh
```
POST /api/v1/auth/refresh
{
  "refresh_token": "eyJhbGciOiJIUzI1NiIs..."
}
```

#### Authorization Levels
- **User**: Can manage own loans, reservations, fines
- **Librarian**: Can manage all loans, users (read), books (read)
- **Admin**: Full access to all endpoints including create/update/delete

### API Key Authentication (for integrations)
- Long-lived API keys for system integrations
- Separate rate limits
- Scoped permissions

## 13. Caching Strategy

### Cache-Control Headers
```
Cache-Control: public, max-age=300
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
```

### Cacheable Endpoints
- **GET /api/v1/books** - Cache for 5 minutes
- **GET /api/v1/categories** - Cache for 1 hour
- **GET /api/v1/books/{book_id}** - Cache for 5 minutes with ETag

### Cache Invalidation
- Book updates invalidate book detail cache
- Loan creation invalidates book availability
- Category changes invalidate category tree

## Conclusion

This REST API design provides a comprehensive, scalable, and maintainable solution for a library management system. It includes:

✅ **Complete entity model** with proper relationships
✅ **Rich API operations** covering all library functions
✅ **Robust scheduling system** for automated tasks
✅ **Comprehensive error handling** with meaningful status codes
✅ **Strong security** with authentication and authorization
✅ **Performance optimization** through indexing and caching
✅ **Clear documentation** following OpenAPI standards
✅ **Professional notification system** for user engagement

The API supports both user-facing operations (search, borrow, reserve) and administrative functions (reporting, fine management, user administration), making it suitable for production deployment in a real library system.
