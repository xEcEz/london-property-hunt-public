# Saved-Search Email Alerts — Per-Portal Setup (Flats Variant)

The flats variant's alert ingestion path relies on each portal sending instant email alerts to your hunt Gmail account when a new listing matches your saved search. This is a one-time setup, ~20 min total across the four portals.

All four saved searches should match the criteria from your `config.md`:
- 1–2 bedrooms
- Max rent £3,500 pcm (hard cap)
- Primary + secondary area list (set as broadly as the portal's filter allows)
- Unfurnished or part-furnished preferred (where filter supports it)

## Rightmove

1. Go to [rightmove.co.uk](https://www.rightmove.co.uk/) and sign in (create account on rvonaar@gmail.com if needed).
2. Search for flats in your first primary area (e.g. Islington). Apply filters: 1+ bedrooms, max 2 bedrooms, max £3,500 pcm, flat type, let agreed = exclude.
3. Click **Create alert** (or bell icon).
4. Choose **Instant alerts** (not daily digest).
5. Repeat for every primary area that doesn't appear in your first search's radius. Rightmove doesn't let you multi-select areas in a single saved search — you'll likely end up with 4–8 saved searches to cover the whole area list.
6. Verify: trigger a test alert by clicking "Send me a test email" or wait for the next real listing. An email should arrive from `property-alerts@rightmove.co.uk` (or similar `@rightmove.co.uk`) within minutes.

## Zoopla

1. Go to [zoopla.co.uk](https://www.zoopla.co.uk/) and sign in.
2. Search: enter "Islington" (or first area), filter to 1–2 bed, max £3,500 pcm, to-rent, flats only.
3. Click **Save search** → **Alert me instantly** (or "Alert me daily" if instant isn't offered; instant is preferred).
4. Repeat for each primary area. Zoopla also limits single-search multi-area selection.
5. Verify emails arrive from `@zoopla.co.uk` sender.

## SpareRoom

1. Go to [spareroom.co.uk](https://www.spareroom.co.uk/) and sign in.
2. In **Flats-to-rent** mode (not rooms), search with your criteria.
3. Click **Save this search**. SpareRoom saved searches come with email alerts on by default (daily or instant — pick instant).
4. Verify emails arrive from `@spareroom.co.uk`.

## OpenRent

1. Go to [openrent.co.uk](https://www.openrent.co.uk/) and sign in.
2. Use the search to set your criteria (1–2 bed, max £3,500, areas, furnished preference).
3. Click **Save search** or the bell icon.
4. Verify emails arrive from `@openrent.com` or `@openrent.co.uk` sender.

## Post-setup check

After setting up all four:
- Wait 24 hours.
- Check the Gmail inbox at rvonaar@gmail.com — you should see at least one email per portal over that window (assuming the market has new listings, which is typical in London).
- Confirm each email's sender matches one of the four domains used in the skill's Gmail query: `@rightmove.co.uk`, `@zoopla.co.uk`, `@spareroom.co.uk`, `@openrent.co.uk` (or `@openrent.com`).
- If a portal uses a different sender domain (e.g. `@rmemail.co.uk` or `@news.zoopla.co.uk`), update the Gmail query in `skill-flats.md`'s ALERT INGESTION section to include it.

## Troubleshooting

- **No alerts after 24h**: check the portal's email settings on your account page; some default to daily digest even when you picked instant. Turn off daily digest entirely to avoid duplicate emails.
- **Alerts in spam**: mark as not-spam and add a filter in Gmail to auto-mark as important and skip spam.
- **Alert volume too high**: narrow saved searches (tighter bed range, area list, price). The skill's hard filters will drop out-of-scope listings anyway, but reducing volume lowers Gmail MCP noise.
- **A portal doesn't offer saved-search alerts on your plan**: skip that portal for alerts; it still gets picked up via local scraping.
