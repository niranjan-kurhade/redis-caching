# Manual Testing Guide - Books Management with Reporting

## Quick Setup Checklist

### 1. Environment Setup
- [ ] Node.js installed
- [ ] Redis server running (`redis-cli ping` should return `PONG`)
- [ ] Dependencies installed (`npm install`)
- [ ] Email configured in `.env` file
- [ ] Server started (`npm run dev`)

### 2. Email Configuration (Gmail)
```bash
# In .env file:
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_gmail_app_password  # Not your regular password!
FROM_EMAIL=your_email@gmail.com
```

**Gmail App Password Setup:**
1. Enable 2FA on Gmail account
2. Go to Google Account Settings → Security → 2-Step Verification → App passwords
3. Generate password for "Mail" application
4. Use generated password in `SMTP_PASS`

## Test Scenarios

### Scenario A: Single User Complete Flow

#### Step 1: User Registration
```bash
curl -X POST http://localhost:3002/auth/signup \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice_reader",
    "email": "your_email@gmail.com",
    "password": "password123"
  }'
```

**Expected Response:**
```json
{
  "success": true,
  "message": "User created successfully",
  "data": { "id": "...", "username": "alice_reader", "email": "your_email@gmail.com" },
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**📋 Copy the token for next steps!**

#### Step 2: Set Token Variable
```bash
export TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

#### Step 3: Submit Bulk Books
```bash
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "books": [
      {
        "title": "The Fellowship of the Ring",
        "author": "J.R.R. Tolkien",
        "genre": "Fantasy",
        "publishedYear": 1954
      },
      {
        "title": "Dune",
        "author": "Frank Herbert", 
        "genre": "Science Fiction",
        "publishedYear": 1965
      },
      {
        "title": "The Great Gatsby",
        "author": "F. Scott Fitzgerald",
        "genre": "Literature",
        "publishedYear": 1925
      }
    ]
  }'
```

**Expected Response:**
```json
{
  "success": true,
  "message": "Books queued for bulk processing. You will receive an email report within 7 minutes.",
  "data": {
    "requestId": "abc-123-def",
    "queuedBooks": 3,
    "estimatedProcessingTime": "2 minutes",
    "estimatedReportTime": "7 minutes"
  }
}
```

#### Step 4: Monitor Processing Status
```bash
# Check immediately (should show "queued")
curl -X GET http://localhost:3002/books/queue \
  -H "Authorization: Bearer $TOKEN"

# Wait 2+ minutes, check again (should show "processed_awaiting_report")
curl -X GET http://localhost:3002/books/queue \
  -H "Authorization: Bearer $TOKEN"

# Wait 5+ more minutes, check again (should show "completed")
curl -X GET http://localhost:3002/books/queue \
  -H "Authorization: Bearer $TOKEN"
```

#### Step 5: Verify Books Added
```bash
curl -X GET http://localhost:3002/books \
  -H "Authorization: Bearer $TOKEN"
```

**Expected:** 3 books in response (or 2-3 if some failed randomly)

#### Step 6: Check Email
- Check your email inbox for PDF report
- Report should arrive within 7 minutes of submission
- Verify PDF contains processing summary

---

### Scenario B: Multi-User Concurrent Testing

#### Create Multiple Users
```bash
# User 1
curl -X POST http://localhost:3002/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"user1","email":"your_email+user1@gmail.com","password":"pass123"}'

# User 2  
curl -X POST http://localhost:3002/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"user2","email":"your_email+user2@gmail.com","password":"pass123"}'

# User 3
curl -X POST http://localhost:3002/auth/signup \
  -H "Content-Type: application/json" \
  -d '{"username":"user3","email":"your_email+user3@gmail.com","password":"pass123"}'
```

#### Set Tokens for Each User
```bash
export TOKEN1="..." # Copy from user1 signup response
export TOKEN2="..." # Copy from user2 signup response  
export TOKEN3="..." # Copy from user3 signup response
```

#### Submit Concurrent Bulk Requests
```bash
# User 1 - Fantasy Books
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer $TOKEN1" \
  -H "Content-Type: application/json" \
  -d '{"books":[{"title":"Harry Potter 1","author":"J.K. Rowling"},{"title":"Lord of the Rings","author":"J.R.R. Tolkien"}]}' &

# User 2 - Mystery Books  
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer $TOKEN2" \
  -H "Content-Type: application/json" \
  -d '{"books":[{"title":"Sherlock Holmes","author":"Arthur Conan Doyle"},{"title":"Agatha Christie","author":"Murder Mystery"}]}' &

# User 3 - Sci-Fi Books
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer $TOKEN3" \
  -H "Content-Type: application/json" \
  -d '{"books":[{"title":"Foundation","author":"Isaac Asimov"},{"title":"Ender Game","author":"Orson Scott Card"}]}' &

wait
```

#### Monitor All Users
```bash
# Check system stats
curl -X GET http://localhost:3002/admin/stats

# Check each user's queue status
curl -X GET http://localhost:3002/books/queue -H "Authorization: Bearer $TOKEN1"
curl -X GET http://localhost:3002/books/queue -H "Authorization: Bearer $TOKEN2" 
curl -X GET http://localhost:3002/books/queue -H "Authorization: Bearer $TOKEN3"
```

