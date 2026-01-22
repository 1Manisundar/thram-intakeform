# Admin Portal User Guide

## Overview
This guide explains how to use the Admin Portal to manage insurance verification submissions.

**Portal URL:** https://admin.yourdomain.com (or configured domain)

---

## Getting Started

### 1. Logging In

1. Navigate to the Admin Portal
2. Enter your email address
3. Enter your password
4. Click "Login"

**Forgot Password?**
1. Click "Forgot Password?" link
2. Enter your email address
3. Check your email for reset link
4. Click link and create new password
5. Login with new password

**Session Timeout:**
- Automatic logout after 8 hours of inactivity
- You'll see a warning 5 minutes before timeout
- Save any work before timeout

---

## Dashboard Overview

### Main Dashboard

**Top Metrics (Cards):**
- **Total Submissions:** All-time submission count
- **Pending Review:** Submissions awaiting admin action
- **Verified Today:** Successfully verified today
- **Requires Action:** Needs user to fix something

**Status Distribution (Chart):**
- Visual breakdown of submissions by status
- Click on any segment to filter by that status

**Recent Activity (Table):**
- Latest 20 submissions
- Quick view of status and submission time

---

## Viewing Submissions

### Submissions List

**Filter Options:**
- **Status:** Filter by verification status
- **Date Range:** Show submissions from specific period
- **Requires User Action:** Show only items needing user resubmission
- **Search:** Search by name or member ID

**Sort Options:**
- **Newest First** (default)
- **Oldest First**
- **Priority (High to Low)**
- **Status (A-Z)**

**Pagination:**
- 20 items per page (default)
- Use arrow buttons to navigate pages
- Jump to specific page number

### Status Color Coding

| Color | Status | Meaning |
|-------|--------|---------|
| 🟢 Green | data_verified | Ready to contact user |
| 🔵 Blue | Processing | OCR or API in progress |
| 🟡 Yellow | partial_verification | Incomplete data, review needed |
| 🟡 Yellow | insurance_invalid | Insurance not active |
| 🔴 Red | ocr_failed | Bad image quality |
| 🔴 Red | data_mismatch | Data doesn't match |
| 🟠 Orange | api_error_manual_review | API failed, manual check needed |

---

## Viewing Individual Submissions

### Submission Detail Page

Click on any submission to view full details.

**User Information Section:**
- Full name
- Date of birth
- Contact information (email, phone)
- Submitted member ID
- Insurance payer name

**Verification Results Section:**

**OCR Results:**
- Member ID extracted from card
- Payer name extracted
- Confidence score (0-100%)
- Shows if data matched user input

**Insurance Verification (Stedi):**
- Eligibility status (Active/Inactive)
- Plan name
- Coverage amount
- Copay amount
- Deductible
- Effective dates

**Insurance Card Images:**
- View front and back of insurance card
- Click to enlarge
- Download option available

**Admin Notes:**
- Previous notes from other admins
- Add your own notes
- Timestamped automatically

**Activity Timeline:**
- Submission date/time
- OCR processing time
- Verification completion time
- Admin contact time (if contacted)

---

## Common Workflows

### Workflow 1: Handling Verified Submissions

**Status:** `data_verified` (Green)

**Steps:**
1. Review verification results
2. Check all insurance details are complete
3. Click "Mark as Contacted"
4. Add notes about the conversation
5. Discuss next steps with user (appointment scheduling, etc.)
6. Save notes

**What to verify:**
- ✅ Insurance is active
- ✅ Coverage amount is sufficient
- ✅ Plan details match user's needs
- ✅ No red flags or unusual data

---

### Workflow 2: Handling OCR Failures

**Status:** `ocr_failed` (Red)

**Why it happens:**
- Image too blurry
- Image too dark/bright
- Card partially cut off
- Poor camera quality

**Steps:**
1. View uploaded insurance card images
2. Assess image quality
3. User already notified automatically (email sent)
4. Wait for user to resubmit with clearer images
5. When resubmitted, verification will run again automatically

**Note:** No action needed from admin - system handles notification

---

### Workflow 3: Handling Data Mismatches

**Status:** `data_mismatch` (Red)

**Why it happens:**
- User typo in form
- OCR misread card
- Wrong insurance card uploaded

**What's shown:**
- Submitted Member ID: ABC123
- Extracted Member ID: ABC 123 (space difference)

