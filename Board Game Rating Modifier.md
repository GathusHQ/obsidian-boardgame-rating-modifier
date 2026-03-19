# 🎲 Modify a Game’s Rating

```dataviewjs
(async () => {
  try {
    // ── 0) Lightbox viewer setup ─────────────────────────────
    const OVERLAY_ID = 'dvjs-bg-image-overlay';
    if (!document.getElementById(OVERLAY_ID)) {
      const overlay = document.createElement('div');
      overlay.id = OVERLAY_ID;
      Object.assign(overlay.style, {
        position: 'fixed', top: '0', left: '0',
        width: '100vw', height: '100vh',
        background: 'rgba(0,0,0,0.85)',
        display: 'none', alignItems: 'center',
        justifyContent: 'center', zIndex: 9999,
        cursor: 'pointer'
      });
      const img = document.createElement('img');
      Object.assign(img.style, {
        maxWidth: '95%', maxHeight: '95%',
        borderRadius: '4px', pointerEvents: 'none'
      });
      overlay.appendChild(img);
      overlay.addEventListener('click', () => overlay.style.display = 'none');
      document.body.appendChild(overlay);
    }
    const showOverlay = src => {
      const o = document.getElementById(OVERLAY_ID);
      o.querySelector('img').src = src;
      o.style.display = 'flex';
    };

    // ── 1) Inject CSS ─────────────────────────────────────────
    const STYLE_ID = 'dvjs-modify-rating-style';
    document.getElementById(STYLE_ID)?.remove();
    const style = document.createElement('style');
    style.id = STYLE_ID;
    style.textContent = `
      .modify-table { width:100%; max-width:800px; margin:24px auto; border-collapse:collapse; }
      .modify-table td { vertical-align:top; padding:12px; }

      .modify-field { display:flex; align-items:center; margin-bottom:4px; }
      .modify-field label { width:140px; font-weight:500; white-space:nowrap; margin-right:8px; }

      /* Text input (not used now but kept for consistency) */
      .modify-field input {
        flex:1;
        padding:6px 8px;
        border:1px solid var(--background-modifier-border);
        border-radius:4px;
        background:var(--background-primary);
        color:var(--text-normal);
        font-size:0.95em;
      }
      
      .modify-table td:first-child {
	    width: 420px;   /* this adjusts the left cell width */
      }

      /* Dropdowns: fixed width so they match exactly */
      .modify-field select {
        width:320px;
        max-width:320px;
        padding:6px 8px;
        border:1px solid var(--background-modifier-border);
        border-radius:4px;
        background:var(--background-primary);
        color:var(--text-normal);
        font-size:0.95em;
      }

      .modify-image {
        position: relative;
        width: 240px;
      }
      .modify-image img {
        display:block;
        width:100%;
        height:auto;
        border:1px solid var(--background-modifier-border);
        border-radius:8px;
        cursor:pointer;
      }
      .modify-image .title-overlay {
        display:block;
        margin-top:8px;
        color:var(--text-on-primary);
        font-size:0.9em;
        text-align:center;
        cursor:pointer;
        text-decoration:none;
      }
      .modify-image .title-overlay:hover {
        text-decoration:underline;
      }
      .modify-message {
        margin-top:12px;
        font-size:0.95em;
        color:var(--text-accent);
        min-height:1.2em;
      }
    `;
    document.head.appendChild(style);

    // ── 2) Pick a random board game ───────────────────────────
    const pages = dv.pages('"Board Games"').values;
    const rnd = pages[Math.floor(Math.random() * pages.length)];
    const coverUrl = rnd['Cover-Img'];
    const gameTitle = rnd.file.name;
    const filePath = rnd.file.path;

    // ── 3) Build the layout ───────────────────────────────────
    const root = dv.container;
    root.empty();
    const table = root.createEl('table', { cls: 'modify-table' });
    const row = table.insertRow();

    // Left cell: form
    const formCell = row.insertCell();

    // ── Game Title dropdown ───────────────────────────────────
    const titleWrap = formCell.createEl('div', { cls: 'modify-field' });
    titleWrap.createEl('label', { text: 'Game Title' });

    const titleSelect = titleWrap.createEl('select');

    pages
      .map(p => p.file.name)
      .sort((a, b) => a.localeCompare(b))
      .forEach(name => {
        const opt = titleSelect.createEl('option', { text: name, value: name });
        if (name === gameTitle) opt.selected = true;
      });

    // ── Rating dropdown (0–5) ─────────────────────────────────
    const ratingWrap = formCell.createEl('div', { cls: 'modify-field' });
    ratingWrap.createEl('label', { text: 'New Rating (0–5)' });

    const ratingSelect = ratingWrap.createEl('select');
    [0,1,2,3,4,5].forEach(n => {
      ratingSelect.createEl('option', { text: n.toString(), value: n.toString() });
    });

    const hint = formCell.createEl('p');
    hint.innerHTML =
      `Select the game you want to update from the dropdown.<br><br>` +
      `Choose a rating from 0 (worst) to 5 (best).`;

    const btn = formCell.createEl('button', { text: '➕ Update Rating' });
    Object.assign(btn.style, {
      marginTop: '1rem',
      padding: '8px 16px',
      borderRadius: '6px',
      cursor: 'pointer',
      fontSize: '0.9em'
    });

    const msg = formCell.createEl('div', { cls: 'modify-message' });

    // ── 4) Live-update image + link when dropdown changes ─────
    titleSelect.addEventListener('change', () => {
      const selected = titleSelect.value;
      const page = pages.find(p => p.file.name === selected);
      if (!page) return;

      imgEl.src = page["Cover-Img"] || "";
      titleLink.textContent = page.file.name;

      titleLink.onclick = (e) => {
        e.preventDefault();
        app.workspace.openLinkText(page.file.path, '', false);
      };
    });

    // ── 5) Update rating button logic ─────────────────────────
    btn.addEventListener('click', async () => {
      const title = titleSelect.value.trim();
      const rating = ratingSelect.value.trim();
      if (!title || rating === "") {
        msg.textContent = '⚠️ Please select a rating.';
        return;
      }

      const path = `Board Games/${title}.md`;
      const file = app.vault.getAbstractFileByPath(path);
      if (!file) {
        msg.textContent = `❌ File not found: ${path}`;
        return;
      }

      const content = await app.vault.read(file);
      const m = content.match(/^---\n([\s\S]*?)\n---/);
      if (!m) {
        msg.textContent = '❌ No frontmatter block detected.';
        return;
      }

      let frontmatter = m[1].split('\n');
      let updatedFM = [...frontmatter];
      let found = false;

      for (let i = 0; i < updatedFM.length; i++) {
        if (/^Star-Rating:/.test(updatedFM[i])) {
          updatedFM[i] = `Star-Rating: ${rating}`;
          found = true;
          break;
        }
      }

      if (!found) {
        const insertAfter = updatedFM.findIndex(l => /^Play-Time:/.test(l));
        const idx = insertAfter !== -1 ? insertAfter + 1 : updatedFM.length;
        updatedFM.splice(idx, 0, `Star-Rating: ${rating}`);
      }

      const updated = `---\n${updatedFM.join('\n')}\n---` + content.slice(m[0].length);
      await app.vault.modify(file, updated);
      msg.textContent = `Updated “${title}” to rating ${rating}.`;
    });

    // Right cell: image + link
    const imgCell = row.insertCell();
    imgCell.addClass('modify-image');
    const imgEl = imgCell.createEl('img', { attr: { src: coverUrl } });
    const titleLink = imgCell.createEl('a', {
      cls: 'title-overlay',
      text: gameTitle,
      attr: { href: '#' }
    });
    titleLink.addEventListener('click', e => {
      e.preventDefault();
      app.workspace.openLinkText(filePath, '', false);
    });

    ['mouseenter', 'mouseover', 'mousedown', 'click'].forEach(evt =>
      imgEl.addEventListener(evt, e => {
        e.preventDefault();
        e.stopImmediatePropagation();
        if (evt === 'mousedown') showOverlay(imgEl.src);
      }, { capture: true })
    );

  } catch (e) {
    dv.paragraph('Error in modify-rating snippet:');
    dv.codeblock(e.stack || e.toString(), 'javascript');
  }
})();

```
