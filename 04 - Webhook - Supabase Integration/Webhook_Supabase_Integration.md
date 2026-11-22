# Webhook Learning – Supabase Integration

## 1. Overview

This n8n workflow demonstrates a webhook-based customer registration process using **Supabase** as the data store.

An external application sends customer details to an n8n Webhook using an HTTP `POST` request. The workflow checks the customer email against the `webhook_customers` table, creates a record when no matching customer is found, and returns a JSON response to the calling application.

The workflow also contains an email-required validation branch and a duplicate-customer response branch. The current exported workflow has a connection/condition issue in the email-validation path; this is described in the limitations section and should be corrected before relying on the workflow in production.

## 2. Objectives

- Receive customer registration data through an HTTP webhook.
- Validate that an email address has been provided.
- Check Supabase for an existing customer with the same email.
- Insert a new customer record when the email is not already registered.
- Return structured JSON responses with appropriate HTTP status codes.
- Demonstrate conditional routing and database operations in n8n.

## 3. Workflow Architecture

### High-level flow represented by the exported workflow

```text
                    +----------------------+
                    | Webhook (POST)       |
                    | /supabase-customer   |
                    +----------------------+
                       |               |
                       |               v
                       |        +-------------+
                       |        | If1         |
                       |        | Email empty?|
                       |        +-------------+
                       |          | true  | false
                       |          |       |
                       |          v       v
                       |     Get many   Respond to
                       |     rows       Webhook2
                       |       |        (400)
                       |       v
                       |     +-------------+
                       |     | If          |
                       |     | Result email|
                       |     | empty?      |
                       |     +-------------+
                       |       | true | false
                       |       |      |
                       |       v      v
                       |   Create a  Respond to
                       |   row       Webhook
                       |      |      (409)
                       |      v
                       |   Respond to
                       |   Webhook1
                       |   (200)
                       |
                       +----> Get many rows
                              (direct parallel connection)
```

**Important:** The diagram reflects the connections and conditions in the supplied JSON, including the direct Webhook-to-`Get many rows` connection. That direct connection means the database lookup can run independently of the email-validation branch. The current wiring therefore does not enforce email validation before the lookup.

### Intended business logic

The intended registration logic is:

1. Receive the POST request.
2. Check whether the email field is present.
3. If the email is missing, return HTTP `400`.
4. If the email is present, normalize it and search Supabase for a matching record.
5. If a matching customer exists, return HTTP `409`.
6. If no matching customer exists, insert the customer and return HTTP `200`.

## 4. Node-by-Node Responsibilities

| Node | Responsibility |
|---|---|
| Webhook | Receives HTTP `POST` requests at the `supabase-customer` path. It uses response mode `Using Respond to Webhook node`, so downstream Respond to Webhook nodes return the response. |
| If1 | Checks whether `body.email`, after trimming whitespace, is empty. In the supplied workflow, its true output is connected to `Get many rows`, while its false output is connected to the missing-email response. This is reversed for the intended validation logic. |
| Respond to Webhook2 | Returns a JSON error message, `Email is required`, with HTTP status `400`. |
| Get many rows | Queries the Supabase `webhook_customers` table for a record whose `email` equals the incoming email after lowercasing and trimming. The query is limited to one row. |
| If | Checks whether the returned item's `email` field is empty. An empty result follows the true output to `Create a row`; a populated email follows the false output to the duplicate response. |
| Create a row | Inserts the incoming customer's `name` and normalized `email` into `webhook_customers`. |
| Respond to Webhook1 | Returns HTTP `200` with a success message and the created customer data. |
| Respond to Webhook | Returns HTTP `409` with the message `Customer already registered`. |

## 5. Webhook Configuration

The Webhook node is configured as follows:

| Setting | Configured value |
|---|---|
| HTTP method | `POST` |
| Path | `supabase-customer` |
| Response mode | Using Respond to Webhook node |
| Workflow name | `Webhook Learning - Supabase Integration` |

The exact production or test URL is generated by the n8n instance. Use the URL displayed by your own Webhook node; do not assume that a URL from another n8n instance will work.

## 6. Expected Request Payload

The workflow reads the customer fields from the incoming request body.

Example JSON payload:

```json
{
  "name": "Test Customer",
  "email": "testcustomer@example.com"
}
```

The Supabase lookup and insert expressions normalize the email using:

```javascript
$('Webhook').item.json.body.email.toLowerCase().trim()
```

This converts the email to lowercase and removes leading/trailing whitespace before lookup or insertion.

The `name` field is mapped from:

```javascript
$('Webhook').item.json.body.name
```

The exported workflow does not show separate validation for a missing or blank `name` field.

## 7. Supabase Table Configuration

The workflow uses the Supabase table:

```text
webhook_customers
```

The configured fields are:

| Column | Usage |
|---|---|
| `name` | Stores the customer's name. |
| `email` | Stores the customer's normalized email address and is used for duplicate lookup. |

The exported workflow shows the table name and field mappings. It does not include the table's SQL definition, column data types, constraints, or indexes.

