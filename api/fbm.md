````markdown
# Order Issue Service

This service manages **order-related support issues** coming from multiple channels:

- Freshdesk
- TikTok
- Email
- Other sources

Each issue is stored in MongoDB via Mongoose and assigned an auto-incrementing `issueId`.

---

## Tech Stack

- Node.js
- Express (assumed)
- MongoDB
- Mongoose

---

## Data Model

### `OrderIssue` Schema

```js
{
  OrderId: String,
  issueId: String,           // Auto-generated (starts from 1600000, increments by 1)
  ticketId: String,          // External ticket ID (Freshdesk, TikTok, etc.)
  channel: String,           // e.g. "freshdesk", "tiktok", "email", "other"
  source: String,            // Same as above or more specific source name
  items: [
    {
      sku: String,
      qty: Number
    }
  ],
  trackingNumber: [String],
  picker: String,
  packer: String,
  palletter: String,

  customerName: String,
  customerEmail: String,

  issueType: String,
  status: String,            // default: "open"
  customerReachedAt: Date,   // when customer first reached out
  shipped_at: Date,
  notes: String,

  discussion: [
    {
      userName: String,
      message: String,
      created_at: Date
    }
  ],

  createdBy: String,
  issueResolvedAt: Date,

  // Mongoose timestamps
  createdAt: Date,
  updatedAt: Date
}
````

### `IssueId` Counter Schema

Used internally to generate `issueId`:

```js
{
  id: String,                 // "issue_id"
  seq: Number                 // starts at 1600000
}
```

On every new `OrderIssue` (without an `issueId`), `seq` is incremented by 1 and saved as `issueId` (as a string).

---

## Base URL

Assuming the API is served under:

```text
/api
```

All endpoints below are relative to this base.

---

## Endpoints

### 1. Create Issue

**URL**

```http
POST /api/order-issue/create
```

**Description**

Create a new order issue.
`issueId` is generated automatically by the backend – **do not send it** in the request.

The required/expected fields vary slightly by `source`.

#### Common Fields (all sources)

```json
{
  "OrderId": "string",
  "channel": "freshdesk | tiktok | email | other",
  "source": "freshdesk | tiktok | email | other",
  "items": [
    {
      "sku": "string",
      "qty": 1
    }
  ],
  "trackingNumber": ["TRK123"],
  "picker": "John",
  "packer": "Jane",
  "palletter": "Jim",
  "customerName": "Customer Name",
  "customerEmail": "customer@example.com",
  "issueType": "string (e.g. 'wrong item', 'damaged')",
  "status": "open | pending | done",
  "notes": "Free text notes",
  "createdBy": "agentName"
}
```

#### Source-specific Notes

##### a) `source = "freshdesk"`

Fields:

* `ticketId` (required)
* `issueType` (required)

Example body:

```json
{
  "OrderId": "111-0944945-3989852",
  "channel": "freshdesk",
  "source": "freshdesk",
  "ticketId": "18362",
  "issueType": "delivery issue",
  "customerName": "Josh",
  "customerEmail": "josh@example.com",
  "customerReachedAt": "2025-11-19T22:36:28Z",
  "status": "open"
}
```

You can obtain ticket information from:

```http
GET /api/issue-ticket/:ticketId
```

(see section **2. Get Ticket Info (by ticketId)**)

---

##### b) `source = "tiktok"`

Fields:

* `ticketId` (required) – TikTok ticket id
* `issueType` (required)
* `customerReachedAt` – when customer first reached out

Example body:

```json
{
  "OrderId": "TT-12345",
  "channel": "tiktok",
  "source": "tiktok",
  "ticketId": "TTK-99999",
  "issueType": "missing item",
  "customerName": "TikTok User",
  "customerReachedAt": "2025-11-21T10:00:00Z",
  "status": "open"
}
```

---

##### c) `source = "email"` or `source = "other"`

Fields:

* `source`: `"email"` or `"other"`
* `issueType` (required)
* `customerReachedAt` or similar “first reached” time

Example body:

```json
{
  "OrderId": "OTH-123",
  "channel": "email",
  "source": "email",
  "issueType": "refund request",
  "customerName": "Email Customer",
  "customerEmail": "customer@example.com",
  "customerReachedAt": "2025-11-20T09:15:00Z",
  "status": "open"
}
```

---

**Success Response**

```json
{
  "ok": true,
  "data": {
    "_id": "mongoObjectId",
    "issueId": "1600001",
    "OrderId": "111-0944945-3989852",
    "ticketId": "18362",
    "source": "freshdesk",
    "status": "open",
    "createdAt": "2025-11-20T10:00:00.000Z",
    "updatedAt": "2025-11-20T10:00:00.000Z",
    "__v": 0
  }
}
```

---

### 2. Get Ticket Info (by `ticketId`)

**URL**

```http
GET /api/issue-ticket/:ticketId
```

**Description**

Fetch ticket details from the external system (e.g. Freshdesk).

**Example Response**

```json
{
  "ok": true,
  "data": {
    "subject": "Inquiry from Amazon customer Josh (Order: 111-0944945-3989852)",
    "product_id": 154000093699,
    "id": 18362,
    "status": 2,
    "statusText": "opening",
    "created_at": "2025-11-19T22:36:28Z"
  }
}
```

> Note: The exact fields depend on the external integration (Freshdesk/TikTok/etc.).

---

### 3. Update Issue Status

**URL**

```http
PUT /api/order-issue/update/:OrderId
```

**Description**

Update the `status` of an issue by its `OrderId`.

**Request Params**

* `OrderId` – the `OrderId` of the issue to update.

**Request Body**

```json
{
  "status": "done"
}
```

> You can use other statuses as needed (e.g. `"open"`, `"pending"`), depending on your business logic.

**Example**

```http
PUT /api/order-issue/update/111-0944945-3989852
Content-Type: application/json

