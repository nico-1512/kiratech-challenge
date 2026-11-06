# Part 4 - Work with AI assistants
I used Codex with the codebase while implementing and documenting the team environment, ChatGPT in an online chat for technical discussion, and OpenCode with the codebase. I also used OpenCode with the Big Pickle free agent (200k context window), primarily to improve, refine, and review Bash scripting after independently provisioning and implementing the scripts.

I did not use an AI assistant to operate the Kubernetes cluster or make live changes. Cluster operations affect shared state, so I kept them under direct operator control.

One incorrect suggestion during this work was to delete a team Application with `kubectl` while leaving its request file in Git. The request directory is watched by an Argo CD Application with automated sync, so Argo CD would recreate the deleted request. I caught this by tracing which Argo CD Application owns the request directory and checking its prune and self-heal settings. I changed the instructions to remove the request file from Git instead.

Another proposal added KRO to the declarative GitOps setup. KRO is a useful tool, but it added an operator and another abstraction for a challenge that could be met with a smaller Argo CD and Helm implementation. I compared it with the stated requirements and kept the simpler design, while retaining admission validation and the team resources.

I reviewed the generated manifests, chart values, Bash scripts, and admission policy against the repository structure and the challenge requirements. AI tools were used as development and review assistants, while implementation decisions, validation, and cluster operations remained under my control.