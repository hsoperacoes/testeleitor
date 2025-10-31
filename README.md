<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1"/>
<title>HS Central — Recebimento de Nota Fiscal</title>
<meta name="color-scheme" content="dark light"/>
<style>
  :root{
    --bg:#0b1220;--card:#121a2b;--txt:#e9eefc;--muted:#8aa0bf;--brand:#43b0ff;
    --ok:#21c55d;--warn:#f59e0b;--err:#ef4444;
    --radius:18px;--shadow:0 12px 36px rgba(17,25,40,.28);--ring:0 0 0 3px rgba(67,176,255,.18);
  }
  *{box-sizing:border-box}
  body{margin:0;font-family:Inter,system-ui,-apple-system,Segoe UI,Roboto,Arial;background:#0b1220;color:var(--txt)}
  .wrap{max-width:980px;margin:24px auto;padding:0 16px}
  header{display:flex;align-items:center;justify-content:space-between;margin-bottom:14px}
  h1{font-size:20px;margin:0}
  .card{background:var(--card);border:1px solid rgba(255,255,255,.08);border-radius:var(--radius);box-shadow:var(--shadow);padding:16px}
  label{display:block;font-size:13px;color:var(--muted);margin-bottom:6px}
  input,select,button{width:100%;padding:12px 14px;border-radius:12px;border:1px solid rgba(255,255,255,.12);background:#0f192b;color:var(--txt)}
  .row{display:grid;grid-template-columns:repeat(12,1fr);gap:12px}
  .col-4{grid-column:span 4}
  .col-6{grid-column:span 6}
  .col-8{grid-column:span 8}
  .col-12{grid-column:span 12}
  .btn{cursor:pointer;background:linear-gradient(180deg,#46b7ff 0%,#2f80ff 100%);border:none;color:#041022;font-weight:700}
  .ghost{background:#0f192b;border:1px solid rgba(255,255,255,.12)}
  .muted{color:var(--muted);font-size:12px}
  .hint{font-size:12px;margin-top:4px}
  .ok{color:#0b1b12;border-color:rgba(33,197,93,.45);background:rgba(33,197,93,.2)}
  .warn{color:#201402;border-color:rgba(245,158,11,.45);background:rgba(245,158,11,.18)}
  .actions{display:flex;gap:10px;flex-wrap:wrap}
</style>
</head>
<body>
  <div class="wrap">
    <header>
      <h1>Recebimento de Nota Fiscal</h1>
      <a class="ghost" href="./" style="text-decoration:none;display:inline-block;padding:10px 14px;border-radius:12px">Voltar à Home</a>
    </header>

    <div class="card" role="form" aria-labelledby="titulo-receb">
      <h2 id="titulo-receb" style="font-size:16px;margin:0 0 12px">Dados do Recebimento</h2>

      <div class="row">
        <div class="col-4">
          <label for="filial">Filial</label>
          <select id="filial" name="filial" required>
            <option value="">Selecione uma filial</option>
            <option>ARTUR</option>
            <option>FLORIANO</option>
            <option>JOTA</option>
            <option>MODA</option>
            <option>PONTO</option>
          </select>
        </div>

        <div class="col-4">
          <label for="dataReceb">Data de Recebimento</label>
          <input id="dataReceb" name="dataReceb" type="date" required/>
        </div>

        <div class="col-8">
          <label for="chaveNFe">Chave de Acesso (44 dígitos)</label>
          <div class="actions">
            <input id="chaveNFe" name="chaveNFe" type="text" inputmode="numeric" pattern="\\d{44}" placeholder="Somente números" required style="flex:1"/>
            <button type="button" id="btnScan" class="btn" style="flex:0 0 auto">Ler Código de Barras</button>
          </div>
          <div class="hint muted">Dica: o leitor fecha sozinho quando captura a chave.</div>
        </div>

        <div class="col-12" style="margin-top:8px">
          <label for="obs">Observações (opcional)</label>
          <input id="obs" name="obs" type="text" placeholder="Alguma informação adicional?"/>
        </div>

        <div class="col-12" style="margin-top:12px;display:flex;gap:10px;flex-wrap:wrap">
          <button id="btnEnviar" class="btn" type="button">Registrar Recebimento</button>
          <button id="btnLimpar" class="ghost" type="button">Limpar</button>
          <span id="statusEnvio" class="muted" aria-live="polite"></span>
        </div>
      </div>
    </div>

    <p class="muted" style="margin-top:10px">HS Operações © 2025</p>
  </div>

  <!-- ========= Leitor de Código de Barras (sem licença) ========= -->
  <script type="module">
  // HSScanner — igual ao que te enviei, adaptado aqui como "drop-in"
  let ZXING=null,running=false,rafId=0,useDetector=('BarcodeDetector'in window),detector=null,stream=null,track=null,imageCapture=null,lastValue=null,stableCount=0;
  let modal=null,video=null,torchBtn=null,statusEl=null,codeEl=null,formatsSel=null;

  const overlayHTML=`
    <div id="hs-modal" style="position:fixed;inset:0;display:grid;place-items:center;background:rgba(0,0,0,.65);z-index:999999">
      <div style="width:min(940px,92vw);background:#0b1220;border:1px solid rgba(255,255,255,.08);border-radius:18px;box-shadow:0 12px 36px rgba(17,25,40,.3);padding:12px;color:#e9eefc;font-family:Inter,system-ui,-apple-system,Segoe UI,Roboto,Arial">
        <div style="display:flex;align-items:center;gap:8px;justify-content:space-between">
          <h3 style="margin:4px 0;font-size:16px">Leitor de Código de Barras — HS</h3>
          <div style="display:flex;gap:6px;align-items:center">
            <button id="hs-torch" style="padding:8px 10px;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:#0f192b;color:#e9eefc">Lanterna</button>
            <button id="hs-close" style="padding:8px 12px;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:#1f2a44;color:#e9eefc">Fechar</button>
          </div>
        </div>

        <div style="display:flex;gap:10px;align-items:center;flex-wrap:wrap;margin:8px 0 6px">
          <div style="font-size:12px;color:#8aa0bf">Formatos</div>
          <select id="hs-formats" multiple size="3" style="min-width:200px;padding:6px 8px;border-radius:10px;border:1px solid rgba(255,255,255,.12);background:#0f192b;color:#e9eefc">
            <option value="qr_code">QR Code</option>
            <option value="ean_13" selected>EAN-13</option>
            <option value="ean_8">EAN-8</option>
            <option value="code_128" selected>Code 128</option>
            <option value="code_39">Code 39</option>
            <option value="upc_a">UPC-A</option>
            <option value="upc_e">UPC-E</option>
            <option value="itf">ITF</option>
            <option value="pdf417">PDF417</option>
            <option value="data_matrix">Data Matrix</option>
            <option value="aztec">Aztec</option>
            <option value="codabar">Codabar</option>
          </select>
          <span id="hs-status" style="margin-left:auto;font-size:12px;background:#0f1a31;border:1px solid rgba(255,255,255,.12);padding:4px 8px;border-radius:8px">Iniciando…</span>
        </div>

        <div id="hs-stage" style="position:relative;overflow:hidden;border-radius:16px;aspect-ratio:16/9;background:#000">
          <video id="hs-video" playsinline style="width:100%;height:100%;object-fit:cover"></video>
          <div style="pointer-events:none;position:absolute;inset:0;display:grid;place-items:center">
            <div style="width:min(68%,520px);aspect-ratio:1.8/1;border:3px solid rgba(255,255,255,.9);border-radius:14px;box-shadow:0 0 0 9999px rgba(0,0,0,.50) inset;outline:2px dashed rgba(255,255,255,.25);outline-offset:8px;animation:hs-glow 1.6s ease-in-out infinite"></div>
            <div style="position:absolute;left:50%;top:50%;width:60%;height:2px;background:rgba(67,176,255,.9);transform:translate(-50%,-50%);box-shadow:0 0 10px rgba(67,176,255,.8)"></div>
          </div>
        </div>

        <div style="margin-top:8px;font-size:13px">
          Último código: <span id="hs-code" style="display:inline-block;padding:.3rem .6rem;border-radius:8px;background:#0f1a31;border:1px solid rgba(255,255,255,.12);font-family:ui-monospace,monospace">—</span>
          <div style="color:#8aa0bf;font-size:12px;margin-top:4px">Dica: aproxime o código da moldura; mantenha o enquadramento estável.</div>
        </div>
      </div>
    </div>

    <style>@keyframes hs-glow{50%{border-color:rgba(255,255,255,.65)}}</style>
  `;

  const canvas=document.createElement('canvas');
  const ctx=canvas.getContext('2d',{willReadFrequently:true});

  function beep(){try{const a=new (window.AudioContext||window.webkitAudioContext)();const o=a.createOscillator();const g=a.createGain();o.type='sine';o.frequency.value=880;o.connect(g);g.connect(a.destination);o.start();g.gain.exponentialRampToValueAtTime(0.0001,a.currentTime+0.18);setTimeout(()=>{o.stop();a.close();},200);}catch{}}
  function vib(){if(navigator.vibrate) navigator.vibrate(80);}
  function selectedFormats(){return Array.from(formatsSel.selectedOptions).map(o=>o.value);}
  function setStatus(t,ok=false){statusEl.textContent=t;statusEl.style.background=ok?'rgba(33,197,93,.2)':'#0f1a31';statusEl.style.borderColor=ok?'rgba(33,197,93,.45)':'rgba(255,255,255,.12)';}
  function mapFormatsToZXing(list,F){const m={'qr_code':F.QR_CODE,'ean_13':F.EAN_13,'ean_8':F.EAN_8,'code_128':F.CODE_128,'code_39':F.CODE_39,'upc_a':F.UPC_A,'upc_e':F.UPC_E,'itf':F.ITF,'pdf417':F.PDF_417,'data_matrix':F.DATA_MATRIX,'aztec':F.AZTEC,'codabar':F.CODABAR};const out=[];for(const f of list){if(m[f]) out.push(m[f]);}return out.length?out:[F.EAN_13,F.CODE_128,F.QR_CODE];}
  async function ensureZXing(){ if(!ZXING){ ZXING=await import('https://unpkg.com/@zxing/library@0.20.0/esm/index.js'); } }

  function fillChaveIfPossible(value){
    const candidates=Array.from(document.querySelectorAll('input,textarea'))
      .filter(el=>{const s=((el.id||'')+(el.name||'')+(el.placeholder||'')).toLowerCase();return /\bchave\b/.test(s)||/\bchave[-_ ]?nfe\b/.test(s);});
    if(candidates.length){const el=candidates[0];el.value=value;el.dispatchEvent(new Event('input',{bubbles:true}));el.dispatchEvent(new Event('change',{bubbles:true}));}
  }
  function publish(value,format){
    codeEl.textContent=`${value} (${format})`;
    setStatus('Capturado',true);beep();vib();
    window.dispatchEvent(new CustomEvent('hs:barcode',{detail:{value,format}}));
    try{window.onHSBarcode?.(value,format);}catch{}
    fillChaveIfPossible(value);
    closeScanner();
  }

  async function loop(){
    if(!running) return;
    const vw=video.videoWidth,vh=video.videoHeight;
    if(!vw||!vh){rafId=requestAnimationFrame(loop);return;}
    const roiW=Math.floor(vw*0.68),roiH=Math.floor(roiW/1.8),sx=Math.floor((vw-roiW)/2),sy=Math.floor((vh-roiH)/2);
    canvas.width=roiW;canvas.height=roiH;ctx.drawImage(video,sx,sy,roiW,roiH,0,0,roiW,roiH);
    let value=null,format='desconhecido';
    try{
      if(useDetector&&detector){
        const bmp=await createImageBitmap(canvas);const codes=await detector.detect(bmp);bmp.close?.();
        if(codes?.length){const c=codes[0];value=c.rawValue||c.data||null;format=c.format||'detector';}
      }else if(ZXING){
        if(!ZXING.reader){ZXING.reader=new ZXING.BrowserMultiFormatReader();const hints=new ZXING.Hints();const F=ZXING.BarcodeFormat;const sel=mapFormatsToZXing(selectedFormats(),F);hints.set(ZXING.DecodeHintType.POSSIBLE_FORMATS,sel);ZXING.reader.setHints(hints);}
        const lum=ZXING.HTMLCanvasElementLuminanceSourceFactory.createFromCanvas(canvas);
        const bmp=new ZXING.BinaryBitmap(new ZXING.HybridBinarizer(lum));
        const res=ZXING.reader.decode(bmp);if(res){value=res.getText();format=res.getBarcodeFormat().toString();}
      }
    }catch(_){}

    if(value){
      if(value===lastValue) stableCount++; else {lastValue=value;stableCount=1;}
      setStatus(`Lendo… (${stableCount}/2)`); if(stableCount>=2){publish(value,format);return;}
    }else{ setStatus('Procurando código…'); lastValue=null; stableCount=0; }
    rafId=requestAnimationFrame(loop);
  }

  async function start(){
    if(running) return;
    const cts={audio:false,video:{facingMode:'environment',width:{ideal:1280},height:{ideal:720}}};
    stream=await navigator.mediaDevices.getUserMedia(cts);
    video.srcObject=stream; await video.play();
    track=stream.getVideoTracks()[0];
    try{
      imageCapture=new ImageCapture(track);
      const caps=track.getCapabilities?.()||{}; if('torch'in caps){torchBtn.disabled=false;torchBtn.onclick=async()=>{const s=track.getSettings?.()||{};await track.applyConstraints({advanced:[{torch:!s.torch}]});};}
      else {torchBtn.disabled=true;}
    }catch{torchBtn.disabled=true;}
    if(useDetector){try{detector=new BarcodeDetector({formats:selectedFormats()});}catch{useDetector=false;}}
    if(!useDetector){await ensureZXing();}
    running=true; setStatus('Procurando código…'); loop();
  }
  function stop(){
    running=false; if(rafId) cancelAnimationFrame(rafId);
    if(stream){stream.getTracks().forEach(t=>t.stop());}
    stream=null; track=null; imageCapture=null; video.srcObject=null; setStatus('Parado');
  }

  // API pública
  window.openScanner=async function openScanner(){
    if(!modal){
      document.body.insertAdjacentHTML('beforeend',overlayHTML);
      modal=document.getElementById('hs-modal'); video=document.getElementById('hs-video');
      torchBtn=document.getElementById('hs-torch'); statusEl=document.getElementById('hs-status');
      codeEl=document.getElementById('hs-code'); formatsSel=document.getElementById('hs-formats');
      document.getElementById('hs-close').addEventListener('click',closeScanner);
      formatsSel.addEventListener('change',()=>{ if(useDetector){ try{detector=new BarcodeDetector({formats:selectedFormats()});}catch{} } else if(ZXING){ ZXING.reader=null; }});
    }
    modal.style.display='grid';
    try{ await navigator.mediaDevices.getUserMedia({video:true}); }catch{}
    await start();
  };
  window.closeScanner=function closeScanner(){ stop(); if(modal){modal.style.display='none';} };

  // ______ Integração com os botões da página ______
  document.getElementById('btnScan')?.addEventListener('click',(e)=>{e.preventDefault();openScanner();});
  window.onHSBarcode=(value)=>{ const input=document.getElementById('chaveNFe'); if(input){input.value=value; input.dispatchEvent(new Event('input',{bubbles:true})); input.dispatchEvent(new Event('change',{bubbles:true}));} };

  // ______ Envio fictício (troque pelo seu Apps Script se quiser) ______
  const btnEnviar=document.getElementById('btnEnviar');
  const btnLimpar=document.getElementById('btnLimpar');
  const statusEnvio=document.getElementById('statusEnvio');

  btnEnviar?.addEventListener('click',async()=>{
    const filial=document.getElementById('filial').value.trim();
    const data=document.getElementById('dataReceb').value;
    const chave=document.getElementById('chaveNFe').value.replace(/\D+/g,'');
    if(!filial || !data || !/^\d{44}$/.test(chave)){
      statusEnvio.textContent='Preencha filial, data e uma chave válida (44 dígitos).';
      statusEnvio.style.color='var(--warn)'; return;
    }
    statusEnvio.textContent='Enviando...'; statusEnvio.style.color='var(--muted)';

    // Exemplo de POST (descomente e troque a URL do seu App Web se quiser)
    // const url='https://script.google.com/macros/s/SEU_APP_WEB/exec';
    // const payload={acao:'recebimentoNFe',filial,data,chave,obs:document.getElementById('obs').value||''};
    // await fetch(url,{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify(payload)});

    await new Promise(r=>setTimeout(r,600)); // simulação
    statusEnvio.textContent='Recebimento registrado com sucesso!'; statusEnvio.style.color='var(--ok)';
  });

  btnLimpar?.addEventListener('click',()=>{
    document.getElementById('filial').value='';
    document.getElementById('dataReceb').value='';
    document.getElementById('chaveNFe').value='';
    document.getElementById('obs').value='';
    statusEnvio.textContent='';
  });
  </script>
</body>
</html>
