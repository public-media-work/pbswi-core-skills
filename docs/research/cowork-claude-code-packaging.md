# How one skill source installs in both Cowork and Claude Code

Research for [#4](https://github.com/public-media-work/pbswi-core-skills/issues/4) (map: #3). Researched 2026-10-21 against Anthropic's first-party docs. Help-center pages carry only relative dates ("updated this week"), so re-check anything the rollout depends on.

## Answer in brief

- **Use one format: a Claude Code plugin marketplace.** Cowork, claude.ai chat and Claude Code all read the same plugin folder (`.claude-plugin/plugin.json` plus `skills/<name>/SKILL.md`) listed in a `.claude-plugin/marketplace.json`. Each surface loads the parts it supports and skips the rest. [S1][S2]
- **One install on a claude.ai account reaches both apps.** A plugin added to a member's claude.ai account, by the member or by the org, loads in chat and Cowork. It also syncs into Claude Code as `<name>@synced` for anyone signed in with that account on v2.1.273 or later. [S1][S3]
- **The catch for this repo:** org-wide sync from github.com only accepts a **private or internal** marketplace repo, and pbswi-core-skills is public. A small private marketplace repo can still list plugins from a public github.com repo, so the public repo can stay the one canonical copy. [S4][S5]
- **`.plugin` files and zips work, but have no update path.** Members can upload either one under Customize > Plugins, and Owners can upload a zip in org settings. These are one-off copies: nothing ties them back to the repo. [S1][S5]

## 1. Does Cowork accept the Claude Code plugin/marketplace format?

Yes, and it is the same format.

- "You can install the same plugin folder everywhere you use Claude, but chat, Cowork, and Claude Code each load a different subset of what the folder can contain." [S2]
- A marketplace is "a directory or repository with a `.claude-plugin/marketplace.json` file that lists your plugins and where to fetch each one." The file requires `name`, `owner` and `plugins`, and each entry requires `name` and `source`. [S6]
- Organization sync takes the same repository: "The repository you sync is a plugin marketplace: it has a `.claude-plugin/marketplace.json` file." [S4]
- "Roll out a plugin to your whole organization" says to use the same repository for the claude.ai org route and for Claude Code managed settings. [S7]

**Components loaded on each surface** [S2]:

| Component | Chat | Cowork | Claude Code |
|---|---|---|---|
| Skills (`skills/<name>/SKILL.md`) | Loads | Loads | Loads |
| Commands (`commands/*.md`, legacy) | Loads as a skill | Loads (`/plugin:command`) | Loads |
| Agents | Ignored | Loads | Loads |
| Hooks | Ignored | Loads | Loads |
| Remote MCP (http/sse) | Via Connectors tab | Loads | Loads |
| Local MCP (stdio, `.mcpb`) | Ignored | Only when the Cowork session runs on the user's computer | Loads |
| MCP using `${user_config.*}` | Ignored if in URL | Ignored if no default (Cowork doesn't prompt) | Prompts |
| Top-level `bin/` executables | **Can't be installed** | **Can't be installed** | Loads |
| LSP, output styles, themes, `settings` | Ignored | Ignored | Loads |

**Rule for this repo:** never ship a top-level `bin/` directory. Put scripts in `scripts/` and call them as `${CLAUDE_PLUGIN_ROOT}/scripts/<name>`. claude.ai rejects any plugin that has a `bin/` directory, whether it arrives by sync or by upload. [S4][S8]

**`.plugin` files and zips:** "Add > Upload plugin: upload a plugin you have as a folder on your computer, as a `.zip` or `.plugin` file. Zip either the plugin folder itself or its contents; both work as long as the archive holds one `.claude-plugin/plugin.json`." [S1] This explains what `cowork-plugin-management:create-cowork-plugin` produces: it zips the plugin directory into `<name>.plugin`. That skill's own SKILL.md describes the step; it is a hint, not a primary source.

## 2. Ways a plugin reaches Cowork users: org-wide vs per-user

| Path | Who does it | Plan | Update path | Fit for this repo |
|---|---|---|---|---|
| **Org marketplace synced from GitHub** (Organization settings > Plugins & skills > Add > Sync from GitHub) | Owner / Primary Owner | Team, Enterprise | Auto-sync on push to the marketplace repo's default branch, or manual **Re-sync** | **Best**, but the marketplace repo must be private or internal |
| **Org marketplace by manual zip upload** | Owner | Team, Enterprise | Re-upload (same name overwrites) | Fallback; manual |
| **Personal marketplace** (Customize > Plugins > Add marketplace, `owner/repo`) | Each member | Any plan with Cowork | **Check for updates**, or **Sync automatically** for github.com | Works with the public repo today; each person sets it up |
| **Upload `.zip` / `.plugin`** (Customize > Plugins > Add > Upload plugin) | Each member | Any | None; re-upload | Demos only |
| **Org-provisioned skill** (admin uploads a skill zip) | Owner | Team, Enterprise | Approve a new version, then everyone gets it | Skills only, no plugin wrapper; duplicates the repo |
| **Shared / published plugin or skill** (member shares or publishes to org) | Any member | Team, Enterprise | Owner saves a new version, then recipients get it at next use | Ad hoc |

Sources: [S1][S4][S5][S9][S10].

Details:

- **Org sync, repository rules.** "on github.com and gitlab.com, the marketplace repository must be private or internal." The Claude GitHub App must be installed on the repo, with its Webhooks (Read & Write) permission approved. Turning on automatic sync takes repo-admin access through the Owner's personal GitHub connection. Members don't need repo access: "Organization sync reads the repository through your organization's GitHub or GitLab connection." [S4][S5]
- **Private plugin sources under org sync.** A plugin source may be private only if it is (a) on github.com under the same owner as the marketplace repo, (b) on the org's GitHub Enterprise host with the app installed, or (c) on the same GitLab host or namespace. "Any other plugin source must be a public repository on github.com, gitlab.com, or bitbucket.org, which organization sync fetches without credentials." Accepted source types are `github`, `url`, `git-subdir`, or a `./` relative path. `npm`, `archive` and `command` are not accepted. [S4][S5]
- **Implication.** A private repo such as `public-media-work/pbswi-marketplace` could hold only `marketplace.json`, with entries that point at this public repo through `git-subdir` (`url: public-media-work/pbswi-core-skills`, `path: plugins/<name>`). The public repo stays the single canonical source. See open question Q1 on how updates propagate through this setup.
- **Availability settings.** Per plugin, an Owner chooses Not available, Available to install, Installed by default, or Required. Per-group overrides are **Enterprise only**. "Installed by default" and "Required" plugins also install in Claude Code, and Required ones can't be disabled there. [S5][S1]
- **Personal marketplaces.** "For a marketplace you add yourself, GitHub and GitHub Enterprise repositories work, and so do public GitLab and Bitbucket repositories. To add a private GitHub repository, connect your GitHub account ... and give the Claude GitHub App access." So a member can add the public pbswi-core-skills repo directly. [S1][S9]
- **The platform docs page is out of date.** It still says "Custom Skills are individual to each user... cannot be centrally managed by admins" and "Custom Skills do not sync across surfaces." [S11] The newer help-center and claude.com pages document org provisioning and claude.ai→Claude Code sync [S3][S10], and this account shows that sync working (see Local evidence). Treat [S11] as superseded on this point.

## 3. How installed copies get updates

- **Org sync:** "Organization sync reads the repository's default branch, and members get the new version after the marketplace syncs. It syncs when an Owner selects Re-sync or, with Sync automatically on, when someone pushes to the default branch. Nothing syncs on a schedule. Tags aren't read." [S7] A sync "replaces all plugins in the marketplace with the repo's current state," can take up to 30 minutes, and on failure "members keep the last synced version." [S5] The help-center article is inconsistent on the trigger. One section says direct pushes trigger a sync; another says only PR merges with a version bump do. Test this on the first release. [S5]
- **Members:** "After a new version syncs from the source, you get it on your account automatically, with nothing to accept." [S9] In Cowork, a plugin's skills load "the next time you start" a task. If a member edits a plugin's files locally, Cowork warns before an update overwrites them. [S9][S1]
- **Claude Code, synced copies:** each terminal session start runs one background sync. It downloads new and updated plugins and removes turned-off ones, then shows "Plugins changed. Run /reload-plugins to activate." Sync needs claude.ai sign-in. API keys, `CLAUDE_CODE_OAUTH_TOKEN`, `apiKeyHelper`, bare or safe mode, and `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` all block it. Turn it off with `syncClaudeAiPlugins: false`; standalone skills use `syncClaudeAiSkills`. [S3][S10]
- **Claude Code, CLI marketplace (`/plugin marketplace add public-media-work/pbswi-core-skills`):** auto-update is **off by default** for third-party marketplaces. A user turns it on in `/plugin` > Marketplaces, or an admin sets `autoUpdate: true` in managed settings. Updates arrive only when the computed version changes. Either bump `version` every release, or leave `version` out of both `plugin.json` and the marketplace entry so the commit SHA is the version. Never set it in both places. [S8][S3]
- **Version semantics differ.** For a marketplace hosted on claude.ai, "the version claude.ai records for the plugin is its version, and the manifest's `version` isn't read." [S3]
- **Name conflicts.** If a developer has the same plugin from a CLI marketplace and from claude.ai sync, Claude Code loads the CLI copy and reports the synced copy as not loaded. [S3][S7] Tell colleagues to pick one route.

## 4. Size and format limits

| Limit | Value | Source |
|---|---|---|
| Plugin package (uncompressed), personal | 200 MB, 5,000 files | [S1][S2] |
| Upload plugin (`.zip`/`.plugin`) | 200 MB | [S2]; org upload "valid .zip under 200 MB" [S5] |
| Marketplace repository archive | 512 MB | [S1] |
| Plugins per marketplace | 500 (personal) / 1,000 (org) | [S1] / [S5] |
| Marketplaces a member can add | 25 | [S1] |
| Plugin name | ≤64 chars, lowercase, hyphens | [S5] |
| Org sync timeout | 30 minutes | [S5] |
| Reserved marketplace names | e.g. `claude-plugins-official`, `anthropic-plugins`; also `inline`, `skills-dir`, `synced` | [S5][S3] |
| SKILL.md `name` | ≤64 chars, lowercase/digits/hyphens, no "anthropic"/"claude" | [S11] |
| SKILL.md `description` | ≤1,024 chars [S11]; the help center says **200** for claude.ai uploads [S12] | conflict, so keep descriptions ≤200 |
| Skill zip structure | folder name must match skill name; folder at zip root; `SKILL.md` required | [S12][S10] |
| In-app file preview | 1 MB (larger files still usable at runtime) | [S1] |
| Git LFS | not fetched by Claude Code clones; keep plugin files out of LFS | [S8] |

## 5. What depends on admin-enabled code execution

- **Skills require code execution on every claude.ai surface.** "Skills require code execution to function, so if code execution is disabled, skills will not be available." On Team and Enterprise, an Owner enables **Cloud code execution and file creation** and **Skills** under Organization settings > Plugins & skills > Policy. Team plans have both on by default. [S10][S13]
- **Plugin marketplaces require Cowork and Skills.** "Cowork and Skills must both be enabled for your organization before you can use plugin marketplaces." Turning Skills off also stops skill and plugin sync to Claude Code. [S5]
- **Where scripts run.** Skills run in a VM with bash and a filesystem; bundled scripts execute there, and only their output enters context. [S11] Local Cowork sessions run code "in an isolated virtual machine" on the user's computer. Cloud sessions (beta) run on Anthropic infrastructure; they are on by default for Team and off for Enterprise. [S14]
- **Network egress:** "Cowork respects your organization's current network egress permissions" (Organization settings > Capabilities > Code execution). The egress rules don't cover web fetch, web search or MCP connectors. [S14] Skills on claude.ai "might have full, partial, or no network access" depending on settings. [S11] A skill script that calls an outside API therefore depends on the admin's egress allowlist. Connector-based access, such as the Airtable connector, does not.
- **Package installs:** "Claude and Claude Code can install packages from standard repositories (Python PyPI, JavaScript npm) when loading skills." That again depends on egress. [S12]
- **Pure-instruction skills** (Markdown only, no scripts) still need the Skills and code-execution toggles to load at all. Past that, they have no extra runtime dependency.

## Local evidence (this workstation, not a primary source)

- `~/.claude/skills/synced/<org>/manifest.json` lists several skills with `"source": "plugin"` and a `backingPluginId`. These include the Cowork-adapted `wayfinder`, `grilling` and `domain-modeling`, which were uploaded to claude.ai as plugins and are now synced into Claude Code.
- `~/.claude/plugins/synced/<org>/manifest.json` lists plugins with `marketplaceName`, `installationPreference: "available"` and a four-digit claude.ai `version`. One of them comes from an org-synced marketplace. This confirms the claude.ai→Claude Code synced-plugin path [S3] is already working on this account, and that claude.ai assigns its own version numbers [S3].

## Recommendation for pbswi-core-skills

1. **Lay out this repo as a marketplace.** Add `.claude-plugin/marketplace.json` at the root, with each plugin under `plugins/<plugin>/` (its own `.claude-plugin/plugin.json` and `skills/<skill>/SKILL.md`). Use `./` relative sources. Don't use `bin/`, and keep descriptions ≤200 chars. Validate with `claude plugin validate .`.
2. **Claude Code users today:** `/plugin marketplace add public-media-work/pbswi-core-skills`, then turn on auto-update. Or, better, let the claude.ai route below sync it in.
3. **Cowork / Team, org-wide:** depends on #12.
   - If the repo goes private: an Owner syncs it directly with auto-sync on, sets each plugin to Installed by default or Available, and members get it in chat, Cowork and Claude Code (`@synced`) with no per-person setup.
   - If the repo stays public: create a small private marketplace repo in the same GitHub org whose entries are `git-subdir` sources pointing at this repo. Or vendor the plugin folders into it with CI or a git subtree, as Anthropic suggests for private sources. [S5]
4. **Interim, no admin needed:** each colleague adds this public repo under Customize > Plugins > Add marketplace with Sync automatically on. That gives one install for both Cowork and Claude Code.
5. **Don't distribute `.plugin` files or skill zips** except for one-off trials, because they never update.

## Open questions

- **Q1. Updates through a thin private marketplace.** When an org-synced private marketplace points at a public `git-subdir` source, does a push to the *source* repo reach members? Auto-sync webhooks fire on the marketplace repo only. Does a manual Re-sync re-fetch the source's HEAD, or does it need a `ref`/`sha` or version bump in the marketplace repo? Not documented; test it.
- **Q2. What triggers auto-sync.** The help center contradicts itself (direct push vs PR merge with version bump) [S5], while claude.com says "each time someone pushes to the default branch" [S4][S7]. Confirm empirically.
- **Q3. Description length.** 200 chars on the claude.ai help center vs 1,024 in the platform spec. Unknown whether a plugin's skills (as opposed to a skill-zip upload) get the 200 limit.
- **Q4. Synced Cowork sessions.** Cowork downloads synced plugins "into the session's own environment" [S3]. Unknown whether a Cowork *cloud* session can run hook or script components that assume a local machine.
- **Q5. Team vs Enterprise.** Docs state that group-level targeting is Enterprise-only. No other Team-plan restriction on GitHub org sync was found, but it is worth confirming in this org's admin console.

## Sources

- [S1] Install plugins (Cowork): https://claude.com/docs/cowork/guide/plugins
- [S2] Plugin feature support across platforms: https://claude.com/docs/plugins/platform-support
- [S3] Claude Code, Plugin loading reference (synced plugins, versions, name conflicts): https://code.claude.com/docs/en/plugins/loading
- [S4] Sync your organization's plugins from a repository: https://claude.com/docs/plugins/org-sync
- [S5] Manage plugins for your organization (help center): https://support.claude.com/en/articles/13837433-manage-claude-cowork-plugins-for-your-organization
- [S6] Claude Code, Create a marketplace: https://code.claude.com/docs/en/plugin-marketplaces
- [S7] Roll out a plugin to your whole organization: https://claude.com/docs/plugins/org-rollout
- [S8] Claude Code, Host and maintain a marketplace: https://code.claude.com/docs/en/plugins/host-marketplace
- [S9] Plugins (overview): https://claude.com/docs/plugins/overview
- [S10] Use skills in Claude: https://support.claude.com/en/articles/12512180-use-skills-in-claude
- [S11] Agent Skills overview (platform docs): https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
- [S12] How to create custom skills: https://support.claude.com/en/articles/12512198-how-to-create-custom-skills
- [S13] Provision and manage skills for your organization: https://support.claude.com/en/articles/13119606-provision-and-manage-skills-for-your-organization
- [S14] Use Claude Cowork on Team and Enterprise plans: https://support.claude.com/en/articles/13455879-use-claude-cowork-on-team-and-enterprise-plans

No fetched page contained instructions aimed at AI agents. The claude.com and code.claude.com pages open with a standard "Documentation Index: fetch llms.txt" navigation note. That note is boilerplate, and it was not acted on.
