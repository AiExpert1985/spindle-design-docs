**File Name**: Community
**Feature**: Identity 
**Phase**: 4
**Created**: 20-Mar-2026
**Modified**: - 

---

**Purpose:** build a sense of shared identity and mutual accountability among users. Grow from one-way story sharing in Phase 1 into a full community with public profiles and follow-up features in Phase 4. Each phase builds on demonstrated demand from the previous one — no phase is built speculatively.

---

## The Core Idea

Every user already has an identity in the app — a nickname and a level earned through real commitment work. Community features make that identity visible to others and create a space where users encourage, support, and learn from each other. The progression from solo habit tracker to social accountability platform is gradual and grounded in what users actually use.

---

## Phase 1 — Story Sharing (Ship with Phase 1 product)

**What it is:** users submit short success stories from inside the app. The developer reviews and publishes the best ones to the app's social media channels. Selected stories may receive a recognition reward inside the app.

**Why Phase 1:** solves the cold start problem before it exists. User-to-user community requires critical mass — without enough users, a community channel is a ghost town. Story sharing requires no critical mass. Even one good story is publishable. It builds social proof and marketing content simultaneously, at zero infrastructure cost.

**How it works:**

- A simple in-app submission form: story text, optional commitment name, optional permission to show nickname and level
- Submissions go to the developer for review — no automated publishing, no user-to-user visibility
- Selected stories are published to the app's social media (Instagram, X, Facebook, or wherever the audience is)
- Featured users receive a recognition marker in the app (a badge, a highlight on their profile) — not a functional reward like premium access, which would incentivise performative rather than authentic stories

**Build cost:** minimal. A text input, a submit action, a submitted flag on the record, and a simple review queue for the developer. No community infrastructure, no moderation system, no feed.

---

## Phase 2 — External Community Channel

**What it is:** an external community channel (Telegram group, Discord server, or similar) where users can connect, share progress, and interact with each other and the developer.

**Why external first:** building an in-app community requires feeds, moderation tools, reporting, content storage, and ongoing infrastructure. An external platform (Telegram, Discord) provides all of this for free and users already know how to use it. The goal of Phase 2 is to test whether users actually want community — not to build the full in-app version speculatively.

**What the channel provides:**

- Users share progress with their nickname and level as identity
- Developer shares product updates, featured stories, and commitment tips
- Users encourage and teach each other
- Direct feedback channel from users to developer

**Success signal for Phase 3:** if the external channel is active — regular posts, users helping each other, organic growth — that is the signal to invest in building the in-app version. If it is quiet, the in-app version would also be quiet.

---

## Phase 3 — In-App Community Feed

**What it is:** community functionality moves inside the app. Users can post updates, react to others' posts, and see a feed of activity from the community.

**Why in-app:** once community demand is demonstrated, bringing it inside the app creates tighter integration with commitment and progression data, keeps users in the product, and allows the nickname and level to display natively alongside posts.

**Key design decisions to make at build time:**

- What can be posted — stories, milestone celebrations, questions, or open posts
- Whether posts are moderated before publishing or reported after
- How nickname and level are displayed alongside posts
- Whether the feed is global or filtered by commitment type or level range

**Moderation reality:** an in-app community requires active moderation. Spam, negativity, and bad actors will appear. This is ongoing work for the developer or a hired moderator. Do not build this without a plan for who moderates and how.

---

## Phase 4 — Public Profiles and Follow System

**What it is:** users can optionally make their account public. Other users can follow them and see their commitment progress. Identity becomes a meaningful social layer.

**The follow system:**

- Public accounts are opt-in — private by default
- Followers see commitment names, current streaks, and level — not log details or notes unless the user chooses to share them
- Users can revoke public status at any time, which removes their profile from all follower feeds immediately

**Design decisions to make at build time:**

- Exactly what data is visible on a public profile
- Whether followers see real-time updates or daily summaries
- Whether there is a mutual follow (friend) concept or one-way only
- How to handle users who make their profile public, gain followers, then go private — what do followers see?

**Privacy first:** the consent flow must be explicit and clear. Users must understand what they are making visible before they confirm. The default is always private.

---

## Identity Design Note (Relevant from Phase 1)

The nickname and level are the identity layer for all community features. For community to work, these must be designed from the start with public display in mind:

- **Nickname** must be unique across all users — enforce at account creation
- **Level** must be something users are proud to display — the progression system should make level feel earned, not arbitrary
- **Nickname changes** should be limited (e.g. once per month) — frequent changes break community identity continuity

If nickname uniqueness and pride-worthy level display are not designed into Phase 1, retrofitting them for Phase 4 is significant rework.

---

## What This Is Not

- A dating or friendship app — the social layer is built around shared commitment work, not general social networking
- A public leaderboard — ranking users against each other creates anxiety and discourages users who are behind. Recognition is for personal achievement, not relative ranking
- A support group — the tone is encouragement and accountability, not therapy. The app does not position itself as a mental health tool