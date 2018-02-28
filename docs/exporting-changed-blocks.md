# Export changed blocks over a network block device connection

XenServer runs an NBD server on the host that can make VDI snapshots accessible as a network block device to NBD clients.
The NBD server listens on port 10809 and uses the "fixed newstyle" NBD protocol.
For more information, see [the NBD protocol documentation](https://sourceforge.net/p/nbd/code/ci/master/tree/doc/proto.md).

NBD connections must be enabled for one or more of the XenServer networks before you can export the changed blocks over NBD.
For more information, see [Enabling NBD connections on XenServer](./enabling-nbd.md).

## Getting NBD connection information for a VDI

From a logged in XenAPI session, you can use the `get_nbd_info` call to get a list of connection details for a VDI snapshot made available as a network block device.

These connection details are specific to the session that creates them and the NBD client uses this logged in session when making its connection.
Any set of connection details in the list can be used by the NBD client when accessing the VDI snapshot.

Each set of connection details in the list is provided as a dictionary containing the following information:

`address`:

-  The IP address (IPv4 or IPv6) of the NBD server.

`port`:

-  The TCP port to connect to the XenServer NBD server on.

`cert`:

-  The TLS certificate used by the NBD server encoded as a string in PEM format.
    When XenServer is configured to enable NBD connections in `FORCEDTLS` mode, the server presents this certificate during the TLS handshake and the NBD client must verify the server TLS certificate against this TLS certificate.
    For more information, see "Verifying TLS certificates for NBD connections".

`exportname`:

-  A token that the NBD client can use to request the export of a VDI from the NBD server.
    The NBD client provides the value of this token to the NBD server using the `NBD_OPT_EXPORT_NAME` option during the NBD option haggling phase of an NBD connection.

    This token contains a reference to a logged in XenAPI session.
    The XenAPI session must remain logged in for this token to continue to be valid.
    Because the token contains a reference to a XenAPI session, you must handle the token securely to prevent the session being hijacked.

    The format of this token is not guaranteed and might change in future releases of XenServer.
    Treat the export name as an opaque token.

`subject`:

-  A subject of the TLS certificate returned as the value of `cert`. This field is provided as a convenience.

### Examples

You can use any of our supported languages to get the list of NBD connection details for a VDI snapshot.
The following examples show how to do it in Python.

Python:

```python
connection_list = session.xenapi.VDI.get_nbd_info(<snapshot_vdi_ref>)
```

This call requires a logged in XenAPI session that remains logged in while the VDI snapshot is accessed over NBD.
This means that this command is not available at the xe command line.

### Errors

You might see the following errors when using this call:

**VDI\_INCOMPATIBLE\_TYPE**:

-  The VDI is of a type that does not support being accessed as a network block device.

    Check that the type of the VDI is not `cbt_metadata`.
    You can use the `get_type` call to find out the type of a VDI.
    If your VDI is `cbt_metadata`, you cannot access it as a network block device. For more information, see [Checking the type of a VDI or VDI snapshot](./deleting-snapshots.md).

**An empty list of connection details**:

-  The VDI cannot be accessed.

    Check that the XenServer host that runs the NBD server has a PIF with an IP address.

    Check that you have at least one network in your pool with the purpose `nbd` or `insecure_nbd`.
    For more information, see [Enabling NBD connections on XenServer](./enabling-nbd.md).

    Check that storage repository the VDI is on is attached to a host that is connected to one of the NBD-enabled networks.

## Exporting the changed blocks using an NBD client

An NBD client running in the backup location can connect to the NBD server that runs on the XenServer host and access the VDI snapshot by using the provided connection details.

The NBD client that you use to connect to the XenServer NBD server can be any implementation that supports the “fixed newstyle” version of the NBD protocol.

When choosing or developing an NBD client implementation, consider the following requirements:

-  The NBD client must support the “fixed newstyle” version of the NBD protocol.
    For more information, see [the NBD protocol documentation](https://sourceforge.net/p/nbd/code/ci/master/tree/doc/proto.md).

-  The NBD client must request an export name returned by the `get_nbd_info` call that corresponds to an existing logged in XenAPI session.
    The client makes this request by using the `NBD_OPT_EXPORT_NAME` option during the NBD option haggling phase of the NBD connection.

-  The NBD client must verify the TLS certificate presented by the NBD server by using the information returned by the `get_nbd_info` call.
    For more information, see “Verifying TLS certificates for NBD connections”.

> **Note**
>
> If you are using the Linux upstream NBD client, a minimum version of 3.15 is required to support TLS.

After the NBD client has made a connection to the XenServer host and accessed the VDI snapshot, you can use the bitmap provided by the `list_changed_blocks` call to select which blocks to read.
For more information, see [Getting the list of blocks that changed between VDIs](./list-changed-blocks.md).

> **Note**
>
> XenServer supports up to 16 concurrent NBD connections.

### Verifying TLS certificates for NBD connections

When connecting to the NBD server using TLS, the NBD client must verify the certificate that the server presents as part of the TLS handshake.

We recommend that you use one of the following methods of verification depending on your NBD client implementation:

-  Verify that the server certificate matches the certificate returned by the `get_nbd_info` call.

-  Verify that the public key of the server certificate matches the public key of the certificate returned by the `get_nbd_info` call.

#### Alternative approach

As a less preferred option, it is possible for the NBD client to verify the certificate that the server presents during the TLS handshake by checking that the certificate meets all of the following criteria:

-  It is signed by a trusted Certificate Authority

-  It has an `Alternative Subject Name` (or, if absent, a `Subject`) that matches the subject returned by the `get_nbd_info` call.