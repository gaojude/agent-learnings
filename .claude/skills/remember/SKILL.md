---
description: Save a learning to the agent-learnings git repo
argument-hint: "[topic: description of the learning]"
---

Save the following learning to the git-backed memory repo:

$ARGUMENTS

Instructions:
1. Parse the topic from the input (text before the colon, or generate a slug from content)
2. IMPORTANT: Always save to the existing `learnings/` folder - do NOT create new folders like `patterns/`, `notes/`, `skills/`, etc.
3. Create a new file at ~/code/projects/agent-learnings/learnings/{topic-slug}.md
4. Format the learning with:
   - Title as H1
   - Date (today's date)
   - Project context if relevant
   - The learning content with clear sections
5. Commit and push to remote:
   ```bash
   cd ~/code/projects/agent-learnings && git add -A && git commit -m "Add learning: {topic}" && git push origin main
   ```
6. Confirm what was saved, output the raw GitHub URL, and copy it to clipboard:
   ```bash
   echo "https://raw.githubusercontent.com/gaojude/agent-learnings/refs/heads/main/learnings/{topic-slug}.md" | pbcopy
   ```
   Then display the URL to the user.
