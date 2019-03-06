# Appendices

## Constraints

The following section lists advisories and constraints to consider when using changed block tracking.

-  Changed block tracking is available only to customers with an Enterprise license for XenServer.
    If a customer without an Enterprise license attempts to use an incremental backup solution for XenServer that uses changed block tracking, they are prevented from enabling changed block tracking on new VDIs.
    However, if the customer has existing VDIs with changed block tracking enabled, they can still perform other changed block tracking actions on these VDIs.

-  Changed block tracking information is lost on Storage XenMotion.
    If you attempt to migrate a VM that has VDIs with changed block tracking enabled, you are prevented from doing so.
    You must disable changed block tracking before Storage XenMotion is allowed.

-  If a host or an SR crashes, XenServer disables changed block tracking for all VDIs on that host or SR.
    Before taking a VDI snapshot, we recommend that you check whether changed block tracking is enabled.
    If changed block tracking is disabled and this is not expected, this can indicate that a crash has occurred or that a XenServer user has disabled changed block tracking.

    To continue using changed block tracking, you must enable changed block tracking again and create a new baseline by taking a full VDI snapshot.
    Subsequent changed block tracking metadata uses this snapshot as a new baseline.

    The set of snapshots and changed block tracking data captured before the crash cannot be used as a baseline or comparison for any snapshots taken after the crash.
    However, the set of incremental backups taken before the crash can be used to create a VDI image to use to restore the VDI to a previous state.

    For more information, see [Incremental backup sets](./using-with-vdi.md).

-  XenServer supports a maximum of 16 concurrent NBD connections.

-  Changed block tracking is not supported for VDIs stored on thin provisioned shared GFS2 block storage.

## Additional Resources

The following resources provide additional information:

-  [GitHub repository of sample code](https://github.com/xenserver/xs-cbt-samples)

-  [Citrix XenServer Management API Guide](http://developer-docs.citrix.com)

-  [Citrix XenServer Software Development Kit Guide](http://developer-docs.citrix.com)

-  [Citrix XenServer documentation](http://docs.citrix.com/en-us/xenserver.html)

-  [NBD protocol documentation](https://sourceforge.net/p/nbd/code/ci/master/tree/doc/proto.md)

-  [nbd-client manpage](http://manpages.ubuntu.com/manpages/zesty/man8/nbd-client.8.html)
