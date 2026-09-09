# Authoritative DNS zone management

Keep three different states separate in code and UI:

- the zone and its RRsets on the PowerDNS primary;
- replication of that zone to the configured secondary nodes;
- public delegation at the domain registrar/parent zone.

A healthy replicated zone is not proof that the public Internet delegates the
domain to the cluster. Check delegation independently through more than one
public recursive resolver and compare the normalized observed NS set with the
active cluster's configured NS set. Show `delegated`, `not delegated`,
`partial/propagating`, and `unknown` as different outcomes. A missing
delegation must not prevent preparing records before a migration.

## System-managed RRsets

The apex SOA and apex NS RRsets belong to the DNS cluster. They may be shown to
operators, but ordinary record forms must not edit or delete them. Enforce this
in the domain service as well as the view because hidden buttons do not protect
direct POST or future API clients.

Do not apply the apex rule to NS records below the zone root: those records
delegate a child name and remain normal user-managed DNS data.

## Migration from another provider

Prepare and verify the new zone before changing the registrar's NS records.
Public DNS cannot enumerate arbitrary owner names, so never present recursive
DNS discovery as a complete zone copy. A safe convenience import may discover
explicitly documented common names and types, such as root A/AAAA/MX/TXT/CAA
and www A/AAAA/CNAME, but its preview must state what was queried and require a
manual comparison with the old provider. DKIM selectors, DMARC, verification
records and arbitrary service subdomains may be absent.

Use a preview before mutation, validate every RRset with the normal record
model, reject system RRsets, preserve TTLs, and apply a group of imported RRsets
in one PowerDNS PATCH where possible. Protect the apply step with the serial
seen during preview and with a deterministic fingerprint of the discovered
names, types and values; exclude recursive-cache TTL from that fingerprint
because it naturally decreases between preview and confirmation. Record every
changed/deleted RRset in the audit log, and stop rather than applying a
network-truncated or otherwise incomplete snapshot.

Once public delegation already points entirely at the new cluster, recursive
queries show the new zone rather than the former provider. Disable that form of
automatic import at that point; recovery must use an export, backup or the old
provider's API instead.

Public recursive discovery is external I/O and must not block opening the
management surface. Render the shell first and load the preview separately,
for example through `PjaxLazyLoad`. Prefer the host operating system resolver
before querying fixed public resolver IPs: containers and production networks
often permit their local resolver while blocking direct outbound UDP/53.

Treat resolver errors and timeouts as an unknown result, never as proof that an
RRset is absent. If any requested RRset could not be checked, show the failure,
keep the empty-state message hidden, and disable the import so that a partial
snapshot cannot silently replace working DNS data.
