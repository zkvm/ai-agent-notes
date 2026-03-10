Phase 1 system prompt (ingest.go:293):

  You are an information extraction engine. Extract distinct, atomic
  facts from a conversation.

  Rules:
  1. Extract facts ONLY from user messages. Ignore assistant/system
  messages entirely.
  2. Each fact must be a single, self-contained statement (one idea per
  fact).
  3. Prefer specific details over vague summaries.
     - Good: "Uses Go 1.22 for backend services"
     - Bad: "Knows some programming languages"
  4. Preserve the user's original language.
  5. Omit ephemeral information (greetings, filler, debugging chatter
  with no lasting value).
  6. Omit information only relevant to the current task with no future
  reuse value.
  7. If no meaningful facts exist, return an empty array.

mem9's definition of "memory" = an atomic, reusable fact about the
user, as judged by an LLM reading what the user said in a conversation

The prompt examples reveal something specific — mem9 captures not just
  facts but life events, decisions, relationships, preferences evolving
  over time. That's essentially a person's trajectory, not just a
  profile.

  What's genuinely novel here is not "agent remembers you better." That's
   just good UX. What's novel is what becomes possible when you have a
  machine-readable, structured, searchable record of a person's stated
  context accumulated over time.

  ---
  Direction 1: Memory as a Mirror, Not a Tool

  Most AI memory is designed to serve the agent — recall context to give
  better answers. Flip it: serve the user by reflecting their own
  patterns back to them.

  Your memory is a record of what you've told agents over months —
  decisions you made, things you wanted, things you were struggling with,
   people you mentioned. You never re-read it. But it's a surprisingly
  honest record because you weren't performing for it — you were just
  asking for help.

  What if the agent surfaced: "Three months ago you said you wanted to
  spend more time on architecture work. Since then you've mentioned being
   stuck in meetings 23 times and architecture zero times." Or: "You've
  described this same problem to me four different ways over six weeks
  without resolving it."

  This isn't the agent being smarter. It's the agent making you smarter
  about yourself. The memory is the product, not the means.

  ---
  Direction 2: The Cold Start Problem for AI Products is Solved by
  Portable Memory

  Every AI product today starts from zero with every user. That's not
  because companies don't want personalization — it's because there's no
  portable way to bring user context across products.

  mem9's tenantID is essentially a portable identity layer. An AI product
   that lets you connect your existing tenant gets months of context on
  day one — your preferences, your background, your current projects. The
   cold start problem disappears.

  This flips the competitive dynamic: instead of each AI product trying
  to accumulate the most user data, users own their context and bring it
  to products. Products compete on what they do with your memory, not on
  who has the most of it. More interesting: a new AI product with zero
  users but great reasoning could immediately compete with incumbents if
  users can bring their own memory.

  ---
  Direction 3: Memory as the Substrate for Longitudinal AI Relationships

  Every AI interaction today is transactional — you have a problem, the
  agent helps, conversation ends. The relationship resets.

  But humans don't work that way with people they trust. A good advisor,
  mentor, therapist remembers not just what you said but how you've
  changed. They notice when your stated goals and your actual behavior
  diverge. They remember what you tried before and why it didn't work.

  Memory makes this possible for AI. Not by making the agent smarter, but
   by giving it the longitudinal view that transforms a transaction into
  a relationship. The agent can say "you tried this approach in September
   and abandoned it — what's different now?" or "this is the third time
  you've started this project — what would make it stick this time?"

  The interesting design question is what kind of relationship this
  enables that wasn't previously possible at scale. A therapist can
  maintain deep longitudinal context for maybe 30-40 patients. An agent
  with cloud memory could maintain it for everyone, simultaneously.
