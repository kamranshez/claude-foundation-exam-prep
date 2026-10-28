# Prompt Engineering

Prompt engineering is the practice of writing clear, structured instructions so Claude reliably produces the output you want.

## 1. Why It Matters

Claude follows instructions closely, so the quality of the prompt largely sets the quality of the result. Improving a prompt is usually cheaper and faster than fine-tuning or changing models.

## 2. Core Techniques

| Technique | What to do | Example |
|-----------|-----------|---------|
| Be clear and direct | State the task, audience, format and constraints explicitly | "Summarize this contract in 5 bullets for a non-lawyer." |
| Give context | Explain the purpose so Claude can make better choices | "This summary goes to the CFO, who cares about costs." |
| Use examples (few-shot) | Show 2-5 varied input/output pairs | Wrap in `<example>` tags |
| Let Claude think | Ask for step-by-step reasoning on hard tasks, or enable extended thinking | "Think through the problem before answering." |
| Use XML tags | Separate instructions, documents and examples | `<document>...</document>` |
| Assign a role | Set expertise and tone in the system prompt | "You are a senior security reviewer." |
| Prefill / format hints | Start the answer or specify the format to control output | Ask for JSON only, or begin with `{` |
| Chain prompts | Split complex work into steps, each with one goal | Extract, then analyze, then write |

## 3. Structuring a Prompt

A solid prompt usually has these parts, in roughly this order:

1. **Role / system prompt**: who Claude is and general rules.
2. **Context and documents**: long inputs go near the top, wrapped in tags.
3. **Task instructions**: what to do, placed after the documents.
4. **Examples**: representative input/output pairs.
5. **Output format**: length, structure, tone.

```text
<document>
{{contract_text}}
</document>

Summarize the contract above in 5 bullets for a non-lawyer.
Flag any clause that auto-renews. Respond in plain text, no headings.
```

## 4. System Prompt vs User Prompt

| Part | Use for |
|------|---------|
| System prompt | Persistent role, rules, tone, constraints |
| User message | The specific task and its input data |

Keeping stable instructions in the system prompt also helps prompt caching, since the prefix stays identical between calls.

## 5. Iterating and Evaluating

- Define success criteria first (accuracy, format, tone, latency).
- Build a small test set with typical and edge cases.
- Change one thing at a time and compare results.
- Say what to do instead of only what not to do.
- If output is wrong, work out which part of the prompt was ambiguous.

## 6. Common Mistakes

- Vague requests ("make it better") with no criteria.
- Contradictory instructions in different parts of the prompt.
- Giving a rule without its reason, so Claude cannot generalize it.
- Examples that are too similar, so Claude copies their pattern.
- Placing untrusted user input next to instructions without delimiting it (prompt injection risk).

## Key Takeaways

- Be explicit: task, audience, format and constraints.
- Examples and XML tags are the most reliable ways to steer format and structure.
- Put long documents first and the question after them.
- Use the system prompt for stable role and rules, the user turn for the task.
- Break complex tasks into chained prompts and give Claude room to reason.
- Test prompts against a set of cases and change one variable at a time.

## Practice Questions

**1. Why wrap documents and examples in XML tags?**
They clearly separate instructions from data, which reduces misinterpretation and makes prompts easier to parse and reuse.

**2. Where should a long document go relative to your question?**
Near the top, with the question or instructions after it; this tends to improve response quality on long inputs.

**3. Which is better: "Don't be wordy" or "Answer in at most three sentences"?**
The second. Positive, specific, measurable instructions work better than vague negatives.

**4. What is few-shot prompting and when is it useful?**
Including a few example input/output pairs in the prompt. It is useful when you need a specific format, style or classification behavior that is hard to describe in words.

**5. A task needs extraction, analysis and report writing. How should you prompt it?**
Chain prompts: one step per goal, passing each output into the next. This is easier to debug and usually more accurate than one giant prompt.
