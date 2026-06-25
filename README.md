# RCV PingIt

**PingIt** is a multi-vantage ping / reachability monitoring module for the
Root Chain Ventures NOC Portal platform.

This repository holds **operator install artifacts only**. PingIt is distributed
as container images and a Helm chart from GHCR; no application source lives here.

## License

**RCV Community License 1.0** — free for personal / home-lab use and any
organization's own internal operations; a paid commercial license is required to
offer it to third parties as a hosted/managed/SaaS service. See [`LICENSE`](LICENSE).
Commercial inquiries: **legal@rootchainventures.com**.

## Install (recommended: with the Portal)

PingIt is designed to run behind the NOC Portal. The simplest path is the
platform umbrella chart — install the Portal and enable PingIt together. See the
[RCV-NOC-Portal](https://github.com/Root-Chain-Ventures-LLC/RCV-NOC-Portal)
quickstart, then add:

```bash
--set pingit.enabled=true --set pingit.postgres.password=<pg-password>
```

## Install (standalone, advanced)

PingIt can also be installed on its own via the generic `rcv-module` chart,
pointed at a running Portal. Copy [`values.example.yaml`](values.example.yaml),
fill in the blanks, then:

```bash
helm install pingit oci://ghcr.io/root-chain-ventures-llc/helm/rcv-module \
  --version 0.1.0 -n rcv -f values.example.yaml
```

Images: `ghcr.io/root-chain-ventures-llc/pingit-web`, `…/pingit-worker`.
