# Server-side request forgery (SSRF)

> Tricking **your server** into making requests to places the attacker can't reach directly —
> internal services, admin endpoints or the cloud metadata service.

[← Handbook](../README.md) · [All concepts](README.md)

---

## Where it comes from

Any feature where the server fetches a URL influenced by a user: link previews, "import from URL",
webhooks, image proxies, PDF generators that load remote assets, OAuth/OpenID discovery URLs.

## Why it's dangerous in the cloud

Cloud VMs and some runtimes expose a **metadata service** at a link-local address
(`169.254.169.254`) that can return the workload's **credentials**. An SSRF that reaches it can
hand an attacker your cloud identity. It can also reach private databases, internal admin APIs and
services that trust "anything inside the network".

## Defences, layered

1. **Allow-list destinations** where possible (only known webhook domains, only your storage bucket).
2. **Validate after DNS resolution.** Resolve the hostname, reject private, loopback, link-local and
   unique-local ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`,
   `169.254.0.0/16`, `::1`, `fc00::/7`, `fe80::/10`), then **connect to that exact IP** so DNS can't
   change between check and use (DNS rebinding).
3. **No redirects**, or re-validate every hop.
4. **Only `http`/`https`** — no `file:`, `gopher:`, `ftp:`.
5. **Limits:** short timeouts, maximum response size, no forwarding of raw responses to the user.
6. **Least-privileged egress:** run the fetcher as an identity with no useful permissions, behind an
   egress proxy or firewall that blocks internal ranges.
7. **Harden metadata access:** on AWS, require **IMDSv2** (session tokens, hop limit); on Google
   Cloud, the metadata server requires the `Metadata-Flavor: Google` header, which plain SSRF
   requests usually can't set — but don't rely on that alone.

## Example check

```ts
import { lookup } from 'node:dns/promises'
import ipaddr from 'ipaddr.js'

async function resolvePublicAddress(url: URL) {
  if (!['http:', 'https:'].includes(url.protocol)) throw errors.invalid('Unsupported scheme')
  const { address } = await lookup(url.hostname)
  const range = ipaddr.parse(address).range()
  if (range !== 'unicast') throw errors.invalid('Destination not allowed')   // private, loopback, linkLocal…
  return address   // connect to this IP (with the original Host header / SNI), redirects disabled
}
```

## Related

[Least privilege](least-privilege-iam.md) · [Workload identity federation](workload-identity-federation.md)

## Further reading

- OWASP, *Server-Side Request Forgery Prevention Cheat Sheet*: https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- AWS, *Use IMDSv2*: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
