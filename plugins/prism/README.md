# Prism Plugin

Author LaTeX papers and documents in OpenAI Prism with Codex: create new projects, draft and revise source, compile PDFs, diagnose LaTeX build issues, and download finished artifacts.

Learn more about Prism at <https://openai.com/prism>.

## Bundled skill

This plugin always routes Prism work through the bundled `$Prism` skill, which knows how to operate the Prism browser bridge end to end.

If Prism is not already logged in, the bundled `$Prism` skill opens the normal Prism login flow and waits for the user to authenticate before continuing.
