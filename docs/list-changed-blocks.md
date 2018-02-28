# Getting the list of blocks that changed between VDIs

You can use the `list_changed_blocks` call to get a list of the blocks that have changed between two VDIs.
Both VDI snapshots must be taken after changed block tracking is enabled on the VDI.

This call takes as parameters references to two VDI snapshots:

-  `VDI_from`: The earlier VDI snapshot.

-  `VDI_to`: The later VDI snapshot. This VDI *cannot* be attached to a VM at the time this comparison is made.

This operation does not require the VM associated with the VDIs to be offline at the time the comparison is made.

The changed blocks are listed in a base64-encoded bitmap.
Each bit in the bitmap indicates whether a 64 kB block in the VDI has been changed in comparison to an earlier snapshot.
A bit set to 0 indicates that the block is the same.
A bit set to 1 indicates that the block has changed.

The bit in the first position in the bitmap represents the first block in the VDI.
For example, if the bitmap is 01100000, this indicates that the first block has not changed, the second and third blocks have changed, and all other blocks have not changed.

## Examples

You can use any of our supported languages to get the bitmap that lists the changed blocks between two VDI snapshots.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
bitmap = session.xenapi.VDI.list_changed_blocks(<previous_snapshot_vdi_ref>, <new_snapshot_vdi_ref>)
```

You can convert the base64-encoded bitmap this call returns into a human-readable string of 1s and 0s:

```python
from bitstring import BitStream
import base64
data = BitStream(bytes=base64.b64decode(bitmap))
```

xe command line:

```python
xe vdi-list-changed-blocks vdi-from-uuid=<previous_snapshot_vdi_uuid> vdi-to-uuid=<new_snapshot_vdi_uuid>
```

## Errors

You might see the following errors when using this call:

**VDI\_MISSING**:

-  The call cannot find one or both of the VDI snapshots.

    Check that the reference or UUID you are using to refer to the VDI snapshot is correct.
    Check that the VDI snapshot exists.

**VDI\_IN\_USE**:

-  The VDI snapshot is currently in use by another operation.

    Check that the VDI snapshot is not being accessed by another client or operation.
    Check that the more recent VDI snapshot is not attached to a VM.
    The newer VDI in the comparison cannot be attached to a VM at the time of the comparison.

**Source and target VDI are unrelated**:

-  The VDI snapshots are not linked by changed block metadata.

    You can only list changed blocks between snapshots that are taken as part of the same set of incremental backups.
    For more information, see [Incremental backup sets](./using-with-vdi.md).