---
name: healthcare-phi-compliance
description: Protect health data and other private data in health apps. Use for HIPAA, GDPR, DISHA, access rules, safe logs, audit records, APIs, data stores, and security reviews.
origin: Health1 Super Speciality Hospitals, contributed by Dr. Keyur Patel
version: "1.0.0"
---

# Healthcare PHI and PII Safety

This skill was contributed by **Health1 Super Speciality Hospitals and Dr. Keyur Patel**.

Use these rules to protect patient, staff, and payment data. They help with HIPAA in the US, GDPR in the EU, DISHA in India, and similar laws.

These rules are a safe base. They are not legal advice. Check the law, contracts, and health rules for the place where the app is used.

## When to Use

Use this skill when you:

- Work with patient or staff records
- Build sign-in or access rules
- Design a health database
- Build an API that returns private data
- Add logs or audit records
- Review code for data leaks
- Add row-level security
- Import, export, print, share, or delete health data
- Use backups, test data, support tools, or outside services

## Core Rules

Use three layers:

1. **Classify:** Know which data is private.
2. **Control:** Let only the right person use it.
3. **Audit:** Record who used it and why.

Also follow these rules:

- Collect only the data you need.
- Show only the data needed for the task.
- Deny access unless a rule allows it.
- Check access on the server for every request.
- Encrypt private data while it moves and while it is stored.
- Keep test, staging, and live data apart.
- Do not use real patient data in tests.
- Set a clear time to keep and delete data.
- Have a plan for leaks and lost devices.

## Data Types

### PHI

PHI is health data that can point to a person. It may include:

- Name, date of birth, address, phone, or email
- Face, voice, photo, or other body data
- SSN, Aadhaar, NHS number, or another national ID
- Medical record, insurance, or claim number
- Diagnosis, medicine, lab result, scan, or care note
- Visit, booking, admission, or discharge data
- A group of facts that can point to one person

A record may still be PHI even if the name is gone. Rare facts, dates, or places may point to the person.

### Other PII

Health apps also hold private data about staff and other people:

- Staff contact details
- Doctor fees and payments
- Pay, bank, and tax data
- Vendor payment data
- Sign-in data and device data

Treat this data with the same care.

## Work Steps

For each new feature:

1. List every private field it reads, writes, sends, prints, or stores.
2. State why each field is needed.
3. Remove fields that are not needed.
4. List each role that may use the data.
5. Add server-side access rules.
6. Add audit events for access and change.
7. Check logs, errors, URLs, caches, files, and backups.
8. Test allowed and blocked users.
9. Test users from another site or tenant.
10. Write down any risk that is still open.

Stop and ask for a privacy or legal review when the right rule is not clear.

## Access Control

Use the least access needed. Check the user, role, site, tenant, patient link, and task.

Do not trust a role or site ID sent by the client. Read it from the signed-in user and trusted server data.

```sql
ALTER TABLE patients ENABLE ROW LEVEL SECURITY;
ALTER TABLE patients FORCE ROW LEVEL SECURITY;

CREATE POLICY "staff_read_own_facility"
  ON patients
  FOR SELECT
  TO authenticated
  USING (
    facility_id IN (
      SELECT facility_id
      FROM staff_assignments
      WHERE user_id = auth.uid()
        AND role IN ('doctor', 'nurse', 'lab_tech', 'admin')
    )
  );
```

Add separate rules for `SELECT`, `INSERT`, `UPDATE`, and `DELETE`. A read rule does not make writes safe.

Watch for database owners, admin roles, and service keys that can skip row rules. Keep these powers on trusted servers only.

### Urgent Access

Some health systems need urgent access, also called break-glass access.

If it is allowed:

- Ask the user for a reason.
- Limit the access time.
- Log the event at once.
- Alert the right review team.
- Review the event after use.

Do not add urgent access unless the owner and legal team approve it.

## Audit Records

Log access and change when the law or policy calls for it. Include reads, prints, exports, shares, and urgent access.

```typescript
interface AuditEntry {
  timestamp: string;
  user_id: string;
  patient_record_id: string;
  action:
    | 'create'
    | 'read'
    | 'update'
    | 'delete'
    | 'print'
    | 'export'
    | 'share'
    | 'break_glass';
  resource_type: string;
  resource_id: string;
  reason?: string;
  changed_fields?: string[];
  ip_address?: string;
  session_id: string;
}
```

Do not copy full patient values into the audit log. For a change, store field names when that is enough. If old and new values must be kept, encrypt them and set strict access rules.

Audit logs should be append-only for app users:

