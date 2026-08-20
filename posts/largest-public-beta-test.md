---
title: We are all participating in the world's largest public beta test
date: 2026-06-22
permalink: /largest-public-beta-test
---

# We are all participating in the world's largest public beta test

A friend hit the thumbs-up button on a chatbot response over coffee last week. They did it without thinking. The answer was good, the button was right there, and the gesture was as light as liking a post on a social feed.

What that click *is*, technically, is a data point in the largest behavioral training loop in the history of consumer software. Hundreds of millions of people, doing this every day, on every model from every provider, are shaping the behavior of the next generation of AI assistants. Most of them have no idea.

I do not think this is a scandal. I think it's worth understanding clearly, because the more I look at it, the more I find it genuinely novel. And the framing of "beta tester" is more honest than "user" right now.

## What a beta used to be

"Beta" is a software term with a specific shape:

- A piece of software, mostly working, sent to a limited cohort.
- The cohort uses it, finds bugs, sends them back.
- The maker fixes the bugs and ships a real version.

Gmail famously stayed in beta for five years. Chrome ships a beta channel today. Game studios run weeks-long public beta phases. The contract is well understood: you get to use the unfinished thing, the maker gets to learn what's broken, and the artifact of the iteration is *code*, a piece of software a human deliberately writes after reading your report.

LLM "betas" (and I'm going to call them that for the rest of this post) are different. They're vastly larger in scale, and the artifact of the iteration is fundamentally different in kind.

## How an LLM actually gets better

A modern instruction-tuned LLM goes through roughly three stages.

1. **Pretraining.** A very large neural network learns to predict the next token across an enormous corpus of mostly-public text. This is the part everyone talks about. It produces a "base model", something with vast world knowledge but no particular bias toward being helpful in conversation.
2. **Instruction tuning.** A much smaller, more carefully curated dataset of "prompt → ideal response" pairs is used to fine-tune the base model into something that follows instructions.
3. **Preference learning.** The model is shown its own outputs, ranked by humans (or by other models trained on human rankings), and trained to prefer responses that humans prefer. This is the family of techniques that includes [RLHF](https://en.wikipedia.org/wiki/Reinforcement_learning_from_human_feedback) and, more recently, simpler approaches like DPO.

The pretraining data is more or less fixed. It's a snapshot of the public internet at some past date. The interesting iteration loop happens on top of it, in stages 2 and 3.

That iteration loop is fed by user data.

## What "your data" actually looks like in the loop

When you have a conversation with an LLM, several things can flow back to the provider, depending on the product tier and the toggles you've set:

- **The conversation itself**: your prompts, the model's responses.
- **Explicit ratings**: the thumbs-up / thumbs-down you may or may not have clicked.
- **Implicit signals**: did the conversation continue? Did you copy a piece of code out? Did you regenerate the response? Did you reword your question (suggesting the first answer missed)?
- **Edits to model output**: when you take the model's draft and rewrite the second paragraph, that delta is a label.

Aggregated across the user base, this is an absurd amount of behavioral information. Traditional product feedback (surveys, NPS scores, user interviews) is slow, expensive, and badly biased toward who actually fills out the survey. What an LLM provider gets instead is a billion-person, 24/7 UX research panel, with real tasks and real moments of "yes, that one".

Some of that flows into the next round of preference learning. Some of it flows into the evaluation sets that determine which model gets shipped. Some of it just informs product decisions ("turns out half our usage is X, who knew").

## The genuinely novel thing

Every previous beta was about iterating on code. The thing that improved between version 0.9 and version 1.0 was a thing humans deliberately edited.

With LLMs, the artifact that improves between version 1.0 and version 1.1 is the *model weights*. The model weights are partly shaped by the aggregate behavior of the previous version's users. A correction I make today, by editing a draft or downvoting an answer, becomes (at population scale) a tiny pressure on what the next model will say to *somebody else* next year.

This is not a thing software users have done before.

It's not exactly authorship. The influence is statistical, anonymized, and indirect. It's not exactly tagging or labeling either, because most of the signal is behavioral rather than deliberate. I don't think we have a clean word for it yet. "Participant" works. "Beta tester" works. "Co-author by accumulation" might be closer.

## What you give and what you get

**What you get:** a tool that, by any reasonable measure, is improving faster than any previous category of software. The Claude or ChatGPT you used a year ago is dramatically worse than what you have now, and the change is concentrated in the post-training stages, the parts shaped by the feedback loop. The capabilities you depend on this year for free wouldn't exist in their current form without a year of usage data.

**What you give:** the shape of your problems. Sometimes the substance of your code, your writing, your work. The patterns of how you ask, what you get stuck on, what you correct, what you accept. You give it to a provider you've made a contract with, implicit if you didn't read the terms, explicit if you did.

Some of it leaks into the model. Most of it informs the product. None of it is "free" in any meaningful sense, even when no money is changing hands.

## Where opting out actually lands

The defaults vary wildly:

- **API access**: most providers, by default, do not train on API traffic. This is the tier developers and companies use when they don't want their inputs to inform anyone's model.
- **Enterprise / business tiers**: same story, generally with contractual guarantees.
- **Consumer free tier**: typically *does* train, usually with a toggle to opt out somewhere in settings. The toggle is rarely in the place you'd guess.
- **Consumer paid tier**: varies a lot by provider. Some treat it like enterprise; some treat it like a slightly fancier consumer tier.

The market is sorting itself into "use this product" vs "we use you" along this exact dimension. Companies that need to use LLMs in confidential contexts pay for the tier where the model is the product. Everyone else, by default, is a participant in the training loop, and pays in data instead of money.

If you've never looked at your provider's data settings, look at them. For a lot of casual usage the bargain is genuinely fine, so the answer isn't necessarily to opt out. But you should know what you're agreeing to before you click thumbs-up.

## Some lessons I keep coming back to

From spending a year watching this play out as both a user and an engineer:

1. **The norms aren't settled yet.** What's acceptable practice in 2026 is going to look quaint, in one direction or the other, by 2030.
2. **The defaults are not neutral.** "Default to training" and "default to not training" describe two very different products. Both are reasonable defaults; both have implications.
3. **The bargain is real.** Nothing about this is free. The improvements that make the tool useful enough to depend on were paid for, in aggregate, by the users who came before you.
4. **The "beta" framing is more honest than "product".** A product is a finished thing. A beta is a thing that's becoming. Today's LLMs are unambiguously the latter.
5. **Your thumbs-up is a vote.** It's tiny, statistical and anonymized, but multiplied by however many people share your taste, it actually changes what the next model says to someone else. That's a strange amount of authorship to give away without thinking about it.

I'm not arguing for or against participating. The fact is that I am participating. So is almost everyone reading this. I just want us to know we're doing it, and to take a beat occasionally before we click the button.

We are early. This is the part of the era that will look weird in hindsight. The norms will arrive late. The defaults will shift.

Until then, read the data settings. Make a choice. Don't sleepwalk through it.

And when the model nails something, when it actually understood what you meant on the first try, when it caught a bug you missed, when it wrote a paragraph better than you'd have written it, give it the thumbs-up. You're voting on what the model will be like for the next person.

## References

- [Reinforcement learning from human feedback (RLHF) — Wikipedia](https://en.wikipedia.org/wiki/Reinforcement_learning_from_human_feedback)
- [InstructGPT paper — *Training language models to follow instructions with human feedback*](https://arxiv.org/abs/2203.02155)
- [Direct Preference Optimization (DPO) paper](https://arxiv.org/abs/2305.18290)
- [Anthropic's *Constitutional AI* paper — RLAIF and self-critique](https://arxiv.org/abs/2212.08073)
- [Gmail beta — Wikipedia](https://en.wikipedia.org/wiki/History_of_Gmail) (the original five-year beta)