{
  "status": "done"
}
```

**Success Response**

```json
{
  "ok": true,
  "data": {
    "_id": "mongoObjectId",
    "issueId": "1600001",
    "OrderId": "111-0944945-3989852",
    "status": "done",
    "issueResolvedAt": "2025-11-21T11:00:00.000Z",
    "updatedAt": "2025-11-21T11:00:00.000Z"
  }
}
```

---

### 4. Add Comment / Discussion to an Issue

**URL**

```http
PUT /api/order-issue/discussion/:OrderId
```

**Description**

Append a new message to the `discussion` array of a given issue.

**Request Params**

* `OrderId` – the `OrderId` of the issue.

**Request Body**

```json
{
  "username": "Agent Name",
  "message": "This is a follow-up note."
}
```

The backend should internally map this to:

```js
discussion.push({
  userName: username,
  message,
  created_at: new Date()
})
```

**Example**

```http
PUT /api/order-issue/discussion/111-0944945-3989852
Content-Type: application/json

{
  "username": "John Agent",
  "message": "Customer confirmed they received the replacement."
}
```

**Success Response**

```json
{
  "ok": true,
  "data": {
    "_id": "mongoObjectId",
    "issueId": "1600001",
    "OrderId": "111-0944945-3989852",
    "discussion": [
      {
        "userName": "John Agent",
        "message": "Customer confirmed they received the replacement.",
        "created_at": "2025-11-21T12:00:00.000Z"
      }
    ],
    "updatedAt": "2025-11-21T12:00:00.000Z"
  }
}
```

---

## Notes & Conventions

* `issueId` is automatically generated and **monotonically increases** starting from `1600000`.
* `status` default is `"open"` when an issue is created.
* All date fields are stored as ISO 8601 strings in UTC.
* `createdAt` and `updatedAt` are handled automatically by Mongoose’s `timestamps: true`.

---

## Example Flow (Freshdesk)

1. Customer creates a Freshdesk ticket.

2. Your backend calls:

   ```http
   GET /api/issue-ticket/18362
   ```

   to fetch ticket details.

3. Backend then creates a new issue:

   ```http
   POST /api/order-issue/create
   ```

   with data from Freshdesk.

4. Agents update issue `status` when done:

   ```http
   PUT /api/order-issue/update/111-0944945-3989852
   {
     "status": "done"
   }
   ```

5. Agents add internal notes via:

   ```http
   PUT /api/order-issue/discussion/111-0944945-3989852
   {
     "username": "John Agent",
     "message": "Replacement order shipped."
   }
   ```

---

```
```
