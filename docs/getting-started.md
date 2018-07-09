# Getting started using changed block tracking

This section steps through the process of using changed block tracking to create incremental backups.

Before getting started with changed block tracking, we recommend that you read the Citrix XenServer Software Developer Kit Guide.
This document contains information to help you become familiar with developing for XenServer.

The examples provided in these steps use the Python binding for the Management API.

-  For more information about the individual RPC calls, see the [Citrix XenServer Management API](https://developer-docs.citrix.com)

-  For more detailed information about individual steps in this process, see the following chapters.

Full Python examples are provided on [GitHub](https://github.com/xenserver/xs-cbt-samples).

The NBD connection examples provided in these steps use the Linux nbd-client.
However, you can use any NBD client that supports the "fixed newstyle" version of the NBD protocol.
For more information, see [the NBD protocol documentation](https://sourceforge.net/p/nbd/code/ci/master/tree/doc/proto.md).

> **Note**
>
> If you are using the Linux upstream NBD client, a minimum version of 3.15 is required to support TLS.

## Prerequisites

Before you start, set up or implement an NBD client at the backup location that supports the "fixed newstyle" version of the NBD protocol.
For more information, see [Exporting the changed blocks using an NBD client](./exporting-changed-blocks.md).

Enable NBD connections on your network.
For more information, see [Enabling NBD connections on XenServer](./enabling_nbd.md).

## Procedure

This procedure is broken down into three sections:

-  **Setting up changed block tracking**
    Perform the steps in this section once, when you start using changed
    block tracking, to enable the changed block tracking capability and
    export a base snapshot that the incremental, changed block exported
    data is compared to.

-  **Taking incremental backups**
    Perform the steps in this section every time you want to take an
    incremental back up of the changed blocks in a VDI.

-  **Restoring a VDI from exported changed block data**
    Perform the steps in this section if you want to use your backed up
    data to restore a VDI to an earlier state.

### Setting up changed block tracking

Before you can take incremental backups of a VDI using changed block tracking, you must first enable changed block tracking on the VDI and export a base snapshot.
To set up changed block tracking for a VDI, complete the following steps

1.  Use the Management API to establish a XenAPI session on the
    XenServer host:

        import XenAPI
        import shutil
        import urllib3
        import requests

        session = XenAPI.xapi_local()
        session.xenapi.login_with_password("<user>", "<password>", "<version>", "<originator>")

1.  **Optional**: If you intend to create a new VM and new VDIs to restore your backed up data to, you must also export your VM metadata.
    Ensure that you export a copy of the VM metadata any time your VM properties change.
    This can be done by using HTTPS or by using the command line.

        session_id = session._session
        url = ("https://%s/export_metadata?session_id=%s&uuid=%s"
                "&export_snapshots=false"
                % (<xs_host>, session_id, <vm_uuid>))

        with requests.Session() as session:
            urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
            request = session.get(url, verify=False, stream=True)
            with open(<export_path>, 'wb') as filehandle:
                shutil.copyfileobj(request.raw, filehandle)
            request.raise_for_status()

    Where &lt;*export\_path*&gt; is the location to save the VM metadata to.

    The export URL includes the parameter `export_snapshots=false`.
    This parameter ensures that the snapshot history is not included in the VM metadata backup.
    The VM metadata is used to create a new VM and this snapshot history does not apply to the new VM.

    If you intend to use your backed up data only to restore existing VDIs and VMs, you can skip this step.

1.  Get a reference for the VDI you want to snapshot:

        vdi_ref = session.xenapi.VDI.get_by_uuid("<vdi_uuid>")

1.  Enable changed block tracking for the VDI:

        session.xenapi.VDI.enable_cbt(<vdi_ref>)

    For more information, see [Using changed block tracking with a virtual disk image](./using-with-vdi.md).

1.  Take a snapshot of the VDI:

        base_snapshot_vdi_ref = session.xenapi.VDI.snapshot(<vdi_ref>)

    This VDI snapshot is the base snapshot.

1.  Export the base VDI snapshot to the backup location. This can be done by using HTTPS or by using the command line.

    For example, at the xe command line run:

        xe vdi-export uuid=<base-snapshot-vdi-uuid> filename=<name of export>

    Or, in Python, you can use the following code:

        session_id = session._session
        url = ('https://%s/export_raw_vdi?session_id=%s&vdi=%s&format=raw'
               % (<xs_host>, session_id, <base_snapshot_vdi_uuid>))
        with requests.Session() as http_session:
            urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
            request = http_session.get(url, verify=False, stream=True)
            with open(<export_path>, 'wb') as filehandle:
                shutil.copyfileobj(request.raw, filehandle)
            request.raise_for_status()

    Where &lt;*export\_path*&gt; is the location you want to write the exported VDI to.

1.  Optional: For each VDI snapshot, delete the snapshot data, but retain the metadata:

        session.xenapi.VDI.data_destroy(<base_snapshot_vdi_ref>)

    This frees up space on the host or SR.

    For more information, see [Deleting VDI snapshot data and retaining the snapshot metadata](./deleting_snapshot.md).

### Taking incremental backups

After taking the initial VDI snapshot and exporting all the data, the following steps can be repeated every time an incremental backup is taken of the VDI.
These incremental backups export only the blocks that have changed since the previous snapshot was taken.

To take an incremental backup, complete the following steps:

1.  Check that changed block tracking is enabled:

        is_cbt_enabled = session.xenapi.VDI.get_cbt_enabled(<vdi_ref>)

    If the value of `is_cbt_enabled` is not `true`, you must complete the steps in the *Setting up changed block tracking* section, before taking incremental backups.
    For more information, see [Incremental backup sets](./using-with-vdi.md).

    If changed block tracking is disabled and this is unexpected, this state might indicate that the host or SR has crashed since you last took an incremental backup or that a XenServer user has disabled changed block tracking.

1.  Take a snapshot of the VDI:

        snapshot_vdi_ref = session.xenapi.VDI.snapshot(<vdi_ref>)

1.  Compare this snapshot to a previous snapshot to find the changed blocks:

        bitmap = session.xenapi.VDI.list_changed_blocks(<base_snapshot_vdi_ref>, <snapshot_vdi_ref>)

    This call returns a base64-encoded bitmap that indicates which blocks have changed.
    For more information, see [Get the list of blocks that changed between VDIs](./list-changed-blocks.md).

1.  Get details for a list of connections that can be used to use to access the VDI snapshot over the NBD protocol.

        connections = session.xenapi.VDI.get_nbd_info(<snapshot_vdi_ref>)

    This call returns a list of connection details that are specific to this session.
    Each set of connection details in the list contains a dictionary of the parameters required for an NBD client connection.
    For more information, see [Getting NBD connection information for a VDI](./export-changed-blocks.md).

    > **Note**
    >
    > Ensure that this session with the host remains logged in until after you have finished reading from the network block device.

1.  From your NBD client, complete the following steps to export the changed blocks to the backup location.
    For example, when using the Linux `nbd-client`:

    1.  Connect to the NBD server.

            nbd-client <address> <port> -N <exportname> -cacertfile <cacert> -tlshostname <subject>

        -  The &lt;*address*&gt;, &lt;*port*&gt;, &lt;*exportname*&gt;, and &lt;*subject*&gt; values passed as parameters to the connection command are the values returned by the `get_nbd_info` call.

        -  The &lt;*cacert*&gt; is a file containing one or more trusted Certificate Authority certificates of which at least one has signed the NBD server's TLS certificate.
           That TLS certificate is included in the values returned by the `get_nbd_info` call.
           If the TLS certificate returned by the `get_nbd_info` call is self-signed, it can be used as the value of `cacert` here to authenticate itself.

           For more information about using these values, see [Getting NBD connection information for a VDI](./exporting-changed-blocks.md).

    1.  Read off the blocks that are marked as changed in the bitmap returned from step 3.

    1.  Disconnect from the block device:

            nbd-client -d <block_device>

    1.  Optional: We recommend that you retain the bitmap associated with each changed block export at your backup location.

    To complete the preceding steps, you can use any NBD client implementation that supports the “fixed newstyle” version of the NBD protocol.
    For more information, see [Exporting the changed blocks using an NBD client](./exporting-changed-blocks.md).

1.  Optional: On the host, delete the VDI snapshot, but retain the metadata:

        session.xenapi.vdi.data_destroy(<snapshot_vdi_ref>)

    This frees up space on the host or SR.

    For more information, see [Deleting VDI snapshot data and retaining the snapshot metadata](./deleting-snapshos.md).

### Restoring a VDI from exported changed block data

When you want to use your incremental backups to restore or import data from a VDI, you cannot use individual exports of changed blocks to do this.
You must first coalesce the exported changed blocks onto a base snapshot.
Use this coalesced VDI to restore or import backed up data.

1.  Create a coalesced VDI.

    For each set of exported changed blocks between the base snapshot and the snapshot you want to restore to, create a coalesced VDI from a previous base VDI and the subsequent set of exported changed blocks.
    Ensure that you apply sets of the changed blocks to the base VDI in the order that they were snapshotted.

    To create a coalesced VDI from a base VDI and the subsequent set of exported changed blocks, complete the following steps:

    1.  Get the bitmap that was used in step 3 to derive this set of exported changed blocks.

    1.  For each block in the VDI:

        -  If the bitmap indicates that the block has changed, read the block data from the set of exported changed blocks and append that data to the coalesced VDI.

        -  If the bitmap indicates that the block has not changed, read the block data from the base VDI and append that data to the coalesced VDI.

    1.  Use the coalesced VDI as the base VDI for the next iteration of this step.
        Or, if you have reached the target snapshot level, use this coalesced VDI in the next step to restore a VDI in XenServer.

    For more information, see [Coalescing changed blocks onto a base VDI](./coalescing-blocks.md).

    You can now use this coalesced VDI to either import backed up data into a new VDI or to restore an existing VDI.

1.  Optional: Create a new VM and new VDI.

    You can create a new VM and new VDI to import the coalesced VDI into.
    However, if you intend to use the coalesced VDI to restore an existing VDI, you can skip this step.

    To create a new VM and new VDI, complete the following steps

    1.  Create a new VDI:

            vdi_record = {
                 "SR": <sr>,
                 "virtual_size": <size>,
                 "type": "user",
                 "sharable": False,
                 "read_only": False,
                 "other_config": {},
                 "name_label": "<name_label>"
            }
            vdi_ref = session.xenapi.VDI.create(vdi_record)
            vdi_uuid = session.xenapi.VDI.get_uuid(vdi_ref)

        Where &lt;*sr*&gt; is a reference to the SR that the original VDI was located on and &lt;*size*&gt; is the size of the original VDI.

    1.  To create a new VM that uses the VDI created in the previous step, import the VM metadata associated with the snapshot level you are using to restore the VDI:

            vdi_string = "&vdi:%s=%s" % (<original_vdi_uuid>, <new_vdi_uuid>)

            task_ref = session.xenapi.task.create("import_vm", "Task to track vm import")

            url = ('https://%s/import_metadata?session_id=%s&task_id=%s%s'
                   % (host, session._session, task_ref, vdi_string))

            with open(<vm_import_path>, 'r') as filehandle:
                urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
                with requests.Session() as http_session:
                    request = http_session.put(url, filehandle, verify=False)
                    request.raise_for_status()

        Where &lt;*vm\_import\_path*&gt; is the location of the VM metadata.

        The `vdi:` query parameter changes the VM from pointing to its original VDI to pointing to the new VDI created in the previous step.
        You might want to create multiple new VDIs.
        If you want to change multiple VDI references for your new VM, add a `vdi:` query parameter for each VDI to the import URL.

        The new VM is created from the imported metadata and its VDI reference is updated to point at the VDI created in the previous step.
        You can extract a reference to this new VM from the result of the task.
        For more information, see the [samples on GitHub](https://github.com/xenserver/xs-cbt-samples/blob/master/cbt_vm_metadata_import.py).

1.  Import the coalesced VDI snapshot to the XenServer host at the UUID of the VDI you want to replace with the restored version.
    This VDI can be either an existing VDI or the VDI created in the previous step.

    In Python, you can use the following code:

        session_id = session._session
        url = ('https://%s/import_raw_vdi?session_id=%s&vdi=%s&format=%s'
               % (<xs_host>, session_id, <vdi_uuid>, 'raw'))
        with open(<import_path>, 'r') as filehandle:
            urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
            with requests.Session() as http_session:
                request = http_session.put(url, filehandle, verify=False)
                request.raise_for_status()

    Where &lt;*vdi\_uuid*&gt; is the UUID of the VDI you want to overwrite with the restored data from the coalesced VDI and &lt;*import\_path*&gt; is the location of the coalesced VDI.