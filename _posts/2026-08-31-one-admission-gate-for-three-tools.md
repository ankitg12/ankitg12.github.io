---
layout: post
title: "One Admission Gate for Three Tools, and a Reply Channel Back to the Model"
date: 2026-08-31
categories: ai agents productivity
series: "AI coding agent productivity"
---

Three extensions had grown that each did the same thing: subscribe to `tool_call`, check whether the tool name matched one specific string, and put a dialog in front of it.

One gated skill creation ({% post_url 2026-08-17-put-skill-creation-behind-a-confirmation-gate %}). One gated subagent spawns ({% post_url 2026-08-19-put-subagent-spawns-behind-an-approval-gate %}). A third gated the `learn` tool that writes to long-term agent memory. Each was individually justified. Together they were three copies of the same dispatch code, three places to fix a bug, and three entries in `config.yml` whose relative handler order was load-order-dependent and therefore not obvious.

The consolidation is unremarkable and that is the point. What was actually interesting only became visible afterwards.

## The dispatcher is a map, not a chain

An admission rule is a tool name plus a handler:

```ts
export interface AdmissionRule {
  toolName: string;
  handle(
    event: ToolCallEvent,
    context: ExtensionContext,
  ): Promise<ToolCallDecision | undefined>;
}
```

The dispatcher keys rules by tool name and subscribes once:

```ts
export class AdmissionDispatcher {
  private rules: Map<string, AdmissionRule> = new Map();

  register(rule: AdmissionRule): this {
    this.rules.set(rule.toolName, rule);
    return this;
  }

  async handle(event, context) {
    const rule = this.rules.get(event.toolName);
    if (!rule) return undefined;
    return rule.handle(event, context);
  }

  attach(pi: ExtensionApi): void {
    pi.on("tool_call", (event, context) => this.handle(event, context));
  }
}
```

A `Map` lookup rather than an array walk is a deliberate constraint: exactly one policy owns each tool, and a second registration for the same name replaces rather than stacks. Two dialogs for one tool call is a bug, not a feature, and the data structure makes it unrepresentable.

The whole extension entry point is then a registration list:

```ts
export default function toolAdmission(pi: ExtensionApi): void {
  const dispatcher = new AdmissionDispatcher();
  dispatcher.register(manageSkillRule);
  dispatcher.register(subagentRule);
  dispatcher.register(learnRule);
  dispatcher.attach(pi);
}
```

Adding a fourth gated tool is one file plus one line. Three `config.yml` extension entries collapsed to one.

## Returning a verdict throws away the interesting part

The `learn` gate offered four choices: Approve, Revise text, Save as Note instead, Deny.

In practice I kept landing between them. The proposed memory would be *almost* right — correct fact, wrong shape. Too much preamble. A file path dropped. Two lessons crammed into one sentence. My options were to hand-edit prose the model could rewrite in a second, or to deny and re-explain from scratch in the next message.

Both are wasteful for the same reason: the dialog was returning a **verdict** when I had **intent**.

## The block reason is already a return channel

The fix required no new mechanism, only noticing what the existing one does. A blocked decision carries a `reason` string:

```ts
export interface ToolCallDecision {
  block?: boolean;
  reason?: string;
  input?: Record<string, unknown>;
}
```

That string is not a log line. It is delivered to the model as the result of its own tool call. The existing "Revise text" path had been using it as an instruction channel all along — it blocks the call and tells the model to re-issue `learn` with the edited text.

So the string can carry *guidance* just as easily as it carries *finished prose*. A fifth option:

```
  Approve              — save to long-term memory
  Revise text          — you edit the memory yourself
> Suggest a change     — tell the agent what to fix; it rewrites
  Save as Note instead — redirect to a durable note file
  Deny                 — do not save
```

Choosing it prompts for free text and blocks with it:

```ts
return {
  block: true,
  reason:
    `User did not approve this memory and gave guidance instead:\n"${guidance}"\n\n` +
    `Proposed memory was:\n"${memoryText}"\n\n` +
    `Rewrite the memory to follow that guidance and re-issue the \`learn\` ` +
    `tool call. Do not ask for confirmation first.`,
};
```

Three details in that string are load-bearing:

| Element | Why it is there |
|---|---|
| The original `memoryText` | The model's proposal may be several turns back in a long context; restating it removes the guess |
| An explicit re-issue instruction | Without it, a block reads as a stop signal and the model reports failure instead of retrying |
| "Do not ask for confirmation first" | Otherwise the model asks permission to retry, which is a second round trip to answer a question already answered |

The retry lands back in the same gate, so a bad rewrite is caught by the same dialog. The loop is bounded by the human at every pass.

## Cost per unit of intent

| Option | What the user types | What the model does |
|---|---|---|
| Approve | nothing | writes the memory |
| Revise text | the full corrected sentence | writes exactly that |
| Suggest a change | "keep the path, drop the preamble" | rewrites, re-submits, re-gated |
| Deny | nothing | drops it |

"Revise text" is still the right choice when the exact wording matters and is short. "Suggest a change" wins whenever describing the fix is cheaper than performing it — which, for anything longer than a line, is most of the time.

## Two labels that are now too close

`Revise text` and `Suggest a change` sit adjacent in the list and read similarly. If the distinction proves muddy in daily use, the honest fix is relabelling to *Edit myself* and *Tell agent what to fix* — naming the actor rather than the action. Left alone for now; a guess about confusion is weaker evidence than a month of actually misclicking.

## What generalizes

An approval dialog is usually built as a boolean and then regretted. The pattern worth keeping is smaller than the extension:

```text
verdict-only gate  -> approve / deny, user re-explains on the next turn
gate with a reason -> approve / deny / here is what to fix instead
```

If the interception point can already send a string back to the model, the dialog can be a conversation rather than a switch. That is a property of the hook, not of the policy — any pre-execution tool hook with a message-carrying block result has it, and most of them are currently spending it on the word "denied".

## Source

- [Oh My Pi](https://github.com/can1357/oh-my-pi)
- Prior: {% post_url 2026-08-17-put-skill-creation-behind-a-confirmation-gate %}
- Prior: {% post_url 2026-08-19-put-subagent-spawns-behind-an-approval-gate %}
