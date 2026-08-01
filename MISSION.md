# The Aquinas Manifesto

## The Problem
In an era of ephemeral data and algorithmic surveillance, human thought is increasingly fragmented, commodified, and vulnerable. Our digital reflections are no longer our own; they are harvested, analyzed, and stored in the clouds of centralized entities, subject to the whims of the attention economy and the erosion of privacy.

But beyond privacy, there is a deeper problem: some of the most important questions a human being can ask have nowhere safe to go. Questions about faith, meaning, existence, and truth are often too taboo, too contentious, or too personal to ask openly. They sit in silence—unexamined, unexplored.

## The Mission
**Aquinas** exists to restore the sanctity of thought. It is a sanctuary for the intellect—a private, sovereign space where the user can engage in deep, rigorous, and uninterrupted inquiry into life's biggest questions.

Now that the technology is capable enough, everyone deserves access to a personal expert in philosophy and theology. Not locked behind a subscription. Not surveilled by a server. Not filtered by an algorithm. Yours, on your device, always.

## Who It's For
Aquinas was built with specific people in mind:

- **The curious and the doubting** — people carrying questions they feel they can't ask out loud, who deserve a private space to explore them honestly
- **Students and young people** — navigating faith, identity, and meaning without always having somewhere safe to bring the hard questions
- **Missionaries and remote workers** — doing meaningful work far from libraries and institutions, who need an intellectual companion and theological resource they can rely on offline
- **Anyone who has ever kept a profound question to themselves** — because they didn't know where to bring it

## The Principles
1. **Sovereignty:** The user is the sole owner of their data. No cloud, no sync, no surveillance. What is thought within Aquinas remains within the user's control.
2. **Permanence:** We reject the ephemeral. Aquinas is built for the preservation of ideas, utilizing a structured architecture that mirrors the permanence of a library.
3. **Rigorous Inquiry:** Inspired by the Scholastic method, the tool facilitates the dissection of arguments, the exploration of contradictions, and the pursuit of truth through structured branching and logical connection.
4. **Accessibility:** Aquinas is free. The biggest questions in the universe should not be paywalled. This is not a product chasing revenue—it is a resource built from genuine conviction.

## The Origin
This project sits at the intersection of its creator's deepest interests: philosophy and theology, design, and development. It was built not to demonstrate capability, but because it needed to exist—and because the people who need it most are often the ones least served by existing technology.

## Implementation note

The current Mac-hosted FastAPI service and local iOS HTTP connection are a development topology
for integrating the on-device product. They do not change the mission: production architecture
must keep model inference and user data under the user's control, without remote telemetry or a
third-party cloud dependency. See `MODEL-INTEGRATION.md` for the live implementation boundary.