**Steps:**
1. Compare both values
2. Check insurance card image to verify correct ID
3. User already notified automatically
4. Wait for user to verify and resubmit
5. Add notes if clarification needed

---

### Workflow 4: Handling Invalid Insurance

**Status:** `insurance_invalid` (Yellow)

**Why it happens:**
- Insurance policy expired
- Coverage terminated
- Member no longer eligible

**Steps:**
1. Review Stedi results showing "Inactive" status
2. Click "Mark as Contacted"
3. Call user to discuss options:
   - Cash payment
   - Different insurance
   - Payment plan
4. Add notes about discussion outcome
5. Update status to "Closed" if resolved

---

### Workflow 5: Handling API Errors

**Status:** `api_error_manual_review` (Orange)

**Why it happens:**
- Stedi API timeout (rare)
- Stedi API temporarily unavailable
- Network issues

**Steps:**
1. Review what data is available
2. Click "Retry Verification" button to trigger workflow again
3. If retry fails, manually verify insurance:
   - Call insurance provider directly
   - Use alternative verification method
4. Manually enter verification results in notes
5. Update status to "data_verified" if verified manually

---

## Admin Actions

### Mark as Contacted

**When to use:** After speaking with user

**Steps:**
1. Click "Mark as Contacted" button
2. Add notes about conversation
3. Status automatically updated to `admin_contacted`
4. Timestamp recorded

**What to include in notes:**
- Date/time of contact
- Method (phone/email)
- Summary of discussion
- Next steps agreed upon

---

### Add Notes

**When to use:** Any time you need to document something

**Steps:**
1. Type note in "Add Note" field
2. Click "Save Note"
3. Note appears in timeline with your name and timestamp

**Note Guidelines:**
- Be specific and concise
- Include relevant details only
- Avoid PHI unless necessary for treatment
- Use professional language

**Examples:**
- ✅ "Called patient, scheduled appointment for 1/25"
- ✅ "User confirmed coverage details are correct"
- ❌ "Called about their diabetes" (too much PHI)
- ❌ "idk what to do with this" (unprofessional)

---

### Retry Verification

**When to use:** For `api_error_manual_review` status

**Steps:**
1. Click "Retry Verification" button
2. System triggers n8n workflow again
3. Wait for verification to complete (1-2 minutes)
4. Refresh page to see updated status

**Note:** Only available for API errors, not for user errors (OCR failed, data mismatch)

---

### Assign Priority

**Priority Levels:**
- 🔴 Urgent - Immediate attention needed
- 🟠 High - Review within 24 hours
- 🟢 Normal - Standard processing
- ⚪ Low - Can wait

**When to use:**
- Urgent: Pre-op patients, emergency cases
- High: Next-day appointments
- Normal: Most cases
- Low: Future appointments (>2 weeks out)

**Steps:**
1. Click priority dropdown
2. Select new priority level
3. Saves automatically

---

## Search & Filtering

### Search by Name or Member ID

**Search Box (top right):**
- Type patient name (partial matches work)
- Or type member ID
- Press Enter or click Search
- Results update automatically

**Examples:**
- "John" → Finds "John Doe", "Johnny Smith"
- "ABC123" → Finds exact member ID match

---

### Filter by Status

**Status Filter Dropdown:**
1. Click "All Statuses" dropdown
2. Select specific status
3. List updates to show only that status
4. Clear filter to show all again

**Common Filters:**
- `data_verified` → Ready to contact
- `requires_user_action` → Waiting on user
- `api_error_manual_review` → Needs admin attention

---

### Filter by Date Range

**Date Range Picker:**
1. Click "Date Range" button
2. Select start date
3. Select end date
4. Click "Apply"
5. List shows submissions in that range

**Quick Filters:**
- Today
- Last 7 Days
- Last 30 Days
- This Month
- Custom Range

---

## Reports & Analytics

### Dashboard Metrics

**Overview Cards:**
- Updated in real-time
- Click to filter by that metric
- Shows trend (up/down) compared to previous period

### Export Data

**Export to CSV:**
1. Click "Export" button (top right)
2. Choose filtered or all data
3. CSV file downloads automatically
4. Open in Excel or Google Sheets

**CSV Includes:**
- All submission data
- Current status
- Verification results
- Timestamps

**Use cases:**
- Monthly reports
- Performance tracking
- Data analysis
- Management reviews