For reliable duplicate prevention, the `email` column should also have a database-level unique constraint or unique index. A workflow lookup alone cannot prevent two simultaneous requests from both passing the check before either inserts a row.

## 8. Response Structure

### Successful registration

The `Respond to Webhook1` node is configured to return HTTP `200`:

```json
{
  "success": true,
  "message": "Customer registered",
  "customer": {
    "...": "Supabase row returned by the Create a row node"
  }
}
```

The actual fields inside `customer` depend on the row returned by Supabase.

### Customer already registered

The `Respond to Webhook` node returns HTTP `409`:

```json
{
  "success": false,
  "message": "Customer already registered"
}
```

### Email missing

The `Respond to Webhook2` node returns HTTP `400`:

```json
{
  "success": false,
  "message": "Email is required"
}
```

**Current behaviour warning:** Because the `If1` branches are connected opposite to the intended validation logic, a request with a non-empty email follows the configured `400` response branch, while an empty email follows the lookup branch. The direct Webhook-to-`Get many rows` connection also allows the lookup to run independently. Correct the wiring and condition before treating these responses as the actual reliable behaviour.

## 9. Technology Stack and Integrations

- **n8n** — workflow orchestration and conditional routing.
- **n8n Webhook node** — receives customer registration requests over HTTP.
- **Supabase** — stores customer records in the `webhook_customers` table.
- **HTTP JSON responses** — returns registration results to the calling application.

## 10. Setup and Configuration

1. Import the workflow JSON into your n8n instance.
2. Create or select a Supabase project.
3. Create the `webhook_customers` table with the required `name` and `email` columns.
4. Configure the Supabase credential in the `Get many rows` and `Create a row` nodes.
5. Verify the table name and field mappings.
6. Correct the email-validation branch and remove the direct Webhook-to-`Get many rows` connection so the lookup runs only after validation succeeds.
7. Configure the Webhook node and copy the test or production URL from your n8n instance.
8. Test the missing-email, new-customer, and duplicate-customer cases before activating the workflow.

Do not export or commit Supabase API keys, access tokens, or other credentials to a public repository. The workflow export may contain credential references or instance metadata; review the JSON before publishing it.

## 11. Important Configuration Notes and Limitations

### Email-validation branch is reversed

The `If1` node checks whether the trimmed email is empty:

```javascript
{{ ($json.body.email || '').trim() }}
```

Its true output is currently connected to `Get many rows`, and its false output is connected to `Respond to Webhook2`.

For the intended behaviour, connect:

- **True (email is empty)** → `Respond to Webhook2` (HTTP 400).
- **False (email is present)** → `Get many rows`.

### Lookup is connected directly to the Webhook

The Webhook node has two outgoing connections: one to `If1` and another directly to `Get many rows`. This allows the lookup to run without waiting for the email-validation result.

For sequential validation, remove the direct Webhook-to-`Get many rows` connection. The lookup should receive input only from the valid-email branch of `If1`.

### Duplicate check depends on the returned row

The second IF node checks whether `$json.email` is empty. The workflow is configured with `alwaysOutputData: true` on `Get many rows`, which is intended to allow the no-match path to continue to the IF node. Confirm this behaviour with test executions in your n8n version.

### Database-level uniqueness is recommended

A lookup followed by an insert is vulnerable to concurrent requests. Add a unique constraint/index on the email column in Supabase and handle any resulting unique-constraint error to make duplicate prevention reliable.

### Other validation is not shown

The exported workflow does not show validation for email format, missing name, or other customer fields. Add these checks if the workflow will accept untrusted external requests.

## 12. Testing Checklist

After correcting the email-validation wiring, test these scenarios:

- [ ] Send a POST request with no `email`; expect HTTP `400`.
- [ ] Send a POST request with an empty or whitespace-only `email`; expect HTTP `400`.
- [ ] Send a valid email that is not in Supabase; expect a new row and HTTP `200`.
- [ ] Send the same email again; expect HTTP `409`.
- [ ] Send the same email with different letter casing or surrounding spaces; verify that it is treated as the same email.
- [ ] Verify that the inserted name and normalized email are correct in Supabase.
- [ ] Test simultaneous requests to confirm that the database unique constraint prevents duplicate records.

## 13. Possible Improvements

- Add email-format validation and required-field validation for `name`.
- Add a unique constraint on the email column in Supabase.
- Add an error-handling branch for Supabase connection or insert failures.
- Return a consistent response schema for all outcomes.
- Add request authentication or signature verification if the webhook is exposed publicly.
- Add logging for registration attempts and failures without exposing sensitive customer data.
- Consider returning HTTP `201 Created` for successful customer creation.

## 14. Learning Outcomes

This workflow demonstrates how to receive external HTTP requests in n8n, validate incoming data, query a Supabase table, conditionally create records, normalize user input, and return custom JSON responses with HTTP status codes.

It also highlights why execution order, IF-node branch direction, database constraints, and error handling matter when building reliable webhook-driven integrations.
