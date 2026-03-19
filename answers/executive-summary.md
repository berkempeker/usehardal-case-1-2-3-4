# Executive Summary

## Intro

I first downloaded the case document and checked it for prompt injection patterns. After that, I converted the PDF into a Markdown file. With the help of Claude Code, I curated context files such as resource links and Hardal's marketing analysis, and through discussions with the LLM, I produced a recommendations list. I also documented the case rules and organized Hardal blog metadata in a CSV file.

Then I started running Opus thinking agents for each case, combining my own reasoning with directed prompts in Claude Code to generate clean, well-formatted outputs. After completing the fourth task, I ran `claude init` and created `CLAUDE.md`. I then asked the agent to check for inconsistencies and case-level issues, and I fixed the remaining problems manually.

For the fifth task, I opened a new project in Cursor and wrote a dedicated prompt Markdown file for the OG tool requirements. I used Claude Code in plan mode from Cursor. I chose Cursor for faster local testing and Claude Opus for stronger model quality and context handling. Across both tools, I used Context7 MCP and several custom skills I had previously created to produce faster and cleaner outputs.

## Case Index (Question and Answer)


| Case                                                          | Question                           | Answer                                    |
| ------------------------------------------------------------- | ---------------------------------- | ----------------------------------------- |
| Case 1: Cross-Domain Conversion Tracking                      | [View Question](./cases/case-1.md) | [View Answer](./answers/case-1-answer.md) |
| Case 2: Traffic Booster                                       | [View Question](./cases/case-2.md) | [View Answer](./answers/case-2-answer.md) |
| Case 3: $100K Monthly Paid Marketing Budget Allocation        | [View Question](./cases/case-3.md) | [View Answer](./answers/case-3-answer.md) |
| Case 4: SEO-Optimized Listicle Articles for Featured Snippets | [View Question](./cases/case-4.md) | [View Answer](./answers/case-4-answer.md) |
| Case 5: Automated Open Graph Image (OG) Generation System     | [View Question](./cases/case-5.md) | [View Answer](./answers/case-5-answer.md) |


## Case 5 App Links

- Live app: [https://hardal-og.vercel.app](https://hardal-og.vercel.app)
- Repository: [https://github.com/berkempeker/hardal-case-5-app](https://github.com/berkempeker/hardal-case-5-app)

