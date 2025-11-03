# public-hc-bundle.js-
// public/hc-bundle.js
// Healing Cottage site pack — remote, self-contained
// Why: Update here in GitHub; Square stays untouched. Toggle logs: window.hcDebug = true
(() => {
  const W = window, D = document;

  // --- Config (edit here later)
  const CFG = (W.HEALING_COTTAGE_CFG ||= {
    freeShippingThreshold: 45,
    popupDelaySec: 5,
    popupCode: 'COVEN10'
  });

  // --- Helpers
  const $ = (s,r=D)=>r.querySelector(s);
  const $$ = (s,r=D)=>Array.from(r.querySelectorAll(s));
  const txt = n => (n && (n.textContent||'').trim()) || '';
  const money = n => '$' + (Math.max(0, Number(n)||0)).toFixed(2);
  const has$ = s => /\$?\d{1,3}(,\d{3})*(\.\d{2})?\b/.test((s||'').replace(/\s/g,''));
  const parse$ = s => { const m=(s||'').replace(/[^\d.,]/g,'').replace(/,/g,'').match(/(\d+(\.\d{2})?)/); return m?parseFloat(m[1]):NaN; };
  const log = (...a)=>{ if(W.hcDebug) console.log('[HC]', ...a); };

  // --- Inject CSS + UI (no HTML needed in Square)
  function ensureStyleAndUI(){
    if(!$('#hc-style')){
      const css = `
      .hc-progress{margin:14px 0;background:#fff;border:1px solid #0b0b14;border-radius:999px}
      .hc-progress .bar{height:12px;border-radius:999px;background:#0b0b14;width:0%}
      .hc-label{font:inherit;margin-top:6px;color:#111}
      .hc-popup{position:fixed;right:16px;bottom:16px;max-width:320px;background:#fff;border:1px solid #0b0b14;border-radius:14px;padding:16px;box-shadow:0 12px 30px rgba(0,0,0,.2);z-index:9999;display:none}
      .hc-btn{display:inline-block;padding:.6rem .9rem;border-radius:8px;border:1px solid #0b0b14;background:#fff;cursor:pointer}
      .hc-btn.primary{background:#0b0b14;color:#fff}
      `;
      const s = D.createElement('style'); s.id='hc-style'; s.textContent = css; D.head.appendChild(s);
    }
    if(!$('#hc-progress-wrap')){
      const wrap = D.createElement('div');
      wrap.id = 'hc-progress-wrap';
      wrap.setAttribute('aria-live','polite');
      wrap.innerHTML = `
        <div class="hc-progress" aria-hidden="true"><div class="bar" style="width:0%"></div></div>
        <div class="hc-label">You’re ${money(CFG.freeShippingThreshold)} away from free shipping.</div>`;
      (D.querySelector('main, #site-wrapper, #sqs-content, body')||D.body).prepend(wrap);
    }
    if(!$('.hc-popup')){
      const pop = D.createElement('div');
      pop.className='hc-popup';
      pop.setAttribute('role','dialog');
      pop.setAttribute('aria-modal','true');
      pop.innerHTML = `
        <h3 style="margin:.2rem 0">Unlock Monthly Ritual Wisdom</h3>
        <p>10% off your first order</p>
        <div style="display:flex;gap:8px;margin-top:10px">
          <button class="hc-btn primary" id="hc-pop-accept">Reveal code</button>
          <button class="hc-btn" id="hc-pop-close" aria-label="Close">Close</button>
        </div>`;
      D.body.appendChild(pop);
    }
  }

  // --- Subtotal discovery (theme-agnostic)
  let subNode = null;
  function findSubtotalNode(){
    const probes = [
      '[data-test="cart-subtotal"]','[data-test="order-summary-subtotal"]','[data-test="checkout-subtotal"]',
      '.cart-summary-subtotal','.order-summary__subtotal','.order-summary .subtotal',
      '[class*="subtotal"] span,[class*="Subtotal"] span','[class*="Money"],[class*="money"]'
    ];
    for(const q of probes){ for(const el of $$(q)){ if(has$(txt(el))) return el; } }
    const labels = $$('*').filter(n=>/^subtotal$/i.test(txt(n)));
    for(const lab of labels){
      const near = [lab.nextElementSibling, ...$$('*', lab.parentElement||D)].find(n=>n && has$(txt(n)));
      if(near) return near;
    }
    return null;
  }
  function getSubtotal(){
    if(subNode && D.contains(subNode)){
      const v = parse$(txt(subNode)); if(!isNaN(v)) return v;
    }
    const n = findSubtotalNode(); if(n){ subNode=n; const v = parse$(txt(n)); if(!isNaN(v)) return v; }
    try{ return JSON.parse(sessionStorage.getItem('hcSubtotal')||'0')||0; }catch(_){ return 0; }
  }
  function setSessionSubtotal(v){ try{ sessionStorage.setItem('hcSubtotal', JSON.stringify(Math.max(0, v||0))); }catch(_){} }

  // --- Progress bar
  function updateProgress(forceVal){
    const wrap = $('#hc-progress-wrap'); if(!wrap) return;
    const need = Math.max(0, CFG.freeShippingThreshold - (Number(forceVal ?? getSubtotal())||0));
    const pct = Math.min(100, Math.round(100*(1-need/CFG.freeShippingThreshold)));
    wrap.querySelector('.bar')?.style.setProperty('width', pct + '%');
    const label = wrap.querySelector('.hc-label');
    if(label) label.textContent = need>0 ? `You’re ${money(need)} away from free shipping.` : 'Free shipping unlocked!';
    log('progress', {subtotal:getSubtotal(), need, pct});
  }

  // --- Add-to-cart detection (no theme selectors)
  let atcSel = null, priceSel = null;
  function resolveATCAndPrice(ctx=D){
    const btns = $$('button, a', ctx).filter(b=>{
      const t = (b.getAttribute('aria-label')||txt(b)).toLowerCase();
      return /add\s*to\s*(cart|bag)|add item|add to order/.test(t) ||
             /add-to-cart/i.test(b.getAttribute('data-action')||'');
    });
    if(btns[0]) atcSel = guessSel(btns[0]);

    const priceProbes = ['[data-test="price"]','[data-test="item-price"]','.product-price','[class*="Price"]','[class*="price"]','[data-price]'];
    for(const q of priceProbes){ const el = $(q, ctx); if(el && has$(txt(el))) { priceSel = q; break; } }
    if(!priceSel && btns[0]){
      let p = btns[0].parentElement;
      for(let i=0;i<5 && p;i++,p=p.parentElement){
        const moneyEl = $$('*', p).find(e=>has$(txt(e)));
        if(moneyEl){ priceSel = guessSel(moneyEl); break; }
      }
    }
  }
  function guessSel(el){
    if(!el) return null;
    if(el.id && /^[A-Za-z][\w\-:.]{3,}$/.test(el.id)) return '#' + el.id;
    const cls = (el.className||'').toString().split(/\s+/).filter(Boolean).slice(0,3).map(c=>'.'+c.replace(/([^\w-])/g,'\\$1')).join('');
    return (el.tagName.toLowerCase() + (cls||''));
  }
  function hookATC(){
    D.addEventListener('click', e=>{
      const node = e.target.closest(atcSel || 'button, a'); if(!node) return;
      const t = (node.getAttribute('aria-label')||txt(node)).toLowerCase();
      if(!(/add\s*to\s*(cart|bag)|add item|add to order/.test(t) || /add-to-cart/i.test(node.getAttribute('data-action')||''))) return;
      const ctx = node.closest('[data-item],[data-product],form,section,article') || D;
      const priceEl = priceSel ? $(priceSel, ctx) : $$('[data-test="price"],[data-test="item-price"],.product-price,[class*="Price"],[class*="price"],[data-price]', ctx).find(n=>has$(txt(n)));
      const delta = parse$(txt(priceEl)) || 0;
      const newSub = (getSubtotal()||0) + (delta||0);
      setSessionSubtotal(newSub);
      setTimeout(()=>updateProgress(newSub), 600); // why: wait for cart UI to update
    }, {capture:true});
  }

  // --- Popup
  function initPopup(){
    const el = $('.hc-popup'); if(!el) return;
    const key = 'hcPopClosed';
    if(localStorage.getItem(key)==='1') return;
    setTimeout(()=>{ el.style.display='block'; }, (CFG.popupDelaySec||5)*1000);
    $('#hc-pop-close')?.addEventListener('click',()=>{ el.style.display='none'; localStorage.setItem(key,'1'); });
    $('#hc-pop-accept')?.addEventListener('click',()=>{ alert('Use code: ' + (CFG.popupCode||'')); el.style.display='none'; localStorage.setItem(key,'1'); });
  }

  // --- Boot
  function boot(){
    ensureStyleAndUI();
    subNode = findSubtotalNode();
    resolveATCAndPrice();
    hookATC();
    initPopup();
    updateProgress();

    const mo = new MutationObserver(()=>{ subNode = subNode && D.contains(subNode) ? subNode : findSubtotalNode(); updateProgress(); });
    mo.observe(D.documentElement, {subtree:true, childList:true, characterData:true});

    setInterval(()=>{ updateProgress(); }, 4000);
    W.addEventListener('storage', (e)=>{ if(e.key==='hcSubtotal') updateProgress(); });

    log('HC bundle loaded', {CFG});
  }

  if (D.readyState === 'loading') D.addEventListener('DOMContentLoaded', boot);
  else boot();
})();