---

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `/` | Focus search box |
| `n` | Next page |
| `p` | Previous page |
| `r` | Refresh list |
| `Esc` | Close modal/clear search |

---

## Mobile Usage

**Mobile Responsiveness:**
- Dashboard works on tablets and phones
- Some features condensed for smaller screens
- Best experience on tablet or desktop

**Recommended for mobile:**
- View submissions
- Read details
- Add quick notes

**Not recommended for mobile:**
- Detailed verification review
- Image analysis
- Long note writing

---

## Troubleshooting

### "Session Expired" Error

**Cause:** Logged out due to inactivity or security

**Solution:**
1. Click "Login" button
2. Re-enter credentials
3. You'll return to where you left off

---

### Can't See Verification Details

**Possible Causes:**
- Page still loading (wait a moment)
- Network issue (check connection)
- Submission deleted (rare)

**Solution:**
1. Refresh page (Ctrl+R or Cmd+R)
2. Check internet connection
3. Contact support if persists

---

### Images Not Loading

**Possible Causes:**
- Slow connection
- Image file corrupted
- Browser issue

**Solution:**
1. Wait for image to load
2. Click "Reload Image" if available
3. Try different browser
4. Check with IT if problem continues

---

### Can't Add Notes

**Possible Causes:**
- Session expired
- Network issue
- Permissions issue

**Solution:**
1. Check you're still logged in
2. Refresh page
3. Try again
4. Contact supervisor if issue persists

---

## Best Practices

### Daily Routine

**Start of Day:**
1. Login to portal
2. Review overnight submissions
3. Check high-priority items first
4. Plan which verifications to handle

**Throughout Day:**
1. Monitor new submissions
2. Respond to urgent items quickly
3. Add notes after each patient contact
4. Keep queue organized

**End of Day:**
1. Complete any pending notes
2. Review tomorrow's priorities
3. Logout securely

---

### Data Privacy

**DO:**
- ✅ Lock computer when stepping away
- ✅ Close browser after finishing
- ✅ Only access data needed for your work
- ✅ Log out at end of day

**DON'T:**
- ❌ Share login credentials
- ❌ Leave computer unlocked
- ❌ Access data out of curiosity
- ❌ Discuss PHI in public areas
- ❌ Take screenshots of patient data

---

### Communication Tips

**When Contacting Patients:**
1. Introduce yourself and organization
2. Verify you're speaking to correct person
3. Explain purpose of call
4. Be clear and professional
5. Document conversation in notes

**When Adding Notes:**
- Include only relevant information
- Be objective and factual
- Use clear language
- Avoid personal opinions
- Think "Would I be comfortable if patient saw this?"

---

## Getting Help

### Support Contacts

**Technical Issues:**
- Email: support@yourdomain.com
- Phone: (555) 123-4567
- Hours: Monday-Friday, 9 AM - 5 PM

**Training Questions:**
- Email: training@yourdomain.com
- Schedule: Weekly office hours

**Security Concerns:**
- Email: security@yourdomain.com
- Phone: (555) 999-9999 (24/7)

---

### Training Resources

**Available Resources:**
- Video tutorials (in portal Help menu)
- Quick reference guide (printable PDF)
- Weekly Q&A sessions
- One-on-one training available

---

## Frequently Asked Questions

**Q: How long does verification take?**  
A: Usually 1-2 minutes for automated processing. If issues arise, admin review may be needed.

**Q: What if user uploaded wrong insurance card?**  
A: System will detect mismatch and notify user to upload correct card.

**Q: Can I manually enter verification results?**  
A: Yes, for API errors. Add results in notes and update status to "data_verified".

**Q: How do I know if user has been notified?**  
A: Check status - if `requires_user_action = true`, they've been emailed automatically.

**Q: Can I delete a submission?**  
A: No, for compliance reasons. Mark as "Closed" instead.

**Q: How long are records kept?**  
A: Indefinitely for HIPAA compliance. Archived after 7 years.

**Q: Can I export only verified submissions?**  
A: Yes, use status filter first, then click Export.

**Q: What if I make a mistake in notes?**  
A: Contact supervisor - notes cannot be edited for audit trail integrity.

---

## Document Information
- **Version:** 1.0
- **Last Updated:** January 22, 2026
- **Audience:** Admin Users
- **Status:** Complete