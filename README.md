<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0">
<title>Modificateur Data Matrix</title>
<link rel="manifest" href="manifest.json">
<meta name="theme-color" content="#1d6aff">
<script src="https://unpkg.com/@zxing/library@latest"></script>
<script src="https://cdn.jsdelivr.net/npm/bwip-js@3/dist/bwip-js-min.js"></script>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
  * { box-sizing: border-box; margin: 0; padding: 0; font-family: 'Inter', sans-serif; }
  body { background: #f0f4f8; padding: 15px; color: #111827; }
  .card { background: white; border-radius: 20px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); margin-bottom: 15px; overflow: hidden; border: 1px solid #dde3ec; }
  .header { padding: 15px; background: #1d6aff; color: white; font-weight: 700; text-align: center; font-size: 18px; }
  
  /* Scanner */
  #live-view { position: relative; width: 100%; aspect-ratio: 1/1; background: #000; display: none; }
  #video { width: 100%; height: 100%; object-fit: cover; }
  .scan-overlay { position: absolute; top: 50%; left: 50%; transform: translate(-50%,-50%); width: 70%; height: 70%; border: 3px solid #fff; border-radius: 15px; box-shadow: 0 0 0 1000px rgba(0,0,0,0.5); }

  /* Interface de Modification */
  #edit-screen { display: none; padding: 15px; }
  
  /* Comparaison Avant / Après */
  .comparison { display: flex; gap: 10px; margin-bottom: 20px; }
  .comp-box { flex: 1; background: #f8fafc; border: 1px solid #e2e8f0; padding: 10px; border-radius: 12px; }
  .comp-box.after { background: #eef3ff; border-color: #dbeafe; }
  .label { font-size: 10px; font-weight: 800; color: #64748b; text-transform: uppercase; margin-bottom: 5px; display: block; }
  .data-preview { font-family: monospace; font-size: 11px; word-break: break-all; color: #334155; height: 60px; overflow-y: auto; }
  .after .data-preview { color: #1d6aff; font-weight: 600; }

  /* Zone de saisie Q */
  .input-group { background: #fff; padding: 20px; border-radius: 15px; border: 2px solid #1d6aff; text-align: center; margin-bottom: 20px; }
  .input-label { display: block; font-weight: 700; color: #1d6aff; margin-bottom: 10px; }
  .qty-field { width: 120px; padding: 10px; font-size: 32px; font-weight: 800; border: none; border-bottom: 4px solid #1d6aff; text-align: center; color: #111827; outline: none; }

  /* Rendu du Code */
  .qr-display { text-align: center; padding: 20px; background: #fff; border-radius: 15px; border: 1px solid #eee; }
  #canvas-holder canvas { max-width: 100%; height: auto; }

  .btn { display: block; width: 100%; padding: 16px; border: none; border-radius: 12px; font-weight: 700; font-size: 16px; cursor: pointer; transition: 0.2s; }
  .btn-blue { background: #1d6aff; color: white; margin-top: 10px; }
  .btn-green { background: #10b981; color: white; margin-top: 15px; }
  .btn-outline { background: transparent; border: 2px solid #cbd5e1; color: #64748b; margin-top: 20px; }
  .btn:active { transform: scale(0.97); }
</style>
</head>
<body>

<div class="card" id="scan-screen">
  <div class="header">Scanner le Data Matrix</div>
  <div id="live-view">
    <video id="video" playsinline></video>
    <div class="scan-overlay"></div>
  </div>
  <div style="padding: 20px;">
    <button class="btn btn-blue" id="start-btn" onclick="startCamera()">Démarrer la Caméra</button>
  </div>
</div>

<div class="card" id="edit-screen">
  <div class="header">Modification du Code</div>
  
  <div class="comparison">
    <div class="comp-box">
      <span class="label">Avant (Original)</span>
      <div class="data-preview" id="old-text"></div>
    </div>
    <div class="comp-box after">
      <span class="label" style="color:#1d6aff">Après (Modifié)</span>
      <div class="data-preview" id="new-text"></div>
    </div>
  </div>

  <div class="input-group">
    <span class="input-label">Modifier la Quantité (Q)</span>
    <input type="number" id="q-input" class="qty-field" oninput="applyModification()">
  </div>

  <div class="qr-display">
    <span class="label">Nouveau Code Généré</span>
    <div id="canvas-holder"></div>
    <button class="btn btn-green" onclick="saveImage()">💾 Enregistrer l'image</button>
  </div>

  <button class="btn btn-outline" onclick="location.reload()">🔄 Nouveau Scan</button>
</div>

<script>
  const codeReader = new ZXing.BrowserMultiFormatReader();
  let originalRaw = "";

  // 1. Démarrer le scanner
  function startCamera() {
    document.getElementById('live-view').style.display = 'block';
    document.getElementById('start-btn').style.display = 'none';

    codeReader.decodeFromVideoDevice(null, 'video', (result, err) => {
      if (result) {
        codeReader.reset();
        originalRaw = result.text;
        showEditScreen();
      }
    });
  }

  // 2. Afficher l'interface de modification
  function showEditScreen() {
    document.getElementById('scan-screen').style.display = 'none';
    document.getElementById('edit-screen').style.display = 'block';
    document.getElementById('old-text').textContent = originalRaw;

    // Détecter automatiquement le Q actuel (ex: cherche Q suivi de chiffres)
    const match = originalRaw.match(/Q(\d+)/);
    if (match) {
      document.getElementById('q-input').value = match[1];
    }
    applyModification();
  }

  // 3. Appliquer la modification en temps réel
  function applyModification() {
    const val = document.getElementById('q-input').value;
    // Remplace le premier Q[chiffres] trouvé par Q[nouvelle valeur]
    const modified = originalRaw.replace(/Q\d+/, "Q" + val);
    
    document.getElementById('new-text').textContent = modified;
    renderNewDataMatrix(modified);
  }

  // 4. Générer l'image du nouveau Data Matrix
  function renderNewDataMatrix(content) {
    const holder = document.getElementById('canvas-holder');
    holder.innerHTML = "";
    const canvas = document.createElement('canvas');
    try {
      bwipjs.toCanvas(canvas, {
        bcid: 'datamatrix',
        text: content,
        scale: 4,
        padding: 10,
        backgroundcolor: 'ffffff'
      });
      holder.appendChild(canvas);
    } catch (e) { console.error("Erreur BWIP:", e); }
  }

  // 5. Sauvegarder l'image sur le téléphone
  function saveImage() {
    const canvas = document.querySelector('#canvas-holder canvas');
    if (canvas) {
      const link = document.createElement('a');
      link.download = 'datamatrix_modifie.png';
      link.href = canvas.toDataURL();
      link.click();
    }
  }

  // Enregistrement SW pour le hors-ligne
  if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('sw.js');
  }
</script>

</body>
</html>
