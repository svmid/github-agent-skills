# Rule: File Upload & Database Transaction

You are an Engineer who understands the fundamental difference between **database transactions** (transactional & rollback-able) and **filesystems** (non-transactional). You NEVER place file upload operations inside `DB::transaction()`.

**Rule: "Transaction boundary ≠ Business flow boundary"**

---

## Design Principles

Database transactions must be:
1. **Fast** — not waiting for external IO
2. **Short** — completed in a brief time
3. **Free of external IO** — no filesystem, network calls, or API calls inside them

**Filesystem ≠ part of a database transaction.**

---

## DO: Upload Files Outside the Transaction

```php
public function update(array $data, int $id): DokumenKegiatan|Error
{
    $model = $this->model->findOrFail($id);

    // 1. Prepare upload (no DB touch yet)
    $upload = $this->processUploadDocument(
        $data,
        'id_dokumen',
        UploadDokumen::KERJASAMA_KEGIATAN,
        $model
    );

    // 2. If there's an upload → perform physical upload FIRST
    if (!empty($upload)) {
        $uploadResult = $upload->executeUpload();

        if (Error::isError($uploadResult)) {
            return $uploadResult;
        }

        $data['id_dokumen'] = $upload->get()?->id;
    } else {
        unset($data['id_dokumen']);
    }

    // 3. Then start DB transaction (short & deterministic)
    DB::beginTransaction();

    try {
        $model->update($data);
        DB::commit();
        return $model;
    } catch (\Throwable $e) {
        DB::rollBack();

        // 4. Cleanup file if DB fails
        if (!empty($upload)) {
            $upload->rollbackUpload();
        }

        return Error::fromException($e);
    }
}
```

**Correct flow:**
1. Upload file to filesystem/object storage
2. If upload succeeds → start DB transaction
3. Save file reference (path/metadata) in DB
4. Commit transaction
5. If DB fails → cleanup the already uploaded file

---

## DON'T: Upload Files Inside a Transaction

```php
// ❌ DO NOT DO THIS
public function update(array $data, int $id): DokumenKegiatan|Error
{
    DB::beginTransaction(); // Transaction started too early

    $model = $this->model->findOrFail($id);
    $upload = $this->processUploadDocument($data, ...);

    if (!empty($upload)) {
        $data['id_dokumen'] = $upload->get()?->id;
    }

    $model->update($data);

    // Upload file INSIDE transaction!
    $upload?->executeUpload();

    DB::commit();
    return $model;
}
```

**Why this is dangerous:**

| Scenario | Impact |
|---|---|
| File uploads successfully, DB rolls back | Orphan file without DB reference |
| Upload is slow (network issue) | DB lock holds resources longer |
| Upload timeout | Transaction timeout, inconsistent data |

**Further impact:**
- Longer database locks → reduced throughput
- Very difficult debugging (cross-layer: DB + filesystem)
- Hard-to-detect data inconsistency

---

## Recommended Rollback Mechanism

Since the filesystem cannot be rolled back automatically, use one of the following mechanisms:

1. **Temporary file** — upload to a temporary location, move to final location after DB commit
2. **Retry / idempotent upload** — upload can be retried without side effects
3. **Manual cleanup** — explicitly delete the file if DB fails (as shown in the DO example above)
