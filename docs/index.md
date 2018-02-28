# Citrix XenServer Changed Block Tracking Guide

Changed block tracking provides a set of features and APIs that enable you to develop fast and space-efficient incremental backup solutions for XenServer.

Changed block tracking is available only to customers with XenServer Enterprise Edition.
If a customer without Enterprise Edition attempts to use an incremental backup solution for XenServer that uses changed block tracking, they are prevented from enabling changed block tracking on new VDIs.
However, if the customer has existing VDIs with changed block tracking enabled, they can still perform other changed block tracking actions on these VDIs.

## How changed block tracking works

When changed block tracking is enabled for a virtual disk image (VDI), any blocks that are changed in that VDI are recorded in a log file.
Every time the VDI is snapshotted, this log file can be used to identify the blocks that have changed since the VDI was last snapshotted.
This provides the capability to backup only those blocks that have changed.

After the changed blocks have been exported, the full VDI snapshots can now be changed into metadata-only snapshots by destroying the data associated with them and leaving only the changed block information.
These metadata only snapshots are linked both to the preceding metadata-only snapshot and to the following metadata-only snapshot.
This provides a chain of metadata that records the full history of changes to this VDI since changed block tracking was enabled.

The changed block tracking feature also takes advantage of network block device (NBD) capabilities to perform the export of data from the changed blocks.

## Benefits of changed block tracking

Unlike some other incremental backup solutions, changed block tracking does not require that the customer keep a snapshot of the last known good state of a VDI available on the host or a storage repository (SR) to compare the current state to.
The customer needs less disk space because, instead of handling and storing large VDIs, with changed block tracking they instead can choose to store space-efficient metadata-only snapshot files.

Changed block tracking also saves the customer time as well as space.
Other backup solutions export a snapshot of the whole VDI every single time the VDI is backed up.
This is a time-consuming process and the customer has to pay that time cost every time they take a backup.
With changed block tracking enabled, the first back up exports a snapshot of the whole VDI.
However, subsequent backups only export the blocks in the VDI that have changed since the previous backup.
This decreases the time required to export the backup in proportion to the percentage of blocks that have changed.

For example, it can take around 10 hours to export a backup of a full 1 TB VDI.
If, after a week, 5% of the blocks in that VDI have changed, exporting the backup takes 5% of the time - 30 minutes.
A backup taken after a day has even fewer changed blocks and takes even less time to export.

The savings in time and space that changed block tracking provides makes it a preferable backup solution for customers using XenServer.
The simple API that XenServer exposes makes it easy for you to develop an incremental backup solution that delivers these benefits to the end user.
You can use this API through the language-agnostic remote procedure calls (RPCs) or take advantage of the language bindings provided for C, C\#, Java, Python and PowerShell.