# EcoShare: brag plan

## Answers
- **What is it:** a neighbourhood app for passing on surplus food and household items, either for free or for community credits.
- **Who it's for, and what it does for them:**
  - Neighbours with good stuff they would otherwise bin, and neighbours who could use it.
  - The app lists items, matches them to people nearby, and makes the handover safe.
- **What sets it apart:**
  - Snapping a photo makes AI fill in the listing.
  - The "For you" feed is ranked by interest, distance and freshness.
  - A one-time pickup code means credits move only at handover.
  - Your impact grows as an EcoTree.
- **Most impressive claim:** "Snap, and AI fills it in." This is the app's own copy, from the welcome screen.
- **Visual hook:** real items (bread, a jacket, books) drop into a bin with a thud, and then get pulled back out by EcoShare.
- **Real UI and flow:** Share with AI suggest → Explore feed (Lugano) → Request and chat → Pickup code and coins → Profile with EcoTree and badges.
- **Tone:** `default`, warm and polished. Organic palette, soft spring motion, a little playful.
- **Share caption:** "Good food and good stuff shouldn't end up in the bin. EcoShare passes them to a neighbour instead."

## Angle
The user asked for a problem explanation, then **every feature demoed with its explanation side by side**.
- The video opens on the problem.
- Each feature then gets a split scene: the phone on the left shows the feature working, and the right side explains it in plain words.
- The target length is about 33s, longer than the usual 20s because the user wants every feature covered.

## Visual identity
These come from `frontend/components/ui/theme.ts` and the app icon.

- **Colours:**
  - Forest `#244D3A`
  - Sage `#A8C3A0`
  - Cream `#F7F5EE`
  - Reward `#FFD166`
  - Coral `#FF8C78`
  - Text `#222B25`
- **Type:** Plus Jakarta Sans 800/700 for headings, Inter for body text. The fonts come from the app's own `@expo-google-fonts` packages.
- **Phone screens:** rebuilt in HTML with the app's real tokens, copy, icon and listing photos, because the app is React Native and its components can't be dropped into a browser frame.
- **Names:** all neighbour names are fictional demo accounts (Luca, Giulia). No real user data is shown.

## Storyboard (30fps, 1920×1080, ~33s)
| # | Time | Scene | On screen | Sound |
|---|---|---|---|---|
| 1 | 0–3.0 | Hook: the problem | Photos of bread, a jacket, books and a mug fall into a bin. "Perfectly good stuff ends up in the bin." | Soft pad starts; each item lands with a muted thud |
| 2 | 3.0–6.0 | Problem, part two | Two neighbour pins on a map, a dashed gap between them. "…while a neighbour nearby could use it." | Two gentle plucks |
| 3 | 6.0–9.0 | Reveal | The icon springs in, then "EcoShare" and "Share food and things with your neighbours." | Whoosh, then the beat comes in |
| 4 | 9.0–13.5 | Feature 01: Snap, AI fills it in | The Share screen gets a photo; the AI button is tapped; title, category and allergen chips type in. Right: "Snap. AI fills it in." "Take a photo — a vision model drafts the title, category and allergens. You check every field." | Tap pop, then a sparkle |
| 5 | 13.5–18.0 | Feature 02: For you, nearby | Explore: Lugano header, bento tiles, cards fading in one after another. Right: "Discover what's near you." "A For you feed ranked by your interests, distance and freshness." | Soft ticks as cards land |
| 6 | 18.0–22.5 | Feature 03: Request and chat | Item detail; the "Request this" button is tapped; chat bubbles appear. Right: "Request & chat." "One tap reserves it. Agree a pickup time in chat — AI screens messages for abuse." | Tap, then message blips |
| 7 | 22.5–27.0 | Feature 04: Safe handover | The pickup code digits appear, the item travels from owner to requester, a check pops and coins fly. Right: "Safe handover." "A one-time pickup code confirms it. Credits move only when the exchange is done." | Coin chimes in key |
| 8 | 27.0–30.5 | Feature 05: Impact | Wallet count-up, the EcoTree grows from 🌱 to 🌳, badges flip in. Right: "Watch your impact grow." "CO₂ avoided, an EcoTree that grows, and badges for sharing." | Rising arpeggio |
| 9 | 30.5–33.5 | Outro | Icon, "EcoShare", "Keep good stuff in use." | Final chord, short tail |

## Readability
- Each right-hand explanation runs about 14–18 words and stays settled for at least 3s.
- The hook line is 7 words, held for 2s.

## Sound
- **Music:** F major, 96 BPM, with a warm pad, a soft plucked arpeggio, and a light kick and shaker from the reveal onwards.
- **Effects:** tuned to F major and mixed under the music.
