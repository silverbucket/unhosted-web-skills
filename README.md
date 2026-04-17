# Unhosted Web Skills

Agent skills for building [unhosted web applications](https://unhosted.org) -- browser-first apps that connect directly to decentralized protocols and services.

## Available Skills

| Skill | Description |
|-------|-------------|
| [sockethub](skills/sockethub) | Multi-protocol messaging gateway bridging web apps to IRC, XMPP, and RSS/Atom via ActivityStreams |
| [remotestorage](skills/remotestorage) | Build offline-first, unhosted web apps with user-owned data storage, automatic sync, and zero backend infrastructure |
| [webfinger](skills/webfinger) | Discover user services from email-like addresses via WebFinger (acct: URIs, JRD responses, remoteStorage + ActivityPub + OpenID Connect discovery) |

## Installation

### Via skills.sh CLI

```bash
npx skills add silverbucket/unhosted-web-skills --skill sockethub
```

### Via Claude Code

```
/plugin marketplace add silverbucket/unhosted-web-skills
/plugin install sockethub@unhosted-web-skills
```
