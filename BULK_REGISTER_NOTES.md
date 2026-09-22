# Bulk resident registration

Wardens, supervisors and global admins can register many students into a bhawan at once from a CSV file.
The frontend page reads and checks the CSV in the browser, then sends the rows as JSON to one backend endpoint.
The endpoint only registers students who are already on Channeli.
Rows for anyone else are skipped and reported.

## Status

| Part | State |
|---|---|
| Backend endpoint | Done and committed on `feat/Bulk-New-Resident` (`5760a81`, fixed in `5effc44`) |
| Backend tests | 14 tests in `tests.py`, all passing |
| Frontend page | Committed on `feat/Bulk-Create-NewResidetns`, still calling a dummy API (see "Left to do") |
| Manual testing on a running server | Not done yet |

## API

`POST /api/bhawan_app/<hostel>/resident/bulk_register/`

### Request

```json
{
  "dry_run": true,
  "rows": [
    {
      "row_number": 2,
      "enrolment_number": "21114002",
      "hostel_code": "rjb",
      "room_no": "A-101",
      "seat": "A",
      "fee_type": "liv",
      "admission_date": "2026-07-20",
      "mobile_no": "9876543210",
      "address": "12 MG Road, Jaipur",
      "father_name": "Rakesh Patel",
      "father_contact": "9876500000",
      "mother_name": "Sunita Patel",
      "mother_contact": "9876511111"
    }
  ]
}
```

- `enrolment_number`, `hostel_code` and `room_no` are required.
- Every other key is optional, and an empty string counts as not given.
- `dry_run` defaults to true, so only an explicit `false` writes to the database.
- Dates are `YYYY-MM-DD`.
- Keys the endpoint does not use are ignored: `full_name`, `branch_code`, `current_semester`, `email` and `dob`.

### Response

```json
{
  "dry_run": true,
  "summary": { "created": 1, "updated": 0, "existing": 0, "skipped": 1 },
  "rows": [
    {
      "row_number": 2,
      "enrolment_number": "21114002",
      "action": "would_create",
      "status": "dry_run",
      "message": ""
    },
    {
      "row_number": 3,
      "enrolment_number": "26114001",
      "action": "skipped",
      "status": "error",
      "message": "Enrollment No 26114001 is not a student on Channeli"
    }
  ]
}
```

The keys stay in snake case, unlike the rest of the Omniport API, because the frontend page reads them that way.

### What happens to each row

| Situation | Result | `action` (real run / dry run) |
|---|---|---|
| Channeli student with no residency in this bhawan | A new residency is created | `created` / `would_create` |
| Channeli student with an active residency in another bhawan | The old residency is ended and a new one is created, message `Moved from rkb` | `created` / `would_create` |
| Channeli student already living in this bhawan, with changes | The residency is updated in place | `updated` / `would_update` |
| Channeli student already living in this bhawan, nothing changed | Nothing is written | `existing` |
| Enrolment number is not a Channeli student | Row skipped, nothing is created | `skipped`, status `error` |
| Invalid value (fee type, date, room longer than 10 characters, and so on) | Row skipped, the message names the field | `skipped`, status `error` |
| Enrolment number repeated in the same upload | Row skipped | `skipped`, status `error` |
| Bhawan the uploader does not manage | Row skipped, `You cannot register residents in rkb` | `skipped`, status `error` |

### Errors for the whole request

- `403` with `detail` when the user is not a warden or supervisor of the bhawan in the URL, or a global admin.
- `400` with `detail` when `rows` is missing, empty, or not a list of objects.

## Behaviour and decisions

- **Only Channeli students.** The endpoint never creates Users, Persons or Students. Students who are not on Channeli yet still go through the registration service or the admin import script.
- **Channeli records are never changed.** Name, branch, semester and contact details of a student stay as they are. The upload only writes the `Resident` row and the parents' `Person` rows.
- **Empty values keep what is stored.** Re-uploading a file with an empty Fee Type does not reset a NOT LIVING student to LIVING, and uploading the same file twice reports every row as `existing`.
- **Room and seat are joined** as `room-seat`, for example `A-101-A`, the same way the admin import script does it. The result must fit in 10 characters.
- **A renamed parent gets a new Person**, because ended residencies of the same student may still point at the old one.
- **Permissions.** The URL's bhawan is checked for every request. Each row's bhawan is checked again, so a supervisor of rjb cannot register students into rkb. Global admins can register into any bhawan.
- **Dry run is the real run rolled back.** Both use the same code inside one transaction, and the dry run rolls it back at the end. The preview therefore always matches what Confirm will do.
- **Rows fail on their own.** Each row runs in its own savepoint, so a bad row is skipped and the rest are still saved. An unexpected server error returns 500 and rolls back the whole upload, so a half-finished import cannot happen.

## Files changed in the backend

| File | Change |
|---|---|
| `views/resident.py` | New `bulk_register` action on `ResidentViewset`. `initial()` now uses `can_manage_residents()`. |
| `managers/bulk_register.py` | New. Row processing, dry-run rollback and the report. |
| `serializers/bulk_register.py` | New. Validates one row, reusing the kernel's enrolment number rule. |
| `managers/services.py` | New `can_manage_residents()`: warden or supervisor of the bhawan, or a global admin. |
| `tests.py` | New. 14 tests that call the real URL as a logged-in user. |

## Running the tests

From the `omniport` folder inside the Django container:

```
python manage.py test bhawan_app
```

The tests cover the dry run, the missing `dry_run` flag, non-Channeli students, every stored field, defaults for empty values, moving between bhawans, updating in place, stray residencies, bad rows, repeated enrolment numbers, bhawan permissions, global admins, non-admins and malformed requests.
Each branch of the endpoint was broken on purpose once, and a test failed every time.

## Left to do

### Frontend (`omniport-frontend/omniport/apps/bhawan_app`)

- [ ] Remove the dummy API in `src/actions/bulk-register.js`: delete `dummyBulkRegisterAdapter` and the `adapter:` line.
- [ ] Update the "Backend API" section of the frontend `README.md`: the endpoint exists now, and non-Channeli students come back as skipped rows.
- [ ] Decide what Confirm does when rows are skipped. Today `previewed` in `src/components/bulk_register/index.js` requires `summary.skipped === 0`, so an admin must delete non-Channeli rows from the file before confirming. If admins should be able to confirm and let those rows be skipped, loosen that check.
- [ ] Drop the columns the backend ignores, or mark them as unused: Name is no longer required, and Branch Code, Current Semester, Email and Date of Birth are not stored.

### Manual testing on a running server

- [ ] Run the backend tests on the laptop.
- [ ] Log in as a supervisor of one bhawan, open Bulk Register Students, upload a CSV with a mix of Channeli students, a non-Channeli enrolment number and a bad row, and check the preview.
- [ ] Confirm, then check the Student Database page shows the new residents with the right rooms.
- [ ] Upload the same file again and check every row says No change.
- [ ] Try a row for another bhawan as a supervisor, and again as a global admin.

### Backend follow-ups

- [ ] `Resident.start_date` defaults to `datetime.now`, which gives a timezone-naive value and a Django warning whenever a residency is created without a start date. Change it to `timezone.now` with a migration. The bulk endpoint already passes an aware time, so it is not affected.
- [ ] Very large uploads run inside one request. A few hundred rows is fine, but a file with thousands of rows could hit the server timeout. Add a row limit or a background job if that becomes a real use case.
