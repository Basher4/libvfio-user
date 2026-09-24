- Twin sockets are not implemented.

- My proposal to introduce DEVICE_IRQ_TRIGGER essentially reverts https://github.com/nutanix/libvfio-user/pull/389
        - Investigate how it differs
        - Are the any deficiencies in the original implementation? Is my implementation better in any way than the original?

- There is an issue with the client https://github.com/nutanix/libvfio-user/issues/279
        - Fixing this issue may be enough for me to revert the revert
        - I may need to implement twin socket, which looks like isn't implemented yet for SOCK at all?

- Would it be simpler to introduce some kind of "sychronizer" proxy that will do reordering? We see what came in last, we know that the client is waiting for a reply, so we buffer all server messages until we get a reply.

- Return a generation on every interrupt registration, and if a IRQ message arrives with an older gen, then the client can throw it away.