---

### Scenario C: Error Testing

#### Test Invalid Data
```bash
# Missing required fields
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"books":[{"title":"","author":""}]}'

# Too many books (limit is 100)
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"books":'$(python3 -c "import json; print(json.dumps([{'title': f'Book {i}', 'author': f'Author {i}'} for i in range(101)]))")'"}'
```

#### Test Authentication
```bash
# No token
curl -X POST http://localhost:3002/books/bulk \
  -H "Content-Type: application/json" \
  -d '{"books":[{"title":"Test","author":"Test"}]}'

# Invalid token  
curl -X POST http://localhost:3002/books/bulk \
  -H "Authorization: Bearer invalid_token" \
  -H "Content-Type: application/json" \
  -d '{"books":[{"title":"Test","author":"Test"}]}'
```

---

## Expected Timeline & Behavior

### Processing Timeline
| Time | Event | Status Check Response |
|------|-------|----------------------|
| T+0 | Submit bulk books | `{"queued": true, "stage": "queued"}` |
| T+2min | Bulk processing runs | `{"processing": true, "stage": "processed_awaiting_report"}` |
| T+5min | Report generation runs | `{"queued": false, "processing": false}` |
| T+7min | Email delivered | Check inbox for PDF |

### Email Report Contents
The PDF report should contain:
- **Header**: Books Management Inc. branding
- **Report Details**: User info, request ID, timestamps
- **Processing Summary**: Success/failure counts, success rate
- **Timeline**: Queue time, processing time, duration  
- **Failed Books**: Details of any failures (if applicable)
- **Footer**: Generation timestamp

### Sample Email Subject
```
📚 Bulk Books Processing Report - 3/3 Successful
```

### Sample PDF Report Structure
```
Books Management Inc.
123 Library Street, Book City, BC 12345

Bulk Book Processing Report
═══════════════════════════════════════

Report Details
─────────────────
Report ID: abc-123-def-456
User: alice_reader  
Email: alice@example.com
Generated: October 13th 2025, 3:45:23 pm

Processing Summary
─────────────────
Total Books Submitted: 3
Successfully Processed: 3 ✓
Failed to Process: 0
Success Rate: 100.0%

Processing Timeline  
─────────────────
Queued At: October 13th 2025, 3:40:15 pm
Processed At: October 13th 2025, 3:42:30 pm
Processing Duration: 2 minutes
```

---

## Monitoring Commands

### Real-Time Monitoring
```bash
# Watch system stats (run in separate terminal)
watch -n 5 "curl -s http://localhost:3002/admin/stats | jq '.data'"

# Monitor Redis keys
redis-cli --scan --pattern "bulk_*" | xargs -I {} redis-cli TTL {}

# Watch server logs
tail -f server.log  # if logging to file
```

### Debug Commands
```bash
# List all Redis keys
redis-cli KEYS "*"

# Check specific user's keys
redis-cli KEYS "*user:YOUR_USER_ID*"

# View bulk queue content
redis-cli GET "bulk_books:user:YOUR_USER_ID"

# View processing status
redis-cli GET "bulk_status:user:YOUR_USER_ID"
```

---

## Troubleshooting

### Email Issues
```bash
# Test email configuration
curl http://localhost:3002/health | jq '.emailConfigured'

# Check SMTP settings
echo "SMTP_USER: $SMTP_USER"
echo "SMTP_PASS: [hidden]"
```

**Common Email Fixes:**
- Verify Gmail App Password (not regular password)
- Check spam/junk folder
- Ensure 2FA enabled on Gmail
- Try different email provider

### Cron Job Issues
```bash
# Check if cron jobs are enabled
curl http://localhost:3002/health | jq '.bulkProcessing, .reportGeneration'
```

**Common Cron Fixes:**
- Verify environment variables are set
- Check server logs for cron execution
- Ensure Redis is running
- Wait full cycle times (2min + 5min)

### Processing Issues
```bash
# Check for stuck jobs
curl http://localhost:3002/admin/stats | jq '.data.pendingBulkJobs, .data.pendingReports'

# Clear Redis if needed (CAUTION: deletes all data)
redis-cli FLUSHDB
```

---

## Success Criteria

✅ **Test Passes If:**
- [ ] Users can register successfully
- [ ] Bulk books are queued immediately  
- [ ] Processing completes within 2-4 minutes
- [ ] PDF reports generated successfully
- [ ] Emails delivered with PDF attachments
- [ ] Multiple users processed independently
- [ ] No data cross-contamination between users
- [ ] Error handling works for invalid data
- [ ] System stats update correctly

📧 **Email Success Indicators:**
- [ ] Email received within 7 minutes
- [ ] PDF attachment present and readable
- [ ] Processing statistics accurate
- [ ] User-specific data only
- [ ] Professional formatting

🧪 **Multi-User Success:**
- [ ] All users receive separate emails
- [ ] Each user's books isolated correctly
- [ ] Concurrent processing doesn't interfere
- [ ] Individual success/failure rates
- [ ] Proper request ID tracking
