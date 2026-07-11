# Architecture

MohitOS is four processes around one Postgres database, with Telegram as the human interface and a self-hosted publisher at the end of the line. The design goal: the approve-to-publish loop must run entirely in the cloud, because my laptop may be asleep or off.

## System diagram

```mermaid
flowchart LR
    subgraph mac["Mac (may be off)"]
        studio["Studio UI<br/>Next.js 16<br/>idea cards, approval queue,<br/>calendar, observability"]
        worker["Worker<br/>pg-boss consumer<br/>50+ job handlers<br/>(render, caption, VO, newsletter)"]
    end

    subgraph cloud["Cloud (always on)"]
        db[("Supabase Postgres<br/>content, jobs (pg-boss),<br/>brands, audit_events")]
        subgraph droplet["DigitalOcean droplet ($6/mo, systemd)"]
            hermes["Hermes bridge<br/>1:1 COO bot"]
            wsbridge["Workspace bridge<br/>brand approval groups"]
        end
        postiz["Postiz (self-hosted OSS)<br/>social scheduler on its own droplet"]
    end

    subgraph ext["External services"]
        ai["AI APIs<br/>OpenRouter (text, images)<br/>fal.ai (video)<br/>ElevenLabs / Kokoro (VO)"]
        social["Instagram / Facebook<br/>(TikTok sandboxed)"]
        beehiiv["Beehiiv newsletter<br/>(drafted via Playwright,<br/>API create is plan-gated)"]
    end

    phone["Mohit's phone<br/>(Telegram)"]

    studio <--> db
    worker <--> db
    worker --> ai
    worker --> beehiiv
    hermes <--> db
    wsbridge <--> db
    wsbridge --> postiz
    postiz --> social
    hermes <--> phone
    wsbridge <--> phone
```

Key properties:

- **One database, no message bus.** pg-boss runs the job queue inside Postgres. Bridges enqueue jobs from the droplet; the Mac worker drains them whenever it is awake. Heavy work (rendering, Playwright) stays on the Mac because a 1 GB droplet cannot afford it.
- **The publish path never touches the Mac.** The workspace bridge reads the captioned queue from Postgres directly, publishes to Postiz in-process on approval, and writes status back. Media is pre-staged to Postiz at caption time (while the Mac is on) so the droplet can publish files it could never read from a laptop disk.
- **Claude Code is the operator console.** I build, debug, and run one-off operations through it; agent skills define house style and per-format recipes. The app itself calls models through OpenRouter with cost-based routing (see [cost-controls.md](cost-controls.md)).

## Content pipeline flow

The daily loop, from idea to published post to measured result:

```mermaid
flowchart TD
    scrape["Morning Desk job (pre-dawn)<br/>brain knowledge graph + Google News scrape<br/>+ record of what's already been made"]
    cards["~10 idea cards on /today<br/>each tagged with a format<br/>(carousel / short / animate)"]
    approve1{"Human gate 1:<br/>approve idea card"}
    gen["Generation job fires immediately<br/>carousel render / faceless short<br/>(VO + AI b-roll + captions) / creative image"]
    caption["Captioning<br/>brand-voice caption drafted,<br/>brand-lint + compliance gate,<br/>media pre-staged to Postiz"]
    card["Telegram approval card<br/>media preview + caption<br/>in the brand workspace group"]
    approve2{"Human gate 2:<br/>Approve & Post / Deny /<br/>Change caption or platforms"}
    publish["Cloud publish<br/>bridge calls Postiz in-process<br/>UTM-tracked link attached"]
    social["Instagram / Facebook"]
    metrics["Metrics poll-back<br/>publisher status reconciled to the DB,<br/>live post link replied into the card's topic"]

    scrape --> cards --> approve1
    approve1 -- approve --> gen
    approve1 -- reject/skip --> cards
    gen --> caption --> card --> approve2
    approve2 -- approve --> publish --> social --> metrics
    approve2 -- deny --> done["status: cancelled"]
    approve2 -- change --> caption
```

Notes on the two gates:

- Gate 1 (idea) exists so money and quota are only spent on content I actually want. A "green tier" of formats can skip this gate for auto-generation, but with hard daily caps (see cost controls). Publishing is never auto-approved regardless of tier.
- Gate 2 (publish) is enforced server-side: the approve endpoint re-runs caption lint and brand-claim lint and rejects violations with a 422 even if a bad caption somehow reached a card.
- Newsletters take a parallel lane: drafted into Beehiiv as drafts via Playwright browser automation (the create API is plan-gated), with a Telegram card linking to the draft for review. Automation never clicks Send.

## Telegram bot topology

Telegram is split into a parent and N brand children, on separate bot tokens (Telegram allows one long-poller per token, so separate bots avoid contention).

```mermaid
flowchart TD
    mohit["Mohit (phone)"]

    subgraph oneone["1:1 lane"]
        hermes["Hermes: the AI COO bot<br/>sole 1:1 surface<br/>business + cross-project notifications<br/>dispatches requests into MohitOS<br/>max 3 loud pings/day<br/>never publishes content"]
    end

    subgraph brands["Brand lane (one bot, N groups)"]
        ws["Workspace bot<br/>one poller, routes by chat id<br/>via the brands registry table"]
        igtre["IGTRE workspace<br/>forum group with function topics:<br/>calendar / selfie nudges /<br/>newsletter / feedback<br/>+ one topic per approval"]
        other["Next brand's group<br/>added with a DB row + env entry,<br/>zero new code"]
    end

    db[("Postgres<br/>brands table maps<br/>chat to brand: voice, lint rules,<br/>publisher channels")]

    mohit <--> hermes
    mohit <--> igtre
    ws --- igtre
    ws --- other
    hermes <--> db
    ws <--> db
```

Why it is shaped this way:

- **One file, two modes.** Both bridges run the same script with a mode flag and their own token and update-cursor file. Not a fork; a deployment concern.
- **Brand isolation is data, not code.** Each brand row carries its approval chat, its publisher channel ids, its voice rules, and its banned-claims list. Feedback sent in a brand's group can only train that brand's skills (the classifier is constrained to a per-brand allowlist). Adding a brand is a row, a Telegram group, and env entries.
- **Hermes is deliberately weak.** It can look things up, dispatch jobs, and nudge me. It cannot publish. Two earlier 1:1 bots were consolidated into it after agent sprawl proved worse than one good agent.
