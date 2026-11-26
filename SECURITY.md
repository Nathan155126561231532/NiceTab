# Security Policy

## Supported Versions

Use this section to tell people about which versions of your project are
currently being supported with security updates.

| Version | Supported          |
| ------- | ------------------ |
| 5.1.x   | :white_check_mark: |
| 5.0.x   | :x:                |
| 4.0.x   | :white_check_mark: |
| < 4.0   | :x:                |

## Reporting a Vulnerability

Unescaped tab titles and group names injected into HTML (dangerouslySetInnerHTML)
Affected: NiceTab v3.0.8 (commit d9851f6, 2026-11-22), likely earlier versions too Type: HTML injection (CWE-79), in extension pages Severity (suggested): Low to Medium. Script execution is blocked by the extension CSP, but injected markup can leak the user's IP address and change what the confirmation dialog shows.

简要说明：删除确认弹窗中，标签页标题（由网页控制）和标签组/分类名称未经 HTML 转义，直接通过 dangerouslySetInnerHTML 渲染。扩展的 CSP 阻止了脚本执行，但恶意网页仍可注入 HTML（例如加载外部图片以获取用户 IP，或伪造弹窗内容）。建议在插入前对这些值进行 HTML 转义。

Summary
Several delete-confirmation dialogs build an HTML string by concatenating untrusted values (tab titles, tab group names, category names) into a react-intl message, then render it with dangerouslySetInnerHTML. Nothing HTML-escapes these values. intl.formatMessage does not escape interpolated values, and sanitizeContent() only strips emoji, 4-byte characters and control characters.

A tab's title is fully controlled by the web page (<title> / document.title). A malicious page can therefore put markup in its title. That markup is parsed inside the extension's own pages when the user saves the tab to NiceTab and later clicks "remove" on it.

Affected code
File	Line	Untrusted value
entrypoints/options/home/TabListItem.tsx	132-146	tab.title
entrypoints/nice-newtab/App.tsx	~139-146	tab.title
entrypoints/options/home/TabGroup.tsx	165-173, 701	groupName
entrypoints/options/home/TabGroupRecycle.tsx	~78, 295	groupName
entrypoints/options/home/TreeNodeActionModals.tsx	~87, 115	node.title
Example (TabListItem.tsx):

const removeDesc = $fmt({
  id: 'home.removeDesc',
  values: {
    type: `${$fmt('home.tab')}${` <div ...>
      [<strong ... class="ellipsis">
        ${tab.title}</strong>
      ]</div>
    `}`,
  },
});
// ...
content: <div dangerouslySetInnerHTML={{ __html: removeDesc }}></div>,
Group and category names can be set by the user. They can also arrive from an imported file (OneTab, Toby, SessionBuddy or HTML bookmarks; note that importExport.ts runs unescapeHtml() on imported titles) or from a synced or shared Gist/WebDAV file.

Impact
The extension-page CSP (script-src 'self' 'wasm-unsafe-eval'; object-src 'self'; on Chromium, script-src 'self'; object-src 'self'; on Firefox) blocks inline scripts and event handlers. As far as I can tell, this prevents JavaScript execution. Payloads such as <img src=x onerror=...> should not run.

The CSP has no default-src or img-src, however, so an attacker can still:

Leak information over the network. A title such as <img src="https://attacker.example/p.gif"> makes the extension page request an attacker-controlled URL when the dialog opens. This reveals the user's IP address, the fact that the page was saved in NiceTab, and when the user tried to delete it.
Spoof the dialog content. Arbitrary markup and inline style (style attributes are not blocked by this CSP) can change the confirmation text or overlay misleading content and links inside a trusted extension page.
The attack needs user interaction: the user must save the malicious tab and then open its delete confirmation, or import a crafted file.

Steps to reproduce
Serve a page with this title (HTML-encoded in the source, so the decoded title contains raw markup):
<title>&lt;img src="https://attacker.example/p.gif"&gt;&lt;b style="color:red;font-size:30px"&gt;INJECTED&lt;/b&gt;</title>
Open it and send the tab to NiceTab.
In the NiceTab list page, click the remove icon on that tab.
Observe: the confirmation dialog renders a red "INJECTED", and a request to attacker.example shows up in the DevTools Network panel of the extension page.
Note: I found this by code review and have not run it against a built copy of the extension. The steps follow from the code paths above.

Suggested fix
Escape untrusted values before they go into HTML strings, or better, stop building HTML strings and render with React elements:

// Option A: escape
const esc = (s = '') =>
  s.replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
   .replace(/"/g, '&quot;').replace(/'/g, '&#39;');
// ... ${esc(tab.title)} ... ${esc(groupName)} ... ${esc(node?.title)}

// Option B (preferred): react-intl rich text, no dangerouslySetInnerHTML
$fmt({ id: 'home.removeDesc', values: { type: <strong>[{tab.title}]</strong> } })
As defense in depth, you could tighten the extension-page CSP, for example by adding default-src 'self'; img-src 'self' data: blob: https://t3.gstatic.com https://icons.duckduckgo.com https://services.bonjourr.fr (adjust to what the favicon logic needs). Remote image loads in extension pages would then be blocked.

Additional note (lower priority)
The Gist access token and the WebDAV password are kept in plain text in extension storage. This is common for extensions. It may still be worth mentioning in the docs that users should use a token limited to the gist scope and a dedicated WebDAV password.

Thank you for maintaining NiceTab. I am happy to help verify a fix.
