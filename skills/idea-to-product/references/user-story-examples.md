# User Story Examples by Feature Type

## Authentication

### US: User Registration
**As a** new visitor  
**I want to** create an account with my email  
**So that** I can save my progress and access features

**Acceptance Criteria:**
- [ ] Email field validates format before submission
- [ ] Password requires minimum 8 characters
- [ ] System sends verification email within 30 seconds
- [ ] User sees confirmation page immediately after submit
- [ ] Duplicate email shows specific error: "Account exists. Try logging in."

### US: Password Reset
**As a** forgetful user  
**I want to** reset my password via email  
**So that** I can regain access to my account

**Acceptance Criteria:**
- [ ] Reset link expires after 1 hour
- [ ] Link can only be used once
- [ ] User receives email within 60 seconds
- [ ] Invalid/expired link shows clear error message

---

## Data Entry

### US: Create New Record
**As a** [user type]  
**I want to** add a new [record type]  
**So that** I can track [thing]

**Acceptance Criteria:**
- [ ] Form validates required fields before submission
- [ ] Success shows toast notification: "[Record] created"
- [ ] User redirected to [view] after creation
- [ ] New record appears in list immediately (optimistic UI)

### US: Bulk Import
**As a** power user  
**I want to** import [records] from CSV  
**So that** I can migrate existing data quickly

**Acceptance Criteria:**
- [ ] System accepts CSV files up to 10MB
- [ ] Preview shows first 5 rows before import
- [ ] Validation errors displayed per-row
- [ ] Progress indicator for imports >100 rows
- [ ] Success summary shows: X imported, Y skipped, Z errors

---

## Search & Filter

### US: Search Records
**As a** user with many [records]  
**I want to** search by [field]  
**So that** I can find specific items quickly

**Acceptance Criteria:**
- [ ] Search triggers after 300ms debounce
- [ ] Results update without page refresh
- [ ] Empty results show "No [records] matching '[query]'"
- [ ] Search highlights matching text in results
- [ ] Keyboard shortcut (Cmd+K) focuses search

### US: Filter by Category
**As a** organized user  
**I want to** filter [records] by [category]  
**So that** I can focus on relevant items

**Acceptance Criteria:**
- [ ] Filters can be combined (AND logic)
- [ ] Active filters displayed as removable chips
- [ ] "Clear all" resets to default view
- [ ] Filter state persists in URL (shareable)
- [ ] Count updates to reflect filtered results

---

## Notifications

### US: Email Notification
**As a** busy user  
**I want to** receive email when [event]  
**So that** I don't miss important updates

**Acceptance Criteria:**
- [ ] Email sent within 5 minutes of trigger
- [ ] Email includes direct link to relevant content
- [ ] Unsubscribe link works in single click
- [ ] User can configure notification preferences
- [ ] Notification logged in activity feed

---

## Payments

### US: Subscribe to Plan
**As a** free user  
**I want to** upgrade to [plan name]  
**So that** I can access [premium features]

**Acceptance Criteria:**
- [ ] Pricing page shows all plan options
- [ ] Stripe checkout opens in same tab
- [ ] Success redirects to [dashboard] with plan active
- [ ] User receives confirmation email within 1 minute
- [ ] Pro features unlock immediately (no refresh needed)

### US: Cancel Subscription
**As a** paying subscriber  
**I want to** cancel my subscription  
**So that** I stop being charged

**Acceptance Criteria:**
- [ ] Cancel option accessible in settings (not hidden)
- [ ] Confirmation modal explains: "Active until [date]"
- [ ] Access continues until billing period ends
- [ ] Confirmation email sent immediately
- [ ] Option to provide feedback (optional)
