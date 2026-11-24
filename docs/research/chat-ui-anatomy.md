# Research: what WhatsApp, Telegram and Hinge chat screens look like in screenshots (Android / iOS / desktop, light and dark)

> **Background research. Not normative.** Collected on 2026-11-24 to inform the [requirements](../requirements.md), [high-level design](../high-level-design.md) and [output format](../output-format.md). Where they disagree, those documents win. Each claim is tagged by how well it was checked: verified, believed, or unverified ([V]/[B]/[U] or similar). Re-check the unverified ones before relying on them. The experiment scripts mentioned below were not kept in the repository.

Evidence labels used below:
- **[V]** verified, with a source.
- **[V-src]** verified by reading the official open-source client code. Telegram Android is DrKLO/Telegram @21f7d10 (2026-11-12), Telegram iOS is TelegramMessenger/Telegram-iOS @24638bc (2026-11-03), and Telegram Desktop is desktop-app/lib_ui `ui/colors.palette`.
- **[U]** believed but unverified. Please check these against real sample screenshots.

## Key takeaways for the requirements/design

1. **Which side a bubble is on does not always tell you the sender.**
   - **WhatsApp:** mirrors the whole chat when its UI language is RTL (Hebrew). Outgoing bubbles move to the LEFT, incoming to the RIGHT, and the back button moves right. [V, community evidence: https://ux.stackexchange.com/q/113216, top answer, score 11]
   - **What decides the side:** the UI language, not the language of the message text. "Using the interface in English but chatting in Hebrew leaves the default LTR flow." [same source]
   - **Telegram Android:** never mirrors. Its manifest sets `android:supportsRtl="false"`, and `ChatMessageCell` places outgoing bubbles at `layoutWidth - backgroundWidth` (the right side) with no RTL check. [V-src]
   - **Telegram iOS:** `ChatMessageBubbleItemNode.swift` has no RTL/layoutDirection references, so it most likely does not mirror either. [V-src, by absence]
   - **Hinge:** has no Hebrew UI. Its app languages are EN, ES, IT, SV, FR, DE, NO, DA, NL, FI, PT. [V: help.hinge.co "Language and Measurement Settings", via search snippet] So it is probably not mirrored on iOS. On Android it is unknown. [U]
   - **Design consequence:** plugins must detect orientation for each screenshot. Ticks appear only on outgoing bubbles, and the bubble palette also helps. Also accept a `ui_rtl` hint, and flag the result when uncertain.
2. **The text inside a bubble follows its own direction.** Hebrew lines are right-aligned inside a bubble even when the bubble is on the left. Mixed Hebrew/English needs Unicode bidi handling after OCR. Telegram also decides per paragraph from the first strong character (`AndroidUtilities.isRTL(text)` in its code). [V-src]
3. **Date separators differ by app and platform. Only some apps need weekday resolution.**
   - **Telegram Android:** never shows "Today", "Yesterday" or weekdays. It shows `MMMM d` (e.g. "November 24") for dates in the last 365 days, otherwise `MMMM d, yyyy`. [V-src: `LocaleController.formatDateChat`, strings `chatDate`/`chatFullDate`]
   - **Telegram iOS:** shows "Today", otherwise "Month d" in the current year and "Month d, yyyy" before that. No "Yesterday", no weekdays. [V-src: `ChatMessageDateHeader.swift`]
   - **WhatsApp:** uses "Today", "Yesterday", weekday names for the past ~week, and full dates for older days. [U, exact rules per platform unverified]
   - Hebrew equivalents: היום (today), אתמול (yesterday), יום ראשון…יום שישי (Sunday…Friday), and יום שבת or שבת (Saturday). Full date: "24 בנובמבר 2026"; short: "24.11.2026". [V: CLDR via Babel 2.18 for weekdays, months and date formats; היום/אתמול are standard Hebrew]
4. **Colours are not a reliable fingerprint.**
   - WhatsApp has let users choose chat themes since Feb 2025: preset themes, custom bubble colours and 30 new wallpapers, set as a default or per chat. [V: https://blog.whatsapp.com/chat-themes-to-reflect-your-style]
   - Telegram has many themes, account colours and custom wallpapers. [V: https://telegram.org/blog/reply-revolution]
   - Design consequence: estimate the palette for each screenshot (the two dominant flat bubble fills plus the background) instead of hard-coding hex values. Ship the default hex values only as priors.
5. **Busy backgrounds are the norm.**
   - WhatsApp's default is a "doodle" pattern, refreshed in May 2024. [V: https://www.meta.com/design-at-meta/blog/whatsapp-user-interface-update/]
   - Telegram's default is a 4-colour gradient with a pattern overlay. On iOS the gradient colours are `#DBDDBB #6BA587 #D5D88D #88B884`. [V-src]
   - Segment by "flat, uniform, rounded-rectangle fill regions" (bubbles are solid fills), not by subtracting the background.
6. **Overlays must be masked or interpreted before messages are cut out.** These are:
   - status bar, app header, compose bar
   - the floating/sticky date pill while scrolling (WhatsApp and Telegram)
   - the scroll-to-bottom button
   - the "N unread messages" bar
   - the Android navigation bar or iOS home indicator

   The floating date is real information: it is the date of the topmost visible message. It is not a separator event, though, and it would duplicate real separators across overlapping screenshots.
7. **Time sits inside each bubble, bottom-right in LTR layouts, for WhatsApp and Telegram.** Ticks appear only on outgoing bubbles. On media the time sits on a dark translucent pill.
   - Telegram's 12h format uses English "AM/PM" even in Hebrew. Its time formatter is forced to `Locale.US` unless the language is ar or ko. [V-src: `LocaleController.java`]
   - Hinge does not show per-message times. It shows centred timestamp separators. [U]
8. **Clues to the capture time and the app:**
   - AOSP saves screenshots as `Screenshot_yyyyMMdd-HHmmss.png`, in local time. [V: AOSP `ImageExporter.java` FILENAME_PATTERN, https://android.googlesource.com/platform/frameworks/base/+/refs/heads/main/packages/SystemUI/src/com/android/systemui/screenshot/ImageExporter.java]
   - Samsung (since Oreo) and some custom ROMs append the app's display label (e.g. `…_WhatsApp`), which is a free hint at the app. [V: https://www.sammobile.com/news/oreo-samsung-naming-screenshots-capture ; LineageOS-style `Screenshot_20201215-090626_Settings.png` per an /e/OS commit]
   - Recommendation: prefer the timestamp in the filename over file mtime, since copying changes mtime.
9. **Reply snippets for media use fixed placeholder words**, e.g. Telegram's "Voice message", "Video message", "Sticker", "GIF". [V-src strings.xml] Matching a reply to its original must therefore support "snippet = media type label (+ thumbnail)", not just a text prefix.
10. **Hinge's chat is far simpler.** It has no replies (U), no photo sending (U; swipestats says the composer has no photo attachment), voice notes (V: help.hinge.co "What are Voice Notes"), a "Sent" label (V: https://www.slashgear.com/1829766/hinge-app-read-receipts/, 2025-04-14), and no read receipts. Its unique element is the opening card: the photo or prompt that was liked, plus the comment. "Your Turn" lives in the match list, not in the chat. [V: https://www.cometchat.com/blog/beyond-the-bubble-hinge]

## WhatsApp

**Regions**
- Status bar → header → message area → compose bar.
- Header on Android: back arrow, avatar, name, "online" or "last seen …", then video, call and ⋮ icons. [U]
- Header on iOS: back chevron (sometimes with an unread count), avatar, name and status, then video and call icons. [U]
- Compose bar on Android: a pill containing emoji, the "Message" placeholder, a paperclip and a camera icon, plus a round green mic/send button. [U]
- Compose bar on iOS: "+", a field with a sticker icon, then camera and mic icons. [U]
- Web/desktop: a left pane with the chat list (unless cropped) and the chat in the right pane. Windows and macOS have native apps whose styling differs from Web. [U]
- A round scroll-down chevron (sometimes with a badge) sits above the compose bar. [U]

**Background**
- The doodle wallpaper is the default. [V, Meta 2024]
- The dark background was `#0b141a`; Android beta 2.24.22.4 (Oct 2024) moved it to `#0a1014` and made incoming bubbles and the default wallpaper darker. [V: https://wabetainfo.com/whatsapp-beta-for-android-2-24-22-4-whats-new/]
- Dark mode became "one shade darker" in May 2024. [V, Meta]

**Bubble colours**
- These are the commonly cited WhatsApp Web values [U]:

  | Theme | Outgoing | Incoming | Chat background | Read-tick blue |
  |---|---|---|---|---|
  | Light | `#D9FDD3` | `#FFFFFF` | `#EFEAE2` | `#53BDEB` |
  | Dark | `#005C4B` | `#202C33` | `#0B141A` | — |

- From the iOS 2021 beta: light outgoing changed `#E1F6CA`→`#E2FDD5`, dark outgoing `#295F60`→`#204F46`. [V: https://www.imore.com/whatsapps-chat-bubbles-are-getting-larger-more-rounded-look-new-colors]
- BetaNews (2026-03-09) reports new-look Android bubbles in testing, so shapes may change. [V headline only; article returned 403]

**Tails** appear on the first bubble of a run from the same sender. Later bubbles in the run have no tail. [U]

**Time and ticks**
- Small grey time at the bottom-right. It sits inline after the last line if there is room, otherwise on its own line. 12h or 24h follows the system.
- Tick meanings [V: https://faq.whatsapp.com/en/iphone/28041116]:
  - one grey tick = sent
  - two grey ticks = delivered
  - two blue ticks = read
  - clock = pending
  - a blue microphone or "Opened" for voice notes and view-once media
- "edited" appears next to the timestamp. [V: https://blog.whatsapp.com/now-you-can-edit-your-whatsapp-messages, 2023-05-22]
- In a mirrored (RTL) layout the time probably moves to the bottom-left. [U]

**Date chips**
- An opaque rounded chip, centred (white or dark grey, with grey text). [U]
- Possible text values: "Today", "Yesterday", a weekday name, or a full date with or without the year. Older Android versions used uppercase. [U]
- The floating date chip appears at the top while scrolling. [U]

**Unread bar:** a full-width "N UNREAD MESSAGES" strip. [U]

**System messages**
- Encryption notice, a yellowish chip at the top of the chat: "Messages and calls are end-to-end encrypted. No one outside of this chat, not even WhatsApp, can read or listen to them. Tap to learn more." [V: https://ssd.eff.org/module/whatsapp] The wording varies by version.
- Security-code-changed notices, disappearing-messages on/off, blocked/unblocked, business-account notice. [U]
- "This message was deleted" (deleted by the sender) and "You deleted this message" (deleted by you), shown in italic with a 🚫 icon. [V text; icon and style U]
- Missed voice/video calls appear as a bubble with a phone icon. [U]

**Reply block**
- A tinted rounded box at the top of the bubble, with a coloured left bar, the sender's name ("You" or the contact), a snippet of 1–3 lines ending in "…", and a thumbnail on the right for media. [U]
- The sender's name colour matches the group colour. Tapping jumps to the original. [V for 2016 behaviour: https://www.androidpolice.com/2016/06/09/whatsapp-2-16-118-beta-adds-message-quotes-replies]
- Media snippets use labels such as "Photo", "Voice message (0:12)", "Video", "Sticker", "GIF", "Document". [U]

**Media** (all [U])
- **Image:** fills the bubble with a thin border, and the time and ticks sit over a dark gradient. With a caption, the time moves into the caption line.
- **Album:** a 2×2 grid; the fourth tile shows "+N".
- **Voice note:** avatar or mic icon, play button, waveform and duration.
- **Video:** play icon in the centre and duration at the bottom-left.
- **Sticker:** no bubble; the time sits in a small pill.
- **GIF:** a GIF badge.
- **Document:** a card with an icon, the filename and "pages • size • type".
- **Link preview:** image, title, description and domain, above the URL text.
- **Other cards:** location map, contact card with buttons, poll with options and vote counts, and view-once media.

**Emoji-only messages:** 1–3 emojis are shown enlarged, and adding any text returns them to normal size. [V-ish: wikiHow mirror https://kiwix.lema.org/wikihow_en_maxi_2023-03/Enlarge-Emoji-on-WhatsApp ; sources conflict on the size tiers] Whether there is no bubble at all is [U].

**Other**
- Forwarded messages show "Forwarded" or "Forwarded many times" at the top of the bubble. [U]
- Reactions show as a pill overlapping the bubble's bottom edge. [U]

## Telegram

**Regions**
- Android header: ←, avatar, name, "last seen recently", ⋮. [U]
- iOS header: "‹ Back" (with a chats count), centred name and status, avatar on the right. [U]
- Android compose bar: emoji, "Message", paperclip, mic. [U]
- iOS compose bar: paperclip, field with a sticker icon, mic. [U]
- The go-down button is white on Android (`chat_goDownButton #FFFFFF`). [V-src] There are extra floating buttons for @mentions and reactions. [U]
- Desktop: chat list on the left, chat on the right. With "adaptive layout" a wide window shows a centred column. [U]

**Default palettes** [V-src]

| Variant | Incoming | Outgoing | Time text | Checks | Reply bar | Wallpaper / other |
|---|---|---|---|---|---|---|
| Android "Classic" (`ThemeColors.java`) | `#FFFFFF` | `#EFFFDE` | in `#A1AAB3`, out `#70B15C` | `#5DB050` (green) | in `#599FD8`, out `#6EB969` | media time pill `#66000000` (black, alpha 0x66); unread bar bg `#FFFFFF`, text `#5695CC` |
| Android Night (`night.attheme`) | `#1F2123` | `#366CAF` | — | — | — | wallpaper `#0F0F10`, action bar `#232326`, compose `#1E1E1F` |
| Android Dark Blue | `#232E3B` | `#3E618A` | — | — | — | wallpaper `#151E27` |
| Android Day (blue bubbles) | `#F0F0F0` | `#2D7ED5` | — | — | — | — |
| iOS Day | `#FFFFFF` | `#E1FFC7` | — | — | — | gradient-and-pattern wallpaper |
| iOS Dark | `#1D1D1D` (alpha) | gradient `#61BCF9`→`#0088FF` | — | — | — | background `#000000` |
| Desktop default (lib_ui) | white | `#EFFDDE` | in `#A0ACB6`, out `#6DB566` | `#57B84C` | out `#5EB854` | service background `#517C417F`; unread bar `#FCFBFA` / `#538BB4` |

**Tails:** drawn at the bottom of the last bubble in a group. Earlier bubbles are "pinned". [V-src: `pinnedBottom` logic in ChatMessageCell]

**Time and ticks**
- `HH:mm` or `h:mm a` at the bottom-right. [V-src]
- One check = sent, two checks = read. There is no "delivered" state. [U]
- "edited" appears before the time. [V-src string `EditedMessage`]

**Date and service messages**
- A translucent dark pill with white text (`chat_serviceText #FFFFFF`). [V-src]
- The formats in takeaway 3 apply.
- Service messages use the same pill, e.g. self-destruct timer changes, "Missed Call", pinned-message notices, "joined Telegram". [V-src strings, e.g. `MessageLifetimeChanged`, `CallMessageIncomingMissed`]
- The unread divider reads "Unread Messages". [V-src]

**Reply block (Replies 2.0, Oct 2023)**
- A tinted block with a left bar, the name in the replied user's account colour (Premium users can add a pattern), and support for partial quotes with a quote icon.
- A reply can point to another chat.
- Link previews can sit above or below the text and can be resized. [V: https://telegram.org/blog/reply-revolution]

**Media**
- Voice note: waveform and duration. Video note: a circular video. Sticker: no bubble, with a time pill. GIF. [V-src strings "Voice message", "Video message", "Sticker", "GIF"]
- Album grid, document card, poll ("Poll", "Anonymous Poll", "Final Results" [V-src]), location, contact. [U]

**Forwarded:** "Forwarded from X" header. [V-src]

## Hinge (mostly [U]; build a sample corpus early)

- **How a chat starts:** A likes a photo or prompt, optionally with a comment. The match forms when B likes back or taps "Match". [V: help.hinge.co article 360011090134] The thread opens with that liked item (photo thumbnail, or prompt question and answer card) and the comment as the first message. [U]
- **Match Note:** shown to the matched person before the conversation starts, up to 255 characters. [V via search snippet of help.hinge.co "What is Match Note"]
- **Header:** back arrow, the match's name and photo, and a Chat/Profile toggle, so you can swipe to the profile from inside the chat [V-ish: CometChat], plus ⋮ (unmatch/report). [U]
- **Bubbles:** outgoing on the right in Hinge purple, incoming on the left in light grey. Plain white or black background with no wallpaper. [U]
- **Timestamps:** centred grey separators such as "Today 8:33 PM" or "Mon 4:12 PM" between message clusters. No per-message time. [U]
- **Status labels:** "Sent" under your latest message; no "Read" label. [V, SlashGear]
- **"Your Turn":** a section or badge in the match list, not in the chat. [V, CometChat; https://help.hinge.co/hc/en-us/articles/26848706496659]
- **Other elements:** typing indicator, message likes (hearts), and voice notes. [V, CometChat and Hinge help] "We Met" feedback cards may appear. [U] iOS and Android look nearly identical apart from the system bars. [U]

## Cheap auto-detect signals (single screenshot)

1. **Filename and metadata:**
   - Samsung appends the app label. [V]
   - AOSP uses `Screenshot_yyyyMMdd-HHmmss`. [V]
   - iOS saves PNGs (often `IMG_####.PNG`) and exact device resolutions, e.g. 1179×2556, 1290×2796, 1206×2622, 1320×2868, 1170×2532, 750×1334. [U]
   - macOS uses `Screenshot YYYY-MM-DD at HH.MM.SS.png`. [U]
2. **Platform from the system bars** [U]:
   - iOS: time on the left with no notification icons, and the battery with the percentage inside it on the right. The bottom has a home indicator.
   - Android: notification icons next to the time, signal/Wi-Fi/battery icons, and a gesture pill or 3-button navigation bar.
   - Desktop: no status bar, a wide aspect ratio, and often a sidebar list.
3. **App from layout:**
   - WhatsApp header has video and call icons, plus doodle texture.
   - Telegram iOS has the avatar on the right of the header, plus gradient+pattern wallpaper and translucent service pills.
   - Hinge has a plain background, a Chat/Profile toggle and the "Sent" label.
4. **Theme:** the mean luminance of background and header (trivial).
5. **Tick glyph style:** WhatsApp blue double ticks versus Telegram green or white checks.

The palette is only a soft prior (see takeaway 4).

## Risks & open questions

- Hinge visuals (bubble colours, separator format, opening card, RTL behaviour on Android) are almost entirely unverified. Hinge's help pages returned 403. Collect real screenshots before designing its plugin.
- WhatsApp hex values and date-chip rules per platform are unverified. There have been redesigns in 2021, 2024 and 2025, and new Android bubbles were reported in testing in 2026. Plugins should carry palette priors with version ranges and be calibrated from samples.
- WhatsApp Web/Desktop in Hebrew: mirroring is assumed, not verified. The tick-based orientation check should resolve it anyway.
- Telegram's Hebrew date pattern (the translated `chatDate`) and whether Hebrew is an official Telegram UI language are unknown. Telegram Desktop's RTL mirroring was not determined.
- Telegram's absence of "Yesterday"/weekdays is verified only for the current code. Older app versions, or screenshots from them, may differ.
- Floating date pills and partially scrolled bubbles at the top or bottom edges will create false or duplicate events. The design needs "cut bubble" handling and an "overlay" class.
- Group chats (later) add sender names, avatars and per-sender colours inside the reply block. The design should leave room for these.
