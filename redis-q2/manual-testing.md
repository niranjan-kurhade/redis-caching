# Manual Testing Guide for Books Management API

## Quick Setup

1. **Start the application**:
   ```bash
   npm run dev
   ```

2. **Verify Redis is running**:
   ```bash
   redis-cli ping
   # Should return: PONG
   ```

## Step-by-Step Testing

### 1. User Registration
```bash
curl -X POST http://localhost:3001/auth/signup \
  -H "Content-Type: application/json" \
  -d '{
    "username": "johndoe",
    "email": "john@example.com",
    "password": "password123"
  }'
```
**Expected**: User created, JWT token returned

### 2. User Login
```bash
curl -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com", 
    "password": "password123"
  }'
```
**Expected**: Login successful, JWT token returned
**Copy the token** from the response for next steps!

### 3. Set Token Variable (replace with your actual token)
```bash
export TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### 4. Get Current User Info
```bash
curl -X GET http://localhost:3001/auth/me \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: Current user details

### 5. Get Books (Empty List - Cache Miss)
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: Empty array, `"source": "database"`

### 6. Get Books Again (Cache Hit)
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: Empty array, `"source": "cache"`

### 7. Add First Book
```bash
curl -X POST http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "The Great Gatsby",
    "author": "F. Scott Fitzgerald",
    "isbn": "978-0-7432-7356-5",
    "publishedYear": 1925,
    "genre": "Fiction"
  }'
```
**Expected**: Book created successfully

### 8. Add Second Book
```bash
curl -X POST http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "1984",
    "author": "George Orwell", 
    "publishedYear": 1949,
    "genre": "Dystopian"
  }'
```
**Expected**: Book created successfully

### 9. Get Books After Adding (Cache Miss)
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: 2 books, `"source": "database"` (cache was invalidated)

### 10. Get Books Again (Cache Hit)
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: 2 books, `"source": "cache"`

### 11. Update a Book (use ID from previous response)
```bash
# Replace BOOK_ID with actual ID from step 9
curl -X PUT http://localhost:3001/books/BOOK_ID \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "The Great Gatsby - Updated",
    "author": "F. Scott Fitzgerald",
    "genre": "Classic Literature",
    "publishedYear": 1925
  }'
```
**Expected**: Book updated successfully

### 12. Get Books After Update (Cache Miss)
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: 2 books with updated info, `"source": "database"`

### 13. Submit Bulk Books
```bash
curl -X POST http://localhost:3001/books/bulk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "books": [
      {
        "title": "Harry Potter and the Philosopher'\''s Stone",
        "author": "J.K. Rowling",
        "genre": "Fantasy",
        "publishedYear": 1997
      },
      {
        "title": "The Hobbit", 
        "author": "J.R.R. Tolkien",
        "genre": "Fantasy",
        "publishedYear": 1937
      },
      {
        "title": "Dune",
        "author": "Frank Herbert",
        "genre": "Science Fiction", 
        "publishedYear": 1965
      }
    ]
  }'
```
**Expected**: Books queued for processing message

### 14. Check Bulk Queue Status
```bash
curl -X GET http://localhost:3001/books/queue \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: Shows queued books count and timestamp

### 15. Wait for Bulk Processing
Wait approximately 2+ minutes for the cron job to process the bulk books.

### 16. Check Books After Bulk Processing
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: 5 books total (2 individual + 3 bulk), `"source": "database"`

### 17. Check Queue Status Again
```bash
curl -X GET http://localhost:3001/books/queue \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: `"queued": false` (queue processed and cleared)

### 18. Delete a Book (use ID from step 16)
```bash
# Replace BOOK_ID with actual ID
curl -X DELETE http://localhost:3001/books/BOOK_ID \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: Book deleted successfully

### 19. Final Books Check (Cache Miss)
```bash
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```
**Expected**: 4 books, `"source": "database"` (cache invalidated by delete)

### 20. Health Check
```bash
curl -X GET http://localhost:3001/health
```
**Expected**: API status and configuration info

## Testing Multiple Users

### Create Second User
```bash
curl -X POST http://localhost:3001/auth/signup \
  -H "Content-Type: application/json" \
  -d '{
    "username": "janedoe",
    "email": "jane@example.com", 
    "password": "password456"
  }'
```

### Login as Second User
```bash
curl -X POST http://localhost:3001/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "jane@example.com",
    "password": "password456"
  }'
```

### Set Second Token
```bash
export TOKEN2="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Verify User Isolation
```bash
# Get books for second user (should be empty)
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN2"

# Get books for first user (should still have books)  
curl -X GET http://localhost:3001/books \
  -H "Authorization: Bearer $TOKEN"
```

## Expected Cache Behavior

| Action | Cache Status | Source |
|--------|--------------|---------|
| First GET /books | Miss | database |
| Second GET /books (within 5 min) | Hit | cache |
| After POST book | Miss | database (invalidated) |
| After PUT book | Miss | database (invalidated) |
| After DELETE book | Miss | database (invalidated) |
| After bulk processing | Miss | database (invalidated) |

## Troubleshooting

### Token Issues
- Make sure to copy the full token from login response
- Token expires in 24 hours by default
- Use `export TOKEN="..."` to set environment variable

### Redis Issues
- Check Redis is running: `redis-cli ping`
- Check Redis database: `redis-cli SELECT 1` then `KEYS *`

### Cron Job Issues
- Check console logs for cron execution
- Bulk processing runs every 2 minutes
- Failed jobs remain queued for retry

### Network Issues
- Ensure server is running on http://localhost:3001
- Check firewall settings
- Verify no port conflicts
