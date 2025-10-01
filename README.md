<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Izin Lokasi — Kirim Manual ke WhatsApp</title>
  <style>
    :root { --bg:#0f172a; --card:#0b1220; --accent:#06b6d4; --text:#e6eef8; }
    *{box-sizing:border-box}
    body{
      margin:0;
      min-height:100vh;
      font-family:Inter, system-ui, -apple-system, "Segoe UI", Roboto, "Helvetica Neue", Arial;
      display:flex;
      align-items:center;
      justify-content:center;
      background:linear-gradient(180deg,#071024 0%, #082234 100%);
      color:var(--text);
      padding:20px;
    }
    .card{
      width:100%;
      max-width:720px;
      background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
      border-radius:12px;
      padding:22px;
      box-shadow: 0 6px 30px rgba(3,7,18,0.6);
      border: 1px solid rgba(255,255,255,0.03);
    }
    h1{margin:0 0 8px 0;font-size:20px}
    p.lead{margin:0 0 18px 0;color:#bcd7e6}
    .row{display:flex;gap:12px;flex-wrap:wrap}
    button{
      appearance:none;
      border:0;
      padding:10px 16px;
      border-radius:10px;
      cursor:pointer;
      font-weight:600;
      background:var(--accent);
      color:#022;
      box-shadow: 0 6px 18px rgba(6,182,212,0.12);
    }
    .muted{color:#93a7bd;font-size:14px}
    #info{
      margin-top:16px;
      background:rgba(255,255,255,0.02);
      padding:12px;
      border-radius:8px;
      border:1px solid rgba(255,255,255,0.02);
      font-size:14px;
    }
    a.link{color:var(--accent); text-decoration:none; font-weight:600}
    .small{font-size:13px;color:#9fb9c9}
    footer{margin-top:14px;color:#8fb1c7;font-size:13px}
  </style>
</head>
<body>
  <div class="card">
    <h1>Izinkan semuanya ya</h1>
    <p class="lead">Klik <strong>OK</strong> untuk meminta izin lokasi. Setelah diizinkan, lokasi dan IP akan muncul di layar. Informasi perangkat akan <strong>tidak</strong> ditampilkan di halaman, namun akan dimasukkan ke dalam pesan WhatsApp jika kamu memilih untuk mengirim.</p>

    <div class="row">
      <button id="btnOk">OK</button>
      <button id="btnRefresh" style="background:#20394a;color:var(--text)">Minta Ulang / Refresh</button>
      <button id="btnWhats" style="display:none">Kirim ke alwizz</button>
    </div>

    <div id="info" aria-live="polite">
      <p class="muted">Belum ada data. Tekan OK untuk memulai.</p>
    </div>

    <footer>
      <div class="small">Catatan: informasi perangkat akan <strong>hanya</strong> dimasukkan ke pesan WhatsApp (tidak terlihat di halaman). Tidak ada pengiriman otomatis — WhatsApp akan terbuka pada saat tombol ditekan.</div>
    </footer>
  </div>

  <script>
    // Nomor tujuan WhatsApp — format internasional tanpa tanda '+'
    const WA_NUMBER = "6285189283822"; // 085189283822 -> 62 85189283822

    const btnOk = document.getElementById('btnOk');
    const btnRefresh = document.getElementById('btnRefresh');
    const btnWhats = document.getElementById('btnWhats');
    const info = document.getElementById('info');

    // Device info collected but NOT shown on page
    const deviceInfo = {
      userAgent: navigator.userAgent || "unknown",
      platform: navigator.platform || "unknown",
      screen: `${screen.width}x${screen.height}`
    };

    let state = {
      lat: null,
      lon: null,
      mapsLink: null,
      ip: null,
      timestamp: null
    };

    function setInfoHtml() {
      const lines = [];
      if (state.lat && state.lon) {
        lines.push(`<p><strong>Lokasi (lat,lon):</strong><br>${state.lat}, ${state.lon}</p>`);
        lines.push(`<p><a class="link" href="${state.mapsLink}" target="_blank" rel="noopener">Lihat di Google Maps</a></p>`);
      } else {
        lines.push(`<p class="muted">Lokasi belum tersedia.</p>`);
      }
      lines.push(`<p><strong>IP Publik:</strong> ${state.ip ? state.ip : '<span class="muted">belum diambil</span>'}</p>`);
      if (state.timestamp) lines.push(`<p class="small">Waktu (lokal): ${new Date(state.timestamp).toLocaleString()}</p>`);
      // NOTICE: intentionally do NOT display deviceInfo anywhere here
      info.innerHTML = lines.join('');
    }

    async function fetchIP() {
      try {
        const r = await fetch('https://api.ipify.org?format=json');
        if (!r.ok) throw new Error('failed to fetch ip');
        const j = await r.json();
        return j.ip;
      } catch (e) {
        try {
          const r2 = await fetch('https://ifconfig.co/json');
          if (!r2.ok) throw new Error('fail2');
          const j2 = await r2.json();
          return j2.ip || null;
        } catch {
          return null;
        }
      }
    }

    async function getLocationAndInfo() {
      info.innerHTML = `<p class="muted">Meminta izin lokasi…</p>`;
      state.timestamp = Date.now();

      if (!navigator.geolocation) {
        info.innerHTML = `<p class="muted">Browser tidak mendukung geolokasi.</p>`;
        return;
      }

      const pos = await new Promise((resolve, reject) => {
        navigator.geolocation.getCurrentPosition(resolve, reject, { enableHighAccuracy: true, timeout: 20000 });
      }).catch(err => {
        const msg = (err && err.message) ? err.message : 'Gagal mengambil lokasi';
        info.innerHTML = `<p class="muted">Gagal mendapatkan lokasi: ${escapeHtml(msg)}</p>`;
        return null;
      });

      if (pos) {
        state.lat = pos.coords.latitude.toFixed(6);
        state.lon = pos.coords.longitude.toFixed(6);
        state.mapsLink = `https://www.google.com/maps?q=${state.lat},${state.lon}`;
      }

      state.ip = await fetchIP();
      setInfoHtml();

      btnWhats.style.display = 'inline-block';
    }

    function escapeHtml(s) {
      return String(s).replace(/[&<>"']/g, function(m){ return { '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;' }[m]; });
    }

    btnOk.addEventListener('click', async () => {
      try {
        await getLocationAndInfo();
      } catch (e) {
        console.error(e);
        info.innerHTML = `<p class="muted">Terjadi kesalahan: ${escapeHtml(String(e))}</p>`;
      }
    });

    btnRefresh.addEventListener('click', async () => {
      state.lat = state.lon = state.mapsLink = null;
      state.ip = null;
      state.timestamp = Date.now();
      setInfoHtml();
      btnWhats.style.display = 'none';
      try { await getLocationAndInfo(); } catch(e){ console.error(e); }
    });

    btnWhats.addEventListener('click', () => {
      // Prepare message content including device info (NOT shown on page)
      const parts = [];
      parts.push(`Lokasi: ${state.lat && state.lon ? state.mapsLink : 'Tidak tersedia'}`);
      parts.push(`Koordinat: ${state.lat && state.lon ? state.lat + ', ' + state.lon : 'Tidak tersedia'}`);
      parts.push(`IP Publik: ${state.ip || 'Tidak tersedia'}`);
      // Device info included only in message
      parts.push(`Perangkat (userAgent): ${deviceInfo.userAgent}`);
      parts.push(`Platform: ${deviceInfo.platform}`);
      parts.push(`Resolusi layar: ${deviceInfo.screen}`);
      parts.push(`Waktu: ${new Date(state.timestamp).toLocaleString()}`);
      const message = parts.join('\n');

      const waUrl = `https://wa.me/${WA_NUMBER}?text=${encodeURIComponent(message)}`;
      window.open(waUrl, '_blank', 'noopener');
    });

    // Initialize UI
    setInfoHtml();
  </script>
</body>
</html>
