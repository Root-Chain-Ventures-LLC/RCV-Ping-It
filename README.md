# RCV PingIt

**PingIt** is a multi-vantage ping / reachability monitoring module for the
Root Chain Ventures NOC Portal platform.

This repository is an **operator pointer only**. PingIt is distributed as
container images from GHCR and is installed **through the NOC Portal** — not on
its own.

## License

**RCV Community License 1.1** — free for personal / home-lab use and any
organization's own internal operations; a paid commercial license is required to
offer it to third parties as a hosted/managed/SaaS service. See [`LICENSE`](LICENSE).
Commercial inquiries: **legal@rootchainventures.com**.

## Install

PingIt is added from the Portal — you never hand-wire its credentials:

1. Install the **NOC Portal** first — see
   [RCV-NOC-Portal](https://github.com/Root-Chain-Ventures-LLC/RCV-NOC-Portal).
2. Sign in, open **Modules**, choose **PingIt**, and click **Install**. The Portal
   mints PingIt's OIDC client + API key and — on **Docker** the Portal
   also deploys it for you via its bundled deployer. **On Kubernetes, module
   install is manual**: after registering, deploy with Helm following the
   "Add modules → Kubernetes" steps in
   [RCV-NOC-Portal](https://github.com/Root-Chain-Ventures-LLC/RCV-NOC-Portal#add-modules).


> On Docker you can alternatively scaffold this module manually with `./install.sh add-module pingit` from the Portal repo, then register it in the Portal UI.

Images: `ghcr.io/root-chain-ventures-llc/pingit-web`, `…/pingit-worker`.
