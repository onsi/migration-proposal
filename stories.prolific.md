The auction should only send resources and identifiers back and forth

Currently we send the entire DesiredLRP/Task payload which gives us yet-another-thing to version.

Instead we should only send the bare minimum required for the rep to make a reservation and the auctioneer to decide how to allocate resources.

Acceptance:

- The /state and /work endpoints on the rep do not return/require Task/DesiredLRP payloads
- The auction request endpoints on the auctioneer do not require Task/DesiredLRP payloads

L: versioning, diego:ga

---

The Task callback URL should only receive a Task identifier

This means we don't have to worry about versioning tasks back to the callback URL.  Instead, the recipient can then reach out, fetch the task, and mark it as completed.

We'll need to update the stager to work with this.  We should also notify lattice, etc. of the change.

Acceptance:

- My registered Task completion callback only receives the TaskID
- I can then fetch the TaskID (it remains in the COMPLETED state)
- I am responsible for subsequently deleting the TaskID
- If I do not remove my COMPLETED task I should expect to receive another notification (eventually: this is existing behavior)
- If I never remove my COMPLETED task the converger should garbage collect it (eventually: this is existing behavior)

In addition:
- Staging should still work
- We should not leak COMPLETED tasks (i.e. the stager should remove that Task)

L: versioning, diego:ga

---

All CC-Bridge communication should happen directly with the BBS

Acceptance:
- I can stop the receptor...
- ...and `cf push` still works

L: versioning, diego:ga

---

All Route-Emitter communication should happen directly with the BBS

Acceptance:
- I can stop the receptor...
- ...and still route to my apps.

L: versioning, diego:ga

---

Remove the Receptor

Acceptance:
- The access VM should not have a Receptor!

L: versioning, diego:ga

---

All access to the BBS should go through one, master-elected, BBS server

We set this up to drive out the rest of the migration work.

Acceptance: 
- I can only direct read/write requests to one BBS server

L: versioning, diego:ga

---

After a BOSH deploy, all Task data in the BBS should be stored in base64 encoded protobuf format

This drives out (see https://github.com/onsi/migration-proposal#the-bbs-migration-mechanism):

- introducing a database version
- teaching the BBS server to run the migration on start
- teaching the BBS server to bump the database version upon a succesful migration

Acceptance:
- After completing a BOSH deploy I see Task data in the BBS stored in base64 encoded protobuf format
- I can (manually) fetch a database version key from etcd

L: versioning, diego:ga

---

If the BBS is killed mid-migration, bringing the BBS back completes the migration

I propose using this to drive out adding encoding/versioning information to each database record.  (We've already done this but I might have ordered the stories this way instead)

This also drives out managing the database version in the case of a failed migration.

Acceptance:
- while true; do
    - I fill etcd with many JSON-encoded tasks
    - I perform a BOSH deploy
    - I kill the BBS mid-migration
    - I bring the BBS back
    - I see protobuf-encoded tasks, only

L: versioning, diego:ga

---

During a migration, the BBS should return 503 for all requests

All clients should forward the 503 error along where appropriate.

Acceptance:
- I set up a long-running migration (perhaps I have a lot of data)
- I see requests to the BBS return 503 during the migration

L: versioning, diego:ga

---

If a migration fails, the BOSH deploy should abort and no Cells should roll

This may prove difficult with BOSH.  Perhaps:

- If the BBS fails to get the lock: it should mark itself as running.
- If the BBS grabs the lock: it only marks itself as running after the migration completes.

?

Acceptance:
- I see the BBS roll first (before Cells roll)
- I can insert malformed data into the database.  This causes the migration to fail.  If this happens, BOSH should stop the deploy.  My apps should continue to run and be routable.  I have no access to the API, however.

L: versioning, diego:ga

---

All LRP data should be stored in base64 encoded protobuf format

This should bump the DB version.  All LRP data should get migrated via a BOSH deploy.

This allows us to practice another migration.

Acceptance:
- After a BOSH deploy I see all LRPs stored in base64 encoded protobuf format

L: versioning, diego:ga

---

As a Diego operator, I would like to specify a set of decryption keys to use to decrypt data at rest, with one encryption key to be used when writing data

The encryption should occur during the migration phase via a BOSH deploy.  We can accomplish this by including the encryption key name in the DB version key.  To ensure success in the face of a mid-migration failure, we should also append the key-name to each record.

Proposed data payload: 0007key-name:base64-encoded-data

Proposed version key:
```
{
    CurrentVersion int
    TargetVersion int

    CurrentEncryptionKeyName string
    TargetEncryptionKeyName string
}
```

(We should flesh out the state machine for managing `CurrentEncryptionKeyName` and `TargetEncryptionKeyName`)

Acceptance:
- After a BOSH deploy all my data is encrypted
- I can change my encryption key.  The BOSH deploy should succeed.

L: versioning, security, diego:ga

---

DesiredLRP data should be split across separate records

This should bump the DB version.  All LRP data should get migrated via a BOSH deploy.

We do not modify the API, but - instead - do whatever work is necessary under the hood to maintain the existing API.

Acceptance:
- After a BOSH deploy I can see that all LRP data has been split in two.

L: perf, versioning, diego:ga

---

As a BBS client, I can efficiently get frequently accessed data for all DesiredLRPs in a domain

Add a new API endpoint

Acceptance:
- NSYNC bulker and Route-Emitter should operate more efficiently because they only require a (small) subset of data.

L: perf, versioning, diego:ga

---

Cut Diego 1.0.0

We draw a line in the sand.  All subsequent work should ensure a clean upgrade path from 1.0.0.

We call this 1.0.0 not to signify that all our GA goals have been met (performance is still somewhat lacking).  However this gives us a reference point to be disciplined about maintaining backward compatibility during subsequent deploys.

L: versioning, diego:ga

---

[CHORE] Diego has an integration suite/environment that validates that upgrades from 1.0.0 to HEAD satisfy the minimal-downtime requirement

This ensure that subsequent work migrates cleanly.

L: versioning