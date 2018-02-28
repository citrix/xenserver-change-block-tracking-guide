# Enabling NBD connections on XenServer

XenServer acts as an NBD server and makes VDI snapshots available over NBD connections.
However, to connect to XenServer over an NBD connection, you must enable NBD connections for one or more networks.

> **Important**
>
> We recommend that you use a dedicated network for your NBD traffic.

By default, NBD connections are not enabled on any networks.

> **Note**
> Networks associated with a XenServer pool that have NBD connections enabled must either all have the `nbd` purpose or all have the `insecure_nbd` purpose. You cannot have a mix of normal NBD networks (`FORCEDTLS`) and insecure NBD networks (`NOTLS`).
> To switch the purpose of all networks, you must first disable normal NBD connections on all networks before enabling either normal or insecure NBD connections on any networks.

## Enabling an NBD connection for a network (`FORCEDTLS` mode)

We recommend that you use TLS in your NBD connections.
When NBD connections with TLS are enabled, any NBD clients that attempt to connect to XenServer must use TLSv1.2.
The NBD server runs in `FORCEDTLS` mode with the "fixed newstyle" NBD handshake.
For more information, see the [NBD protocol documentation](https://sourceforge.net/p/nbd/code/ci/master/tree/doc/proto.md).

To enable NBD connections with TLS, use the `purpose` parameter of the network.
Set this parameter to include the value `nbd`.
Ensure that you wait for the setting to propagate before attempting to use this network for NBD connections.
The time it takes for the setting to propagate depends on your network and is at least 10 seconds.
We recommend that you use a retry loop when making the NBD connection.

### Examples

You can use any of our supported language bindings to enable NBD connections.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
session.xenapi.network.add_purpose(<network_ref>, "nbd")
```

xe command line:

```python
xe network-param-add param-name=purpose param-key=nbd uuid=<network-uuid>
```

## Enabling an insecure NBD connection for a network (`NOTLS` mode)

We recommend that you do not enable insecure NBD connections.
Instead use `FORCEDTLS` NBD connections.
However, the ability to connect to the XenServer over an insecure NBD connection is provided for development and testing with the NBD server operating in `NOTLS` mode as described in the NBD protocol.

To enable insecure NBD connections, use the `purpose` parameter of the network.
Set this parameter to include the value `insecure_nbd`.
Ensure that you wait for the setting to propagate before attempting to use this network for NBD connections.
The time it takes for the setting to propagate depends on your network and is at least 10 seconds.
We recommend that you use a retry loop when making the NBD connection.

### Examples

You can use any of our supported language bindings to enable insecure NBD connections.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
session.xenapi.network.add_purpose(<network_ref>, "insecure_nbd")
```

xe command line:

```python
xe network-param-add param-name=purpose param-key=insecure_nbd uuid=<network-uuid>
```

## Disabling NBD connections for a network

To disable NBD connections for a network, remove the NBD values from the `purpose` parameter of the network.

### Examples

You can use any of our supported language bindings to disable NBD connections.
The following examples show how to do it in Python and at the xe command line.

Python:

```python
session.xenapi.network.remove_purpose(<network_ref>, "nbd")
```

Or, for insecure NBD connections:

```python
session.xenapi.network.remove_purpose(<network_ref>, "insecure_nbd")
```

xe command line:

```python
xe network-param-remove param-name=purpose param-key=nbd uuid=<network-uuid>
```

Or, for insecure NBD connections:

```python
xe network-param-remove param-name=purpose param-key=insecure_nbd uuid=<network-uuid>
```