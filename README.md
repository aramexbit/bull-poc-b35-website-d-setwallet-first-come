# PoC for Bull.family bug bounty report B-35

## Website `/d/` `setWallet` is first-come with no auth beyond the secret URL (M-12 / K-1206)

**Report B-35.** The website chunk under test is `website-d-chunk.js` — the
`/d/` buyer page chunk live 20 Nov 2026 (`page-16d1cf063b916c2c.js`),
sha256 `4140f5163dc863341ecab713d67f62982d7908422705b26c4940904f7e64320c`,
12,878 bytes. The file is included verbatim; its sha256 must still match
after checkout.

### The bug

On the `/d/` buyer page, the connected wallet can claim the buyer slot with
one click ("Set as buyer wallet"), and the click handler is:

```js
async function $(){ W && await L({ action: "setWallet", wallet: W }) }
```

where `L()` is a plain unauthenticated POST to the deal's buyer endpoint:

```js
async function L(e){
  let n = await fetch("/api/buyer/".concat(t), { method: "POST",
    headers: { "content-type": "application/json" }, body: JSON.stringify(e) });
  // ...
}
```

So `setWallet` is just `POST /api/buyer/<token> {"action":"setWallet","wallet":<any-address>}`.
There is no signature, no proof of ownership of the demand, and no check
that the caller is the buyer who created the demand — the only "secret" is
the `/d/<token>` URL itself, which is shared with the counterparty (the
seller) by design. The page even shows the button whenever no buyer wallet
is set yet (`!z.buyerWallet && <button onClick={$}>Set as buyer wallet</button>`),
and once a wallet is set the real buyer sees "The connected wallet is not
your buyer wallet."

### How to reproduce (no funds, no signature)

1. Open `website-d-chunk.js` and find `async function $(){W&&await L({action:"setWallet",wallet:W})}`
   and `L()`'s `fetch("/api/buyer/".concat(t), {method:"POST", ...})` — note
   there is no signature or auth header anywhere in the call.
2. Open a fresh demand's `/d/<token>` URL before the real buyer sets their
   wallet, connect any wallet, and click "Set as buyer wallet" (or
   `POST /api/buyer/<token> {"action":"setWallet","wallet":"<attacker-address>"}`).
3. The attacker address is now recorded as `buyerWallet`. When the real
   buyer arrives, the page tells them their wallet is not the buyer wallet,
   and the "Set as buyer wallet" button is gone — the slot is taken.

### Honest impact

Denial of service / authentication gap, not theft: whoever sets the wallet
first locks the real buyer out of their own demand page (offers, confirm,
deposit and cancel all gate on `buyerWallet`). But the attacker cannot steal
funds this way — to open the escrow deal they would have to fund it
themselves with their own payment, and the seller's tokens go to the
recorded buyer wallet only after that funding. The precondition is knowing
the `/d/<token>` URL, which is shared with the counterparty by design, so
the realistic attacker is a counterparty (or anyone the URL leaks to), not
the open internet. The fix is to require the demand creator to prove
ownership (e.g. a signature over the token + wallet) before `setWallet` is
accepted, and/or to allow the creator to replace the wallet.

### Evidence

- `website-d-chunk.js` — live `/d/` chunk, verbatim (sha256 above).
