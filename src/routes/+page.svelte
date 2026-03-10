<script>
  import TrustBar from "$lib/TrustBar.svelte";
  import ToolHeader from "$lib/ToolHeader.svelte";
  import ToolFooter from "$lib/ToolFooter.svelte";

  let input = $state("");
  let output = $state("");
  let copied = $state(false);
  let detections = $state([]);
  let replaceMode = $state("redact");

  const PATTERNS = [
    {
      name: "Email",
      re: /[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}/g,
      fake: () => "user@example.com",
    },
    {
      name: "SSN",
      re: /\b\d{3}-\d{2}-\d{4}\b/g,
      fake: () => `${rand(100, 899)}-${rand(10, 99)}-${rand(1000, 9999)}`,
    },
    {
      name: "Credit Card",
      re: /\b(?:\d[ -]*?){13,19}\b/g,
      test: (m) => luhnCheck(m.replace(/\D/g, "")),
      fake: () => "4111 1111 1111 1111",
    },
    {
      name: "Phone",
      re: /(?:\+?1[-.\s]?)?(?:\(?\d{3}\)?[-.\s]?)?\d{3}[-.\s]?\d{4}\b/g,
      fake: () => `(555) ${rand(200, 999)}-${rand(1000, 9999)}`,
    },
    {
      name: "IP Address",
      re: /\b(?:\d{1,3}\.){3}\d{1,3}\b/g,
      fake: () =>
        `${rand(10, 192)}.${rand(0, 255)}.${rand(0, 255)}.${rand(1, 254)}`,
    },
    {
      name: "IPv6",
      re: /\b(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}\b/g,
      fake: () => "2001:0db8:85a3:0000:0000:8a2e:0370:7334",
    },
    {
      name: "AWS Key",
      re: /\b(?:AKIA|ABIA|ACCA|ASIA)[0-9A-Z]{16}\b/g,
      fake: () => "AKIAIOSFODNN7EXAMPLE",
    },
    {
      name: "API Key",
      re: /\b(?:sk|pk|api|key|token|secret|bearer)[-_]?[a-zA-Z0-9]{20,}/gi,
      fake: () => "sk_live_EXAMPLE_KEY_REPLACED",
    },
    {
      name: "JWT",
      re: /\beyJ[a-zA-Z0-9_-]{10,}\.eyJ[a-zA-Z0-9_-]{10,}\.[a-zA-Z0-9_-]{10,}/g,
      fake: () => "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0.EXAMPLE",
    },
    {
      name: "Private Key",
      re: /-----BEGIN (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----[\s\S]*?-----END (?:RSA |EC |DSA |OPENSSH )?PRIVATE KEY-----/g,
      fake: () => "[PRIVATE_KEY_REMOVED]",
    },
    {
      name: "Street Address",
      re: /\b\d{1,5}\s(?:[A-Z][a-z]+\s){1,3}(?:St|Ave|Blvd|Dr|Ln|Rd|Way|Ct|Pl|Cir|Ter|Pkwy|Hwy)\.?\b/g,
      fake: () => "123 Example St",
    },
    {
      name: "Zip Code",
      re: /\b\d{5}(?:-\d{4})?\b/g,
      fake: () => `${rand(10000, 99999)}`,
    },
    {
      name: "Date of Birth",
      re: /\b(?:0[1-9]|1[0-2])\/(?:0[1-9]|[12]\d|3[01])\/(?:19|20)\d{2}\b/g,
      fake: () => "01/01/1990",
    },
  ];

  function rand(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }

  function luhnCheck(num) {
    if (num.length < 13 || num.length > 19) return false;
    let sum = 0;
    let alt = false;
    for (let i = num.length - 1; i >= 0; i--) {
      let n = parseInt(num[i], 10);
      if (alt) {
        n *= 2;
        if (n > 9) n -= 9;
      }
      sum += n;
      alt = !alt;
    }
    return sum % 10 === 0;
  }

  function scan() {
    if (input.trim() === "") {
      output = "";
      detections = [];
      return;
    }

    let found = [];

    for (const pattern of PATTERNS) {
      const re = new RegExp(pattern.re.source, pattern.re.flags);
      let match;
      while ((match = re.exec(input)) !== null) {
        const val = match[0];
        if (pattern.test && !pattern.test(val)) continue;
        // Skip if overlapping with existing detection
        const start = match.index;
        const end = start + val.length;
        const overlaps = found.some((f) => start < f.end && end > f.start);
        if (!overlaps) {
          found.push({
            name: pattern.name,
            value: val,
            start,
            end,
            fake: pattern.fake,
          });
        }
      }
    }

    found.sort((a, b) => a.start - b.start);
    detections = found;
    buildOutput();
  }

  function buildOutput() {
    if (detections.length === 0) {
      output = input;
      return;
    }

    let result = "";
    let cursor = 0;

    for (const d of detections) {
      result += input.slice(cursor, d.start);
      if (replaceMode === "redact") {
        result += `[${d.name.toUpperCase()}]`;
      } else {
        result += d.fake();
      }
      cursor = d.end;
    }
    result += input.slice(cursor);
    output = result;
  }

  function handleInput() {
    scan();
  }

  function handleModeChange() {
    buildOutput();
  }

  function copyOutput() {
    if (!output) return;
    navigator.clipboard.writeText(output).then(() => {
      copied = true;
      setTimeout(() => (copied = false), 1500);
    });
  }

  function clear() {
    input = "";
    output = "";
    detections = [];
    copied = false;
  }

  function loadSample() {
    input = `Hi, can you help me debug this? My name is Sarah Johnson and my email is sarah.johnson@company.com. I'm getting an error when calling our API.

Here's the config:
API_KEY=sk_live_4eC39HqLyjWDarjtT1zdp7dc
AWS_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE
Server IP: 192.168.1.105

The user reported it — their info:
Phone: (415) 555-2671
SSN: 329-07-8432
Address: 742 Evergreen Ter, Springfield
Credit card ending in: 4532 8921 0045 6789

Error log attached. Can you tell me what's wrong?`;
    scan();
  }

  function getCategoryColor(name) {
    switch (name) {
      case "Email":
        return "#9b8fde";
      case "SSN":
      case "Credit Card":
        return "#dc2626";
      case "Phone":
        return "#e8628a";
      case "IP Address":
      case "IPv6":
        return "#e8a830";
      case "AWS Key":
      case "API Key":
      case "JWT":
      case "Private Key":
        return "#f2714d";
      case "Street Address":
      case "Zip Code":
        return "#6baf8d";
      case "Date of Birth":
        return "#f59e6c";
      default:
        return "#999";
    }
  }

  function getSeverity(name) {
    const critical = ["SSN", "Credit Card", "Private Key", "AWS Key"];
    const high = ["API Key", "JWT", "Email", "Phone"];
    if (critical.includes(name)) return "critical";
    if (high.includes(name)) return "high";
    return "medium";
  }

  let criticalCount = $derived(
    detections.filter((d) => getSeverity(d.name) === "critical").length,
  );
  let highCount = $derived(
    detections.filter((d) => getSeverity(d.name) === "high").length,
  );

  // Build highlighted HTML for the preview
  let highlightedHtml = $derived.by(() => {
    if (!input || detections.length === 0) return "";
    let html = "";
    let cursor = 0;
    for (const d of detections) {
      html += escapeHtml(input.slice(cursor, d.start));
      const sev = getSeverity(d.name);
      html += `<mark class="hl hl-${sev}" title="${d.name}">${escapeHtml(d.value)}</mark>`;
      cursor = d.end;
    }
    html += escapeHtml(input.slice(cursor));
    return html;
  });

  function escapeHtml(str) {
    return str
      .replace(/&/g, "&amp;")
      .replace(/</g, "&lt;")
      .replace(/>/g, "&gt;")
      .replace(/"/g, "&quot;")
      .replace(/\n/g, "<br>");
  }
</script>

<TrustBar sourceUrl="https://github.com/user/localprompt" />

<div class="page-shell">
  <ToolHeader
    title="Prompt Sanitizer"
    description="Strip personal data before pasting into AI — nothing leaves your browser."
  />

  <div class="tool-area">
    <div class="panel">
      <label class="tool-label" for="input">paste your text</label>
      <textarea
        id="input"
        class="tool-textarea"
        bind:value={input}
        oninput={handleInput}
        placeholder="Paste the text you're about to send to ChatGPT, Claude, Gemini..."
        spellcheck="false"
      ></textarea>
    </div>

    <div class="panel">
      <label class="tool-label" for="output">safe to paste</label>
      <textarea
        id="output"
        class="tool-textarea"
        class:output-valid={detections.length > 0}
        value={output}
        readonly
        placeholder="Sanitized text appears here..."
        spellcheck="false"
      ></textarea>
    </div>
  </div>

  <div class="controls">
    <button class="btn btn-primary" onclick={scan}>Scan</button>
    <button class="btn" onclick={copyOutput} disabled={!output}>
      {copied ? "✓ Copied" : "Copy safe text"}
    </button>
    <button class="btn" onclick={clear}>Clear</button>
    <button class="btn btn-sample" onclick={loadSample}>Sample</button>

    <div class="mode-toggle">
      <button
        class="btn btn-mode"
        class:btn-active={replaceMode === "redact"}
        onclick={() => {
          replaceMode = "redact";
          handleModeChange();
        }}>[REDACT]</button
      >
      <button
        class="btn btn-mode"
        class:btn-active={replaceMode === "fake"}
        onclick={() => {
          replaceMode = "fake";
          handleModeChange();
        }}>Fake data</button
      >
    </div>

    <div class="spacer"></div>

    {#if detections.length > 0}
      {#if criticalCount > 0}
        <span class="pill pill-critical">
          {criticalCount} critical
        </span>
      {/if}
      {#if highCount > 0}
        <span class="pill pill-high">
          {highCount} high
        </span>
      {/if}
      <span class="pill pill-count">
        {detections.length} total found
      </span>
    {:else if input.trim().length > 0}
      <span class="pill pill-ok">
        <span class="pill-dot"></span>
        looks clean
      </span>
    {/if}
  </div>

  {#if detections.length > 0}
    <div class="detections-panel">
      <label class="tool-label">detected</label>
      <div class="preview-box">
        {@html highlightedHtml}
      </div>
      <div class="detection-list">
        {#each detections as d}
          <div class="detection-row">
            <span
              class="detection-badge"
              class:badge-critical={getSeverity(d.name) === "critical"}
              class:badge-high={getSeverity(d.name) === "high"}
              class:badge-medium={getSeverity(d.name) === "medium"}
              >{d.name}</span
            >
            <span class="detection-value">{d.value}</span>
            <span class="detection-arrow">→</span>
            <span class="detection-replaced">
              {#if replaceMode === "redact"}
                [{d.name.toUpperCase()}]
              {:else}
                {d.fake()}
              {/if}
            </span>
          </div>
        {/each}
      </div>
    </div>
  {/if}

  <ToolFooter />
</div>

<style>
  .panel {
    display: flex;
    flex-direction: column;
    gap: 5px;
  }

  .btn-sample {
    font-size: 12px;
    padding: 8px 12px;
    color: var(--text-muted);
  }

  .mode-toggle {
    display: flex;
    gap: 0;
    margin-left: 8px;
  }

  .btn-mode {
    border-radius: 0;
    font-size: 11px;
    font-family: var(--font-mono);
    padding: 6px 12px;
  }

  .btn-mode:first-child {
    border-radius: var(--radius-sm) 0 0 var(--radius-sm);
  }

  .btn-mode:last-child {
    border-radius: 0 var(--radius-sm) var(--radius-sm) 0;
    border-left: none;
  }

  .btn-active {
    background: var(--accent);
    color: white;
    border-color: var(--accent);
  }

  .spacer {
    margin-left: auto;
  }

  .pill-critical {
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 600;
    padding: 4px 10px;
    border-radius: 999px;
    background: #fef2f2;
    color: #dc2626;
    margin-right: 4px;
  }

  .pill-high {
    font-family: var(--font-mono);
    font-size: 12px;
    font-weight: 600;
    padding: 4px 10px;
    border-radius: 999px;
    background: #fff7ed;
    color: #ea580c;
    margin-right: 4px;
  }

  .pill-count {
    font-family: var(--font-mono);
    font-size: 12px;
    padding: 4px 10px;
    border-radius: 999px;
    background: var(--bg-input);
    color: var(--text-muted);
  }

  /* ---- detections panel ---- */
  .detections-panel {
    margin-top: 16px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .preview-box {
    font-family: var(--font-mono);
    font-size: 12px;
    line-height: 1.7;
    padding: 14px;
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    background: var(--bg-surface);
    max-height: 200px;
    overflow-y: auto;
    color: var(--text-secondary);
  }

  .preview-box :global(.hl) {
    padding: 1px 4px;
    border-radius: 3px;
    font-weight: 600;
  }

  .preview-box :global(.hl-critical) {
    background: #fecaca;
    color: #991b1b;
  }

  .preview-box :global(.hl-high) {
    background: #fed7aa;
    color: #9a3412;
  }

  .preview-box :global(.hl-medium) {
    background: #fef08a;
    color: #854d0e;
  }

  .detection-list {
    border: 1px solid var(--border-light);
    border-radius: var(--radius-md);
    background: var(--bg-surface);
    overflow: hidden;
  }

  .detection-row {
    display: flex;
    align-items: center;
    padding: 8px 14px;
    gap: 10px;
    font-family: var(--font-mono);
    font-size: 11px;
    border-bottom: 1px solid var(--border-light);
  }

  .detection-row:last-child {
    border-bottom: none;
  }

  .detection-badge {
    font-size: 9px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
    padding: 2px 8px;
    border-radius: 999px;
    white-space: nowrap;
    flex-shrink: 0;
  }

  .badge-critical {
    background: #fecaca;
    color: #991b1b;
  }

  .badge-high {
    background: #fed7aa;
    color: #9a3412;
  }

  .badge-medium {
    background: #fef08a;
    color: #854d0e;
  }

  .detection-value {
    flex: 1;
    color: #dc2626;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    text-decoration: line-through;
  }

  .detection-arrow {
    color: var(--text-muted);
    flex-shrink: 0;
  }

  .detection-replaced {
    flex: 1;
    color: var(--text-primary);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  button:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  @media (max-width: 640px) {
    .mode-toggle {
      display: none;
    }

    .controls {
      flex-wrap: wrap;
    }

    .detection-replaced {
      display: none;
    }

    .detection-arrow {
      display: none;
    }
  }
</style>
