# Gmail Email Feature 

This document explains how the Gmail integration works: when the system sends an email on its own, what the different email statuses mean, and where in the app this feature shows up.

## The short version

Formhaus Atelier is connected to a Gmail account. When certain things happen — an invoice is marked as sent, an order is ready for pickup, and so on — the system automatically emails the client or supplier for you. You don't have to write or send anything yourself. The system also keeps a record of whether the email actually went out, and it watches for bounce-backs or replies so you know if something went wrong.

## When exactly does an email get sent?

An email only goes out when a staff member takes one of these specific actions:

| What you do in the app | Who gets emailed | What the email says |
|---|---|---|
| Mark an **invoice** as **"Sent"** | The client | Their invoice is ready, with a link to view it online |
| Mark an **invoice** as **"Paid"** | The client | A receipt confirming payment was received |
| Mark a **local order** as **"Ready"** | The client | Their order is ready for pickup or delivery |
| Create an **overseas order** with a supplier attached | The supplier | A purchase order for the items being ordered |

That's it — those are the only four moments that trigger an email automatically. Every other status you can set (like "Draft," "On Hold," "Packing," "Out for Delivery," "Picked Up," etc.) does **not** send an email. This is intentional, so clients and suppliers aren't flooded with messages for every small internal step — they only hear from us at the moments that actually matter to them.

There's also a manual resend option, but it's only exists as a backend capability today — there is **no button in the app** for it yet. Adding a proper "Resend Email" button to the invoice/order pages would be a small follow-up feature, not something that exists today.

## What happens after you click the button?

Sending isn't instant — it happens in the background, usually within about 15 seconds. This means:

1. You mark the invoice/order as Sent/Paid/Ready (or create the overseas order).
2. The system quietly queues up the email.
3. Within moments, a background process picks it up and actually sends it through Gmail.
4. If it fails for some reason (e.g., a temporary connection hiccup), the system automatically retries a few times before giving up and flagging it.

You'll get an in-app notification either way, so you always know if an email went out successfully or ran into trouble.

## The different email statuses, explained

There are two separate things being tracked for every email: **did we manage to send it**, and **did it actually get delivered**. These are kept separate on purpose — successfully sending an email is not the same as knowing it landed in someone's inbox.

### 1. Send status (did we manage to send it?)

- **Pending** — The email is queued up and waiting to go out (this normally only lasts a few seconds).
- **Sent** — Gmail has accepted the email and sent it on its way.
- **Error** — Something went wrong while trying to send it. The system will automatically retry a few times; if it keeps failing, it will stop trying and an admin should check what's going on (usually a Gmail connection issue).

### 2. Delivery status (what actually happened to it after sending?)

- **Unknown** — The default. We don't have any signal yet that anything went wrong, but we also can't be 100% sure it landed in the inbox (this is normal — most emails stay in this state forever, since there's no "delivered" confirmation from Gmail).
- **Delivered** — Reserved for future use; the system doesn't currently have a way to confirm this positively.
- **Bounced** — The email came back as undeliverable (for example, the address was wrong or the mailbox was full). If this happens, an admin gets notified so the issue can be followed up on directly with the client or supplier.

### 3. Replies

If a client or supplier replies to one of these emails, the system detects it and creates an in-app notification like "X replied on [date]" so staff know to follow up — you don't need to be watching the inbox yourself.

## Where else Gmail shows up in the app

- **Invoice emails include a link** the client can click to view their invoice online — no login required, no PDF attachment (that's not built yet).
- **Every email sent, bounced, or replied-to generates an in-app notification**, so this feature adds to your existing notifications rather than replacing them — you don't have to check a separate inbox.
- **Connecting/reconnecting Gmail** is an admin-only action, done once from an admin settings area, and only needs attention again if the connection is ever revoked or expires.

## Good to know

- Only four actions trigger outbound emails today (invoice sent, invoice paid, order ready, supplier PO on order creation). If a future request needs a new automatic email (e.g., "order out for delivery"), We can add it won't be an issue.
- "Sent" does not guarantee the recipient read it or even received it — only "Bounced" is a hard signal that something went wrong.
- Resending only works today through backend/developer access — there's no in-app button yet for staff to resend an email themselves.
