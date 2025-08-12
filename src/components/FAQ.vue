<template>
    <section id="faq" class="faq theme-dark">
        <div class="container">
          <div class="card">
            <div class="card-header">
              <h2>Frequently Asked Questions</h2>
            </div>
            <div class="card-body">
        <details>
            <summary>
                What risks are involved?
            </summary>
            <p>
                Stremio developers don't recommend fiddling with Cinemeta, use this at your own risk.
            </p>
        </details>
        <details>
            <summary>
                Are my credentials safe?
            </summary>
            <p>
                Yes. We only use your credentials to authenticate with the Stremio API. We do not store them in any way; as soon as you refresh the page they are gone.
            </p>
            <p>
                If you're still in doubt, we highly encourage you to go read the source code. You'll see there's no hidden funny business going on.
            </p>
        </details>
            </div>
          </div>
        </div>
    </section>
</template>

<style scoped>
.faq.theme-dark {
  --panel: #252526;
  --panel-2: #2d2d30;
  --border: #3c3c3c;
  --text: #d4d4d4;
  --muted: #9da0a6;
}
.faq.theme-dark { color: var(--text); padding: 8px 16px 16px; }
.container { max-width: 960px; margin: 0 auto; }
.card { background: var(--panel); border: 1px solid var(--border); border-radius: 10px; overflow: hidden; }
.card-header { padding: 14px 16px; border-bottom: 1px solid var(--border); }
.card-body { padding: 8px 16px 16px; }
.card-body details { background: var(--panel-2); border: 1px solid var(--border); border-radius: 8px; padding: 12px 14px; margin-top: 10px; }
.card-body summary { cursor: pointer; font-weight: 600; }
.card-body a { color: #67b8ff; text-decoration: none; }
.card-body a:hover { text-decoration: underline; }
</style>
