# Copyable Email and Message Text

When showing the user an email, reply, DM or other message they may copy and paste, never put it in a block quote (`>` lines). Copying a block quote brings along the `>` markers or mangles the line breaks.

## Required

- Put the message text in a fenced code block marked `text`, so it copies exactly as written.
- Inside the block, use plain text only: no Markdown bold, italics or link syntax. Write links as full URLs.
- Keep the real line breaks and blank lines between paragraphs, the way the email should look.
- Put notes about the message (what to check before sending, why it says something) outside the code block.

## Example

````text
❌ BAD
> Hi Megan,
>
> Here's the **summary video**.

✅ GOOD
```text
Hi Megan,

Here's the summary video:
https://www.loom.com/share/...
```
````
