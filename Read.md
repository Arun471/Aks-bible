1. Install prerequisites
Claude Code: npm install -g @anthropic-ai/claude-code
Azure CLI, then authenticate: az login
kubectl if you don't have it already
2. Add the AKS MCP server (official, from Microsoft — exposes az and kubectl operations to Claude, plus AKS-specific diagnostics like node log collection and control-plane logs)
Docker is the simplest cross-platform install:
Code
Start with --access-level readonly. Switch to readwrite (or admin, which unlocks fetching cluster credentials) only once you trust the workflow — those levels let it create/delete node pools, scale clusters, and apply manifests.
3. Optional: add the general Azure MCP server too, for anything outside AKS itself (resource groups, ARM deployments, networking):
Code
4. Give it project context. Create a CLAUDE.md in your project folder noting your subscription, resource group, cluster name(s), and any recurring gotchas (e.g. "our ingress uses App Gateway, not nginx"). Claude reads this at the start of every session, so it doesn't relearn your setup each time.
5. Just talk to it. Example prompts:
"Why is node aks-nodepool1-xxx in NotReady state?"
"List pods stuck in CrashLoopBackOff in the prod namespace and pull their events"
"Walk me through creating a new AKS cluster with Azure CNI Overlay, 3-node dev pool, Entra ID auth
