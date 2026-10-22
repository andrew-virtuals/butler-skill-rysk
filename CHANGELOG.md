# Changelog

## 0.1.0

- First release. Screens and quotes fully collateralised cash-secured puts on Rysk
  V12 over its RFQ socket, against live market-maker quotes.
- **Physically settled**: an assigned put delivers the asset, so "you now own 2 ETH
  at $2,150" is true rather than a cash loss dressed up as a purchase.
- Handles the two things the venue does not document: `price` is **per unit**, and
  sizes step per asset (ETH 0.5, BTC 0.05, SOL 10, HYPE 50) with the steps discovered
  by walking a ladder rather than hard-coded.
- A short size fit is a **note**, not a refusal — gating on it hid HYPE, the
  best-paying asset on the venue.
- Carries a stdlib WebSocket client because Rysk is RFQ over a socket and the
  container ships no websocket package.
- Quotes carry `valid_for_seconds`; anything under twenty is refused, since a signed
  quote lives sixty seconds.