```sql
ALTER TABLE audit_log ENABLE ROW LEVEL SECURITY;

CREATE POLICY "audit_insert_own_event"
  ON audit_log
  FOR INSERT
  TO authenticated
  WITH CHECK (user_id = auth.uid());

CREATE POLICY "audit_no_update"
  ON audit_log
  FOR UPDATE
  TO authenticated
  USING (false);

CREATE POLICY "audit_no_delete"
  ON audit_log
  FOR DELETE
  TO authenticated
  USING (false);
```

These rules do not stop a database owner from changing logs. Use backups, alerts, and a locked audit store when stronger proof is needed.

Limit who can read audit logs. The logs are private data too.

## Common Leak Paths

### Errors

Do not send names, health facts, IDs, or raw database errors to the client.

```typescript
logger.error('Record lookup failed', {
  recordId: patient.id,
  requestId,
});

throw new Error('Record not found');
```

Use an opaque random ID. Do not use a medical record number as the log ID.

### Logs

Do not log:

- Full patient or staff objects
- Request or response bodies with private data
- Tokens, passwords, cookies, or secret keys
- Search text that may hold a name or diagnosis

Protect log access. Set a short keep time. Remove private data before sending errors to an outside service.

### URLs

Do not put names, diagnoses, national IDs, or record numbers in URLs. URLs may enter browser history, proxy logs, and chat messages.

Use an opaque random ID when an ID is needed.

### Browser Storage and Cache

Do not store PHI in `localStorage` or `sessionStorage`.

Keep it in memory when possible. Clear it on sign-out and timeout. Use safe cache headers on private pages:

```http
Cache-Control: no-store
Pragma: no-cache
```

Do not let a service worker cache private pages or API replies.

### Files and Downloads

Exports and print files need access checks too.

- Use short-lived links.
- Do not make files public.
- Add a safe file name.
- Set the right content type.
- Delete temp files.
- Log print and export events.
- Hide private data in previews when it is not needed.

### Messages and Outside Services

Do not send PHI by email, text, chat, or push notice unless the use is approved and safe.

Do not send PHI to an outside service until the owner has checked the contract, location, access, and security needs.

### Keys

Never place admin or service keys in browser or mobile app code. Keep them on trusted servers.

Do not commit keys to source control. Change a key at once if it may have leaked.

## Schema Labels

Mark private columns so reviews and tools can find them:

```sql
COMMENT ON COLUMN patients.name IS 'PHI: patient_name';
COMMENT ON COLUMN patients.dob IS 'PHI: date_of_birth';
COMMENT ON COLUMN patients.aadhaar IS 'PHI: national_id';
COMMENT ON COLUMN doctor_payouts.amount IS 'PII: financial';
```

Labels do not protect data by themselves. Access rules and encryption are still needed.

## Edge Cases

Check these cases:

- A worker changes site, team, or role
- A user account is locked or removed
- One person has more than one role
- A patient asks for a copy or fix
- A child becomes an adult
- A guardian loses access
- Two patient records are merged
- Data is restored from backup
- A task runs with no signed-in user
- A support worker views live data
- A link is shared with the wrong person
- A phone or laptop is lost
- A failed job retries the same export
- A report has so few rows that one person is easy to guess

## Concrete Example

Request: “Add an API that lists today’s lab results for a doctor.”

Use this plan:

1. Return only the fields shown on the page.
2. Read the doctor ID from the signed-in session.
3. Find the doctor’s current site from trusted server data.
4. Filter results by that site.
5. Check that the doctor may view each patient.
6. Do not accept a trusted site or doctor ID from the URL.
7. Record one audit event for each patient record viewed.
8. Return a plain error with a request ID.
9. Set `Cache-Control: no-store`.
10. Test these cases:

```text
Allowed doctor, same site: results are returned
Allowed doctor, other site: no results are returned
Removed doctor: access is denied
Missing session: access is denied
Changed site ID in request: access is still denied
Server error: no PHI appears in the reply or logs
```

## Release Check

Before release, confirm:

- No PHI appears in client errors
- No PHI appears in normal logs or stack traces
- No PHI appears in URLs
- No PHI is saved in browser storage
- Private pages and replies are not cached
- No admin or service key is in client code
- All private tables have access rules
- Read and write rules were tested on their own
- Site and tenant walls were tested
- Audit events cover required actions
- Audit logs have strict read and write rules
- Sessions expire and sign-out clears private data
- Every private API checks the user on the server
- Exports, prints, backups, and temp files are safe
- Test data has no real patient data
- Old data has a safe delete plan
- The leak response plan has an owner and contact path