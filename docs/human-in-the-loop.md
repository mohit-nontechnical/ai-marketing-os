# Human-in-the-loop

The core product decision in MohitOS: agents generate everything, humans ship everything. Every piece of content crosses a human tap before it reaches an audience, and the machinery around that tap (channels, callbacks, feedback, notification budgets) is where most of the design effort went.

## Telegram forum topics as channels

The brand workspace is a Telegram forum supergroup, used the way a team uses Slack channels:

- **Pinned function topics** act as standing channels: content calendar, selfie-video nudges, newsletter, and feedback. Every new function gets its own topic at build time; nothing new is allowed to default into the general chat (that rule exists because unrouted traffic is how notification spam creeps back).
- **One topic per approval.** Each post awaiting approval gets its own thread: a media preview (photo, video, or a carousel album) plus a card with inline buttons. When it is resolved, the topic title gets a checkmark or cross prefix and the topic is closed. The group's topic list reads as a live approval queue.

## The approval card

Each card offers three paths:

- **Approve & Post.** Server-side, the approve endpoint re-runs the caption lint and brand-claims lint before publishing; a violation returns a 422 and the card stays open. The lint list includes fabrication tripwires: specific phrases from anecdotes the LLM repeatedly invented are hard-blocked, so a fake story cannot ship even if a human is moving fast. (At its worst, one invented anecdote appeared in 18 queued posts in a single day. The gate caught them.)
- **Deny.** Post is cancelled, topic closed.
- **Change.** Sub-buttons for caption and platforms. Caption edits use Telegram's force-reply so the bot receives the reply without loosening bot privacy settings; the new caption is linted before it is accepted. Platform toggles patch the post immediately and re-render the card.

Two correctness details that mattered in practice:

- After any tap, the inline keyboard is removed by editing the reply markup specifically. Editing the message text fails on photo cards (they have captions, not text), which would have left live buttons behind and allowed double-approves.
- On first run, the bridge seeds all currently pending posts as already-pushed without sending anything, so a restart never sprays weeks of old drafts at my phone.

## The feedback loop: corrections become directives

The feedback topic is the self-training channel. Any message I send there flows:

1. Bridge picks it up and posts it to an ingest endpoint, tagged with the brand resolved from the source group.
2. A cheap LLM classifies which agent skill the note belongs to, constrained to that brand's skill allowlist (an IGTRE note physically cannot land in another brand's skill).
3. The normalized directive is stored in a `feedback_directives` table (statuses: applied, proposed, retired) and appended to that skill's `SKILL.md` under a "Learned directives" section.
4. The bot confirms back which skill was updated and what the directive says.

The result: every correction I send from my phone permanently changes how the content machine behaves. Directive hygiene (deduplication, superseding contradictions, retiring stale rules) is treated as a first-class concern, because a contradictory rulebook degrades both the machine and the training signal.

## Notification fatigue is a system failure

At its worst, the system sent me about 60 Telegram pings in a morning. That got a root-cause diagnosis like any other incident:

- Escalations fired once per unactioned idea instead of once per sweep, and each escalation double-sent through two code paths.
- Informational idea cards were sent without the silent flag, so context messages pinged like alerts.
- No quiet hours, so catch-up runs after laptop wake fired at whatever hour the lid opened.

The fixes are structural, not cosmetic:

- **Batched escalation.** All unactioned ideas roll into one card and one push per sweep.
- **Noise tiers in the notification layer.** Every notification carries a level: critical (audible ping, reserved for act-now items), info (delivered silently into chat history, the default), or silent (no Telegram at all; visible only in the studio's status surfaces). The stated budget is at most 3 loud pings a day, and Hermes is bound by the same rule.
- **Quiet hours** (22:00 to 07:30) and digest windows on notification sources, enforced in data rather than in each caller.

The watch item, recorded in the postmortem: any new code path that sends directly instead of going through the notification layer is how the spam returns.

## Why the human gate stays

It would be one line of code to auto-publish. It stays a human tap because:

- This is finance-adjacent content with compliance rules; a person is accountable for every claim that ships.
- The publish gate is what makes aggressive automation everywhere else safe. I can let agents generate freely because the blast radius stops at a queue.
- The bottleneck data backs it up: at one point the system had 94 pieces made and 11 shipped. The scarce resource is decision attention, so the system's job is to make each decision cheap (one card, full context, one tap), not to remove the decision.
