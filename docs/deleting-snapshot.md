# Deleting VDI snapshot data and retaining the snapshot metadata

A VDI snapshot is made up of both data and metadata.
The data is the full image of the disk at the time the snapshot was taken.
The metadata includes the changed block tracking information.

After the snapshot data on the host has been exported to the backup location, you can use the `data_destroy` call to delete only the snapshot data and retain only the snapshot metadata on the host.
This action converts the snapshot that is stored on the host or SR into a smaller metadata-only snapshot.
The `type` field of the snapshot changes to be `cbt_metadata`.

Metadata-only snapshots are linked to the metadata-only snapshots that precede and follow them in time.

You can use the `data_destroy` call only for snapshots for VDIs that have changed block tracking enabled.

> **Note**
>
> The API also provides a `destroy` call, which deletes both the data in the snapshot and the metadata in the snapshot.

Do not use the `destroy` call to delete snapshots that are part of a set of changed block tracking backups unless you are sure that you no longer need the changed block tracking metadata.

For example, use `destroy` to remove a metadata-only snapshot that is older than age allowed by your retention policy.

## Examples

You can use any of our supported language bindings to delete the data in a snapshot and convert the snapshot to a metadata only snapshot.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
session.xenapi.VDI.data_destroy(<snapshot_vdi_ref>)
```

xe command line:

```python
xe vdi-data-destroy uuid=<snapshot_vdi_uuid>
```

## Errors

You might see the following errors when using this call:

**VDI\_MISSING**:

-  The call cannot find the VDI snapshot.

    Check that the reference or UUID you are using to refer to the VDI snapshot is correct. Check that the VDI exists.

**VDI\_NO\_CBT\_METADATA**:

-  No changed block tracking metadata exists for this VDI snapshot.

    Check that changed block tracking is enabled for the VDI.
    You cannot use the `data_destroy` call on VDIs that do not have changed block tracking enabled.
    For more insformation, see [Using changed block tracking with a virtual disk image](./using-with-vdi.md).

**VDI\_IN\_USE**:

-  The VDI snapshot is currently in use by another operation.

    Check that the VDI snapshot is not being accessed by another client or operation.
    Check that the VDI is not attached to a VM.

    If the VDI snapshot is connected to a VM snapshot by a VBD, you receive this error.
    Before you can run `VDI.data_destroy` on this VDI snapshot, you must remove the VM snapshot.
    Use `VM.destroy` to remove the VM snapshot.

## Checking the type of a VDI or VDI snapshot

The value of the `type` VDI field shows what type of VDI or VDI snapshot an object is.
The values this field can have are stored in the `vdi_type` enum.
You can query the value of this field by using the `get_type` call.

A metadata-only VDI snapshot has the type `cbt_metadata`.

### Examples

You can use any of our supported language bindings to query the VDI type of a VDI or VDI snapshot.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
vdi_type = session.xenapi.VDI.get_type(<snapshot_vdi_ref>)
```

xe command line:

```python
xe vdi-param-get param-name=type uuid=<snapshot_vdi_uuid>
```