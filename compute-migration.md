# Shared compute rollout

Installed package: `getcolors/automq` at published revision `010bc92f602f4fa7eb54113fdc016284b020ee01`.
The installed skill directories and root launchers were copied from a verified
Skills CLI installation of that revision. The existing skills-lock.json entries were updated from that installation.

The package now delegates compute, remote state and machine-key ownership to
colors-compute. Its shared and node state keys live beneath `<profile>/compute/`;
managed Kubernetes uses the library managed-cluster state. Existing application
state and persistent application data must be retained.

This is a payload/configuration refresh, not a resource or state migration.
No live provider calls, create/delete, private key reads, or state transfers
were performed. Legacy deployment state was not inspected. Before a real
operation, establish ownership and review an explicit migration from the old
compute state layout. The library refuses recognized legacy remote state;
do not remove that guard, discard old state, or treat a new empty state key as
proof that the existing deployment has no resources. Keep the committed destroy
guard and the deployment profile unchanged.

Validation: the actual copied green launcher completed `build` in
temporary directories with a sanitized environment and published dependencies.
Generated compute documents were present. Any rendered backend documents used
compute state keys and contained no credentials.
This proves offline rendering, not live credentials, migrated ownership, or
application health.
