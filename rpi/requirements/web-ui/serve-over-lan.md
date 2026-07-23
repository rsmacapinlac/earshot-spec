# Serve the web UI over the LAN

The application serves a web UI over the LAN, reachable at the Pi's IP on a configured
port.

The Pi is headless with a single LED as its only local feedback channel. The web UI is
the surface for everything the button cannot express — which, given only two gestures
exist (record/stop and shutdown-hold), is nearly everything.

- Bound per `[web].bind_address` and `[web].port`
  ([configuration.md](../../specs/configuration.md#web)); the default binds all interfaces
  so it is reachable at the Pi's LAN IP.
- **Trusted LAN, no authentication in v1.** Anyone who can reach the IP can use it.
  Acceptable for a home device; see [Out of scope (v1)](README.md#out-of-scope-v1).
- Reachable at `http://<pi-ip>:<port>/`, and at `http://<hostname>.local:<port>/` where
  the OS mDNS responder is running — earshot neither installs nor configures mDNS
  ([install-service.md](../../specs/install-service.md#fr-8-one-line-install)).

Without a LAN the device still records, but nothing else is reachable — see
[capability tiers](../non-functional/no-internet-graceful-degradation.md#capability-tiers).
