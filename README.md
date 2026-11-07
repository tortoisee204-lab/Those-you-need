armDelete(del, 'Delete', function () {
        state.products = state.products.filter(function (x) { return x.id !== p.id; });
        delete cart[p.id]; saveCart();
        setDirty(); renderAll(); renderAdminList();
      });
      ul.appendChild(h('li', { class: 'arow' }, [
        thumb(p),
        h('div', { class: 'ainfo' }, [h('strong', { text: p.name }), h('span', { class: 'muted', text: money(p.price) + (p.sample ? ' · sample' : '') })]),
        del
      ]));
    });
    $('rmSamples').hidden = !state.products.some(function (p) { return p.sample; });
  }
  function removeSamples() {
    var btn = $('#rmSamples');
    if (!btn.dataset.armed) {
      btn.dataset.armed = '1'; btn.textContent = 'Tap again to remove samples';
      setTimeout(function () { delete btn.dataset.armed; btn.textContent = 'Remove all sample products'; }, 3500);
      return;
    }
    delete btn.dataset.armed; btn.textContent = 'Remove all sample products';
    state.products.forEach(function (p) { if (p.sample) delete cart[p.id]; });
    state.products = state.products.filter(function (p) { return !p.sample; });
    saveCart(); setDirty(); renderAll(); renderAdminList();
  }
  function readImage(file) {
    return new Promise(function (resolve, reject) {
      var fr = new FileReader();
      fr.onerror = function () { reject(new Error('read')); };
      fr.onload = function () {
        var im = new Image();
        im.onerror = function () { reject(new Error('decode')); };
        im.onload = function () {
          var max = 720, k = Math.min(1, max / Math.max(im.width, im.height));
          var c = document.createElement('canvas');
          c.width = Math.max(1, Math.round(im.width * k)); c.height = Math.max(1, Math.round(im.height * k));
          var x = c.getContext('2d');
          x.fillStyle = '#ffffff'; x.fillRect(0, 0, c.width, c.height);
          x.drawImage(im, 0, 0, c.width, c.height);
          resolve(c.toDataURL('image/jpeg', 0.8));
        };
        im.src = fr.result;

readAsDataURL(file);
    });
  }
  function onPhoto(e) {
    var f = e.target.files && e.target.files[0], pv = $('#pPreview');
    pendingImg = ''; pv.hidden = true;
    if (!f) return;
    readImage(f).then(function (url) { pendingImg = url; pv.src = url; pv.hidden = false; },
      function () { $('#addMsg').textContent = 'That file could not be read as a photo. Try a JPG or PNG.'; });
  }
  function addProduct(e) {
    e.preventDefault();
    var name = $('#pName').value.trim(), price = Number($('#pPrice').value), msg = $('#addMsg');
    if (!name) { msg.textContent = 'Enter a product name.'; return; }
    if (!isFinite(price) || price < 0 || $('#pPrice').value === '') { msg.textContent = 'Enter a price in taka.'; return; }
    var id = 'p' + Date.now().toString(36) + Math.random().toString(36).slice(2, 5);
    state.products.unshift({ id: id, name: name, price: Math.round(price), desc: $('#pDesc').value.trim(), img: pendingImg, sample: false });
    $('#addForm').reset(); pendingImg = ''; $('#pPreview').hidden = true;
    msg.textContent = 'Added. Press Publish changes to show it to customers.';
    setDirty(); renderAll(); renderAdminList();
    $('#pName').focus();
  }

  /* ---------- publish ---------- */
  function buildDoc(s) {
    var json = JSON.stringify(s).replace(/</g, '\\u003c');
    var close = '</' + 'script>';
    return '<!doctype html><html><head><meta charset=utf8><meta name=viewport content="width=device-width,initial-scale=1,viewport-fit=cover"><style>' + RESET + '</style></head><body>' +
      '<title>' + TITLE + '</title>\n' +
      '<meta name="claude:viewport-fit-sides" content="cover">\n' +
      '<link rel="stylesheet" href="' + FONT_HREF + '">\n' +
      '<style id="page-style">' + $('#page-style').textContent + '</style>\n' +
      '<div id="root"></div>\n' +
      '<script type="application/json" id="shop-data">' + json + close + '\n' +
      '<script id="app-code">' + $('#app-code').textContent + close +
      '</body></html>';
  }
  function publish() {
    var st = $('#pubState'), btn = $('#publishBtn');
    if (!artifactNS) { st.textContent = 'Publishing is not available in this view.'; return; }
    if (!state.shop.name || !state.shop.name.trim()) state.shop.name = 'Those you need';
    var html = buildDoc(state);
    if (html.length > 14 * 1024 * 1024) { st.textContent = 'Photos are too large. Delete some products or use smaller photos.'; return; }
    btn.disabled = true; st.textContent = 'Publishing…';
    artifactNS.publish(html).then(function () {
      dirty = false; st.textContent = 'Published';
    }, function (err) {
      var c = err && err.code;
      btn.disabled = false;
      if (c === 'conflict') st.textContent = 'The shop was updated elsewhere. Reloading the latest version.';
      else if (c === 'too_large') st.textContent = 'Photos are too large. Delete some products or use smaller photos.';
      else if (c === 'rate_limited') st.textContent = 'Too many saves. Wait a moment, then publish once more.';
      else if (c === 'not_writer' || c === 'not_granted' || c === 'capability_disabled') { st.textContent = 'This view is read-only, so changes cannot be published.'; btn.hidden = true; }
      else st.textContent = 'Publishing failed. Try again in a moment.';
    });
  }

  /* ---------- boot ---------- */
  function renderAll() {
    $('#shopName').textContent = state.shop.name || 'Those you need';
    renderGrid(); renderBadge(); renderOwnerNote();
  }
  buildShell();
  renderAll();
  renderCart();
  document.addEventListener('visibilitychange', function () {});

  (async function () {
    try {
      if (window.claude && window.claude.use) {
        var u = await window.claude.use('user');
        var a = await window.claude.use('artifact');
        var ce = u ? await u.canEdit() : false;
        if (a && ce) { artifactNS = a; canManage = true; $('#manageBtn').hidden = false; }
      }
    } catch (e) {}
    renderAll();
  })();
})();
</script>

</body></html>
