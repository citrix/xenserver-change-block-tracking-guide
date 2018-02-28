# Using changed block tracking with a virtual disk image

The changed block tracking capability can be enabled and disabled for individual virtual disk images (VDIs).

## Incremental backup sets

When you enable changed block tracking for a VDI you start a new set of incremental backups for that VDI.
The first action you must take when starting a set of incremental backups is to create a baseline snapshot and to backup its full data.

After you disable changed block tracking, or after changed block tracking is disabled by XenServer or a user, no further incremental backups can be added to this set.
If changed block tracking is enabled again, you must take another baseline snapshot and start a new set of incremental backups.

You cannot compare VDI snapshots taken as part of one set of incremental backups with VDI snapshots taken as part of a different set of incremental backups.
If you attempt to list the changed blocks between snapshots that are part of different sets, you get an error with the message `Source and target VDI are unrelated`.

You can use some or all of the data in previous incremental backup sets to create VDIs that you can use to restore the state of a VDI.
For more information, see [Coalescing changed blocks onto a base VDI](./coalescing-block.md).

## Enabling changed block tracking for a VDI

By default, changed block tracking is not enabled for a VDI.
You can enable changed block tracking by using the `enable_cbt` call.

When changed block tracking is enabled for a VDI, additional log files are created on the SR to list the changes since the last backup.
Blocks of 64 kB within the VDI are tracked and changes to these blocks recorded in the log layer.

The associated VM remains in the same state as before changed block tracking was enabled.

To enable changed block tracking, an SR must be attached, be writable, and have enough free space for the log files to be created on it.
The associated VM can be in any state when changed block tracking is enabled or disabled.
It is not required that the VM be offline.

Changed block tracking can only be enabled for a VDI that is one of the following types:

-  user

-  system

In addition, if the `VDI.on_boot` field is set to `reset`, you cannot enable changed block tracking for the VDI.

### Examples

You can use any of our supported language bindings to enable changed block tracking for a VDI.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
session.xenapi.VDI.enable_cbt(<vdi_ref>)
```

xe command line:

```python
xe vdi-enable-cbt uuid=<vdi-uuid>
```

### Errors

You might see the following errors when using this call:

**VDI\_MISSING**:

-  The call cannot find the VDI.

    Check that the reference or UUID you are using to refer to the VDI is correct. Check that the VDI exists.

**VDI\_INCOMPATIBLE\_TYPE**:

-  The VDI is of a type that does not support changed block tracking.

    Check that the type of the VDI is `system` or `user`.
    You can use the `get_type` call to find out the type of a VDI.
    If your VDI is an incompatible type, you cannot enable changed block tracking.
    For more information, see [Checking the type of a VDI or VDI snapshot](./deleting-snapshots.md).

**VDI\_ON\_BOOT\_MODE\_INCOMPATIBLE\_WITH\_OPERATION**:

-  The value of the `on_boot` field of the VDI is set to `reset`.

    Check the value of the `on_boot` field by using the `get_on_boot` call.
    If appropriate, you can use the `set_on_boot` call to change the value of this field to `persist`.

**SR\_NOT\_ATTACHED, SR\_HAS\_NO\_PBDS**:

-  The call cannot find an attached SR.

    Check that there is an SR attached to the host and that the SR is writable.
    You cannot enable changed block tracking unless the host has access to an SR to which the changed block information can be written.

If you attempt to enable changed block tracking for a VDI that already has changed block tracking enabled, no error is thrown.

## Disabling changed block tracking for a VDI

You can disable changed block tracking for a VDI by using the `disable_cbt` call.

When changed block tracking is disabled for a VDI, the active disks are detached and reattached without the log layer.
The associated VM remains in the same state as before changed block tracking was disabled.

### Examples

You can use any of our supported language bindings to disable changed block tracking for a VDI.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
session.xenapi.VDI.disable_cbt(<vdi_ref>)
```

xe command line:

```python
xe vdi-disable-cbt uuid=<vdi-uuid>
```

### Errors

You might see the same sorts of errors for this call and you might for the `enable_cbt` call.

If you attempt to disable changed block tracking for a VDI that already has changed block tracking disabled, no error is thrown.

## Checking whether changed block tracking is enabled

The value of the boolean `cbt_enabled` VDI field shows whether changed block tracking is enabled for that VDI.
You can query the value of this field by using the `get_cbt_enabled` call.

A return value of `true` indicated that changed block tracking is enabled for this VDI.

### Examples

You can use any of our supported languages to check whether a VDI has changed block tracking enabled.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
is_cbt_enabled = session.xenapi.VDI.get_cbt_enabled(<vdi_ref>)
```

xe command line:

```python
xe vdi-param-get param-name=cbt-enabled uuid=<vdi-uuid>
```