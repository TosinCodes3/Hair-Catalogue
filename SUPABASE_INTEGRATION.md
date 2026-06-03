# Supabase Integration Guide for Hair Catalogue

## Overview
This guide adds complete Supabase functionality to your Hair Catalogue app, allowing you to:
- Upload and manage haircuts dynamically
- Store images in Supabase Storage instead of local folders
- Create, read, update, and delete haircuts via a management UI

---

## Step 1: Set Up Supabase Project

### Create a Supabase Account
1. Go to [supabase.com](https://supabase.com)
2. Sign up and create a new project
3. Wait for provisioning to complete

### Create Database Table

Go to **SQL Editor** and run this:

```sql
create table if not exists haircuts (
  id uuid default gen_random_uuid() primary key,
  name text not null,
  description text,
  category text,
  hair_type text,
  tags text[],
  images jsonb,
  created_at timestamptz default now()
);

alter table haircuts enable row level security;

create policy "Allow all" on haircuts for all using (true) with check (true);
```

### Create Storage Bucket

1. Go to **Storage** → **New bucket**
2. Name it exactly: `haircut-images`
3. Check **Public bucket**
4. Click **Create**

Then go to **Storage** → **Policies** and add this policy (or run in SQL Editor):

```sql
create policy "Public access" on storage.objects 
  for all using (bucket_id = 'haircut-images') 
  with check (bucket_id = 'haircut-images');
```

### Get Your API Keys

Go to **Project Settings** → **API**:
- Copy **Project URL** (e.g., `https://xxxxxxxxxxxx.supabase.co`)
- Copy **anon / public** key (under "Project API Keys")

You'll need these to connect in the app.

---

## Step 2: Add Supabase Functions to index.html

Add this JavaScript section **before the closing `</script>` tag**:

```javascript
/* ─── SUPABASE SETUP ─── */
let sbUrl = null, sbKey = null, sbConnected = false;

const SUPABASE_SETTINGS_KEY = 'supabaseSettings';

function loadSupabaseSettings() {
  try {
    const settings = JSON.parse(localStorage.getItem(SUPABASE_SETTINGS_KEY) || '{}');
    sbUrl = settings.url || null;
    sbKey = settings.key || null;
    sbConnected = !!(sbUrl && sbKey);
    updateSupabaseStatus();
  } catch (e) {
    sbConnected = false;
  }
}

function saveSupabaseSettings(url, key) {
  localStorage.setItem(SUPABASE_SETTINGS_KEY, JSON.stringify({ url, key }));
  sbUrl = url;
  sbKey = key;
  sbConnected = true;
  updateSupabaseStatus();
}

function clearSupabaseSettings() {
  localStorage.removeItem(SUPABASE_SETTINGS_KEY);
  sbUrl = null;
  sbKey = null;
  sbConnected = false;
  updateSupabaseStatus();
}

function updateSupabaseStatus() {
  const banner = document.getElementById('sbStatusBanner');
  const connectBtn = document.getElementById('sbConnectBtn');
  const disconnectBtn = document.getElementById('sbDisconnectBtn');
  
  if (!banner) return;
  
  if (sbConnected) {
    banner.style.display = 'flex';
    banner.className = 'sb-status-banner connected';
    banner.innerHTML = '<i class="ti ti-check"></i> Connected to Supabase';
    if (connectBtn) connectBtn.style.display = 'none';
    if (disconnectBtn) disconnectBtn.style.display = 'block';
  } else {
    banner.style.display = 'none';
    if (connectBtn) connectBtn.style.display = 'block';
    if (disconnectBtn) disconnectBtn.style.display = 'none';
  }
}

async function supabaseConnect() {
  const url = (document.getElementById('sbUrl')?.value || '').trim();
  const key = (document.getElementById('sbKey')?.value || '').trim();
  
  if (!url || !key) {
    alert('Please enter both Project URL and API key');
    return;
  }
  
  try {
    const response = await fetch(`${url}/rest/v1/haircuts?limit=1`, {
      headers: { 'apikey': key }
    });
    
    if (!response.ok) throw new Error('Invalid credentials');
    
    saveSupabaseSettings(url, key);
    alert('✓ Connected to Supabase!');
    await loadSbCuts();
  } catch (error) {
    alert('Connection failed: ' + error.message);
  }
}

function supabaseDisconnect() {
  if (confirm('Clear Supabase credentials?')) {
    clearSupabaseSettings();
    alert('Disconnected');
  }
}

async function sbFetch(endpoint, options = {}) {
  if (!sbConnected) throw new Error('Supabase not connected');
  
  const url = `${sbUrl}/rest/v1${endpoint}`;
  const headers = {
    'apikey': sbKey,
    'Authorization': `Bearer ${sbKey}`,
    'Content-Type': 'application/json',
    ...options.headers
  };
  
  const response = await fetch(url, { ...options, headers });
  
  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Supabase error: ${error}`);
  }
  
  return response.json();
}

async function sbStorageUpload(bucket, path, file) {
  if (!sbConnected) throw new Error('Supabase not connected');
  
  const url = `${sbUrl}/storage/v1/object/${bucket}/${path}`;
  const response = await fetch(url, {
    method: 'POST',
    headers: {
      'apikey': sbKey,
      'Authorization': `Bearer ${sbKey}`
    },
    body: file
  });
  
  if (!response.ok) throw new Error('Upload failed');
  
  return `${sbUrl}/storage/v1/object/public/${bucket}/${path}`;
}

async function uploadHaircut(name, desc, category, hairType, tags, files) {
  if (!sbConnected) {
    alert('Connect Supabase first!');
    return;
  }
  
  const uploadedImages = [];
  
  for (let i = 0; i < files.length; i++) {
    const file = files[i];
    const timestamp = Date.now();
    const path = `${name.toLowerCase().replace(/\s+/g, '-')}-${timestamp}-${i}.${file.name.split('.').pop()}`;
    
    const publicUrl = await sbStorageUpload('haircut-images', path, file);
    uploadedImages.push({
      src: publicUrl,
      label: i === 0 ? 'Front' : `Angle ${i}`
    });
  }
  
  const haircut = {
    name,
    description: desc,
    category,
    hair_type: hairType,
    tags: tags.split(',').map(t => t.trim()).filter(t => t),
    images: uploadedImages
  };
  
  await sbFetch('/haircuts', {
    method: 'POST',
    body: JSON.stringify([haircut])
  });
  
  return haircut;
}

async function loadSbCuts() {
  if (!sbConnected) return [];
  
  try {
    const data = await sbFetch('/haircuts?order=created_at.desc');
    return data;
  } catch (error) {
    console.error('Error loading haircuts:', error);
    return [];
  }
}

async function deleteSbCut(id) {
  if (!sbConnected) return;
  
  await sbFetch(`/haircuts?id=eq.${id}`, {
    method: 'DELETE'
  });
}

async function injectSbCutsIntoApp() {
  if (!sbConnected) return;
  
  const sbCuts = await loadSbCuts();
  
  // Convert Supabase format to app format
  const converted = sbCuts.map((hc, idx) => ({
    id: hc.id || idx,
    name: hc.name,
    cat: hc.category,
    hairType: hc.hair_type,
    tags: hc.tags || [],
    desc: hc.description,
    imgs: hc.images || []
  }));
  
  // Merge with existing cuts
  cuts.unshift(...converted);
}

/* ─── SUPABASE MANAGEMENT UI ─── */
let mhSelectedFiles = [];

function openManageHaircuts() {
  go('s15');
  loadMhLibrary();
}

function switchMhTab(tab) {
  ['upload', 'library'].forEach(t => {
    document.getElementById(`mhTab${t.charAt(0).toUpperCase() + t.slice(1)}`).classList.toggle('active', t === tab);
    document.getElementById(`mhPane${t.charAt(0).toUpperCase() + t.slice(1)}`).classList.toggle('active', t === tab);
  });
}

function mhHandleFiles(files) {
  mhSelectedFiles = Array.from(files);
  renderMhPreview();
}

function renderMhPreview() {
  const strip = document.getElementById('mhPreviewStrip');
  
  if (!mhSelectedFiles.length) {
    strip.style.display = 'none';
    return;
  }
  
  strip.style.display = 'flex';
  strip.innerHTML = '';
  
  mhSelectedFiles.forEach((file, i) => {
    const url = URL.createObjectURL(file);
    const thumb = document.createElement('div');
    thumb.className = 'mh-preview-thumb';
    thumb.innerHTML = `
      <img src="${url}" alt="Preview">
      <div class="mh-angle-badge">${i === 0 ? 'Hero' : `Angle ${i}`}</div>
      <button class="mh-remove-img" onclick="removeMhFile(${i})"><i class="ti ti-x"></i></button>
    `;
    strip.appendChild(thumb);
  });
}

function removeMhFile(idx) {
  mhSelectedFiles.splice(idx, 1);
  renderMhPreview();
}

async function mhSubmit() {
  if (!mhSelectedFiles.length) {
    alert('Select at least one image');
    return;
  }
  
  const name = document.getElementById('mhName').value.trim();
  if (!name) {
    alert('Enter haircut name');
    return;
  }
  
  const btn = document.getElementById('mhSubmitBtn');
  btn.disabled = true;
  btn.textContent = '⏳ Uploading...';
  
  try {
    const desc = document.getElementById('mhDesc').value;
    const cat = document.getElementById('mhCategory').value;
    const hairType = document.getElementById('mhHairType').value;
    const tags = document.getElementById('mhTags').value;
    
    await uploadHaircut(name, desc, cat, hairType, tags, mhSelectedFiles);
    
    showMhToast('✓ Haircut uploaded!');
    
    document.getElementById('mhName').value = '';
    document.getElementById('mhDesc').value = '';
    document.getElementById('mhTags').value = '';
    mhSelectedFiles = [];
    renderMhPreview();
    
    await loadMhLibrary();
  } catch (error) {
    alert('Upload error: ' + error.message);
  } finally {
    btn.disabled = false;
    btn.textContent = '⬆ Upload Haircut';
  }
}

async function loadMhLibrary() {
  if (!sbConnected) {
    document.getElementById('mhLibContent').innerHTML = '<div class="mh-empty"><i class="ti ti-database"></i><div class="mh-empty-title">Not Connected</div><div class="mh-empty-sub">Connect Supabase in Settings first</div></div>';
    return;
  }
  
  const loading = '<div class="mh-loading"><div class="mh-spinner"></div><p>Loading...</p></div>';
  document.getElementById('mhLibContent').innerHTML = loading;
  
  try {
    const haircuts = await loadSbCuts();
    
    if (!haircuts.length) {
      document.getElementById('mhLibContent').innerHTML = '<div class="mh-empty"><i class="ti ti-inbox"></i><div class="mh-empty-title">No Haircuts Yet</div><div class="mh-empty-sub">Upload your first haircut above</div></div>';
      return;
    }
    
    const html = haircuts.map(hc => {
      const imgs = hc.images || [];
      const thumb = imgs[0]?.src || '';
      const tags = (hc.tags || []).join(', ');
      
      return `
        <div class="mh-cut-card">
          ${thumb ? `<img src="${thumb}" class="mh-cut-card-img" alt="${hc.name}">` : ''}
          <div class="mh-cut-card-body">
            <div class="mh-cut-card-name">${hc.name}</div>
            <div class="mh-cut-card-meta">${hc.hair_type || 'Unknown'} hair</div>
            ${tags ? `<div class="mh-cut-card-meta">${tags}</div>` : ''}
            ${hc.description ? `<div class="mh-cut-card-desc">${hc.description}</div>` : ''}
            <div class="mh-cut-actions">
              <button class="mh-delete-btn" onclick="confirmDeleteMhCut('${hc.id}', '${hc.name}')">
                <i class="ti ti-trash"></i> Delete
              </button>
            </div>
          </div>
        </div>
      `;
    }).join('');
    
    document.getElementById('mhLibContent').innerHTML = html;
  } catch (error) {
    document.getElementById('mhLibContent').innerHTML = `<div class="mh-empty"><i class="ti ti-alert-circle"></i><div class="mh-empty-title">Error</div><div class="mh-empty-sub">${error.message}</div></div>`;
  }
}

function confirmDeleteMhCut(id, name) {
  if (confirm(`Delete "${name}"? This cannot be undone.`)) {
    deleteSbCut(id).then(() => {
      showMhToast('✓ Deleted');
      loadMhLibrary();
    }).catch(e => alert('Delete failed: ' + e.message));
  }
}

function showMhToast(msg) {
  const toast = document.getElementById('mhToast');
  toast.textContent = msg;
  toast.classList.add('show');
  setTimeout(() => toast.classList.remove('show'), 2000);
}

// Load Supabase settings on page load
loadSupabaseSettings();
```

---

## Step 3: Add UI Elements to HTML

Add these screens/modals to your HTML **before the closing `</div>` of `<div class="app">`**:

```html
<!-- ── SUPABASE SETUP SCREEN (s15) ── -->
<div class="screen" id="s15">
  <div class="topbar">
    <div class="back-btn" onclick="go('s7')"><i class="ti ti-arrow-left"></i></div>
    <div class="topbar-title">Manage Haircuts</div>
    <div style="width:34px"></div>
  </div>
  <div class="mh-body">
    <div class="mh-tabs">
      <button class="mh-tab active" id="mhTabUpload" onclick="switchMhTab('upload')">Upload</button>
      <button class="mh-tab" id="mhTabLibrary" onclick="switchMhTab('library')">Library</button>
    </div>
    
    <!-- Upload pane -->
    <div class="mh-pane active" id="mhPaneUpload">
      <div class="mh-upload-zone" id="mhUploadZone">
        <i class="ti ti-cloud-upload"></i>
        <div class="mh-upload-zone-label">Tap to select images</div>
        <div class="mh-upload-zone-sub">First image = hero · Add more for angle views</div>
        <input type="file" id="mhFileInput" accept="image/*" multiple onchange="mhHandleFiles(this.files)">
      </div>
      <div class="mh-preview-strip" id="mhPreviewStrip" style="display:none;"></div>
      
      <div class="mh-field">
        <label>Haircut name *</label>
        <input id="mhName" type="text" placeholder="e.g. Brush Back">
      </div>
      <div class="mh-field">
        <label>Description</label>
        <textarea id="mhDesc" rows="3" placeholder="Short description..."></textarea>
      </div>
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:10px;">
        <div class="mh-field">
          <label>Category</label>
          <select id="mhCategory">
            <option value="">— Select —</option>
            <option value="brushback">Brush Back</option>
            <option value="burstfade">Burst Fade</option>
            <option value="taper">Taper</option>
            <!-- Add more as needed -->
          </select>
        </div>
        <div class="mh-field">
          <label>Hair type</label>
          <select id="mhHairType">
            <option value="straight">Straight</option>
            <option value="wavy">Wavy</option>
            <option value="curly">Curly</option>
            <option value="textured">Textured</option>
          </select>
        </div>
      </div>
      <div class="mh-field">
        <label>Tags (comma separated)</label>
        <input id="mhTags" type="text" placeholder="e.g. Skin Fade, Taper">
      </div>
      <button class="mh-submit-btn" id="mhSubmitBtn" onclick="mhSubmit()">
        <i class="ti ti-upload"></i> Upload Haircut
      </button>
    </div>
    
    <!-- Library pane -->
    <div class="mh-pane" id="mhPaneLibrary">
      <div id="mhLibContent" style="flex:1;"></div>
    </div>
  </div>
</div>

<!-- ── SUPABASE SETUP SCREEN (s16) ── -->
<div class="screen" id="s16">
  <div class="topbar">
    <div class="back-btn" onclick="go('s7')"><i class="ti ti-arrow-left"></i></div>
    <div class="topbar-title">Connect Supabase</div>
    <div style="width:34px"></div>
  </div>
  <div class="sb-setup-body">
    <div class="sb-logo-row">
      <div class="sb-logo-icon"><i class="ti ti-database"></i></div>
      <div>
        <div class="sb-logo-text">Supabase</div>
        <div class="sb-logo-sub">Backend for your haircut library</div>
      </div>
    </div>
    
    <div id="sbStatusBanner" style="display:none;"></div>
    
    <div class="sb-field">
      <label>Project URL</label>
      <input id="sbUrl" type="url" placeholder="https://xxxxxxxxxxxx.supabase.co">
    </div>
    <div class="sb-field">
      <label>Anon / Public Key</label>
      <input id="sbKey" type="text" placeholder="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...">
    </div>
    
    <button class="sb-connect-btn" id="sbConnectBtn" onclick="supabaseConnect()">
      <i class="ti ti-plug"></i> Connect
    </button>
    <button class="sb-disconnect-btn" id="sbDisconnectBtn" style="display:none;" onclick="supabaseDisconnect()">
      Disconnect & clear credentials
    </button>
  </div>
</div>

<div class="mh-toast" id="mhToast"></div>
```

---

## Step 4: Add CSS Styles

Add these CSS classes to your `<style>` section:

```css
/* ── SUPABASE SETUP & MANAGE HAIRCUTS ── */
.sb-setup-body{padding:24px 20px 40px;overflow-y:auto;flex:1;}
.sb-logo-row{display:flex;align-items:center;gap:14px;margin-bottom:28px;}
.sb-logo-icon{width:48px;height:48px;border-radius:14px;background:var(--bg2);border:0.5px solid var(--border2);display:flex;align-items:center;justify-content:center;flex-shrink:0;}
.sb-logo-icon i{font-size:24px;color:var(--text2);}
.sb-logo-text{font-size:22px;font-weight:700;color:var(--text);letter-spacing:-0.03em;}
.sb-logo-sub{font-size:12px;color:var(--text3);margin-top:2px;}
.sb-field{margin-bottom:16px;}
.sb-field label{display:block;font-size:11px;font-weight:600;color:var(--text3);text-transform:uppercase;letter-spacing:0.08em;margin-bottom:8px;}
.sb-field input{width:100%;padding:13px 14px;border-radius:12px;border:0.5px solid var(--border2);background:var(--bg2);color:var(--text);font-family:inherit;font-size:14px;outline:none;transition:border-color 0.15s;}
.sb-field input:focus{border-color:var(--opt-on-border);}
.sb-connect-btn{width:100%;padding:15px;background:var(--lock-bg);color:var(--lock-text);border:none;border-radius:12px;font-size:15px;font-weight:700;cursor:pointer;display:flex;align-items:center;justify-content:center;gap:9px;font-family:inherit;margin-top:8px;transition:opacity 0.15s;}
.sb-connect-btn:active{opacity:0.8;}
.sb-status-banner{padding:12px 14px;border-radius:12px;font-size:13px;font-weight:500;margin-bottom:20px;display:flex;align-items:center;gap:10px;}
.sb-status-banner.connected{background:rgba(93,187,138,0.15);color:#5dbb8a;border:0.5px solid rgba(93,187,138,0.3);}
.sb-disconnect-btn{width:100%;padding:13px;background:transparent;border:0.5px solid var(--border2);border-radius:12px;font-size:13px;color:var(--text2);cursor:pointer;font-family:inherit;margin-top:8px;}

/* Manage screen */
.mh-body{display:flex;flex-direction:column;flex:1;overflow:hidden;}
.mh-tabs{display:flex;border-bottom:0.5px solid var(--border);flex-shrink:0;}
.mh-tab{flex:1;padding:12px 8px;background:transparent;border:none;border-bottom:2px solid transparent;font-family:inherit;font-size:13px;font-weight:600;color:var(--text2);cursor:pointer;transition:all 0.15s;}
.mh-tab.active{color:var(--text);border-bottom-color:var(--text);}
.mh-pane{display:none;flex-direction:column;flex:1;overflow-y:auto;padding:16px;}
.mh-pane.active{display:flex;}

/* Upload form */
.mh-upload-zone{border:2px dashed var(--border2);border-radius:16px;background:var(--bg2);padding:28px 20px;display:flex;flex-direction:column;align-items:center;gap:10px;cursor:pointer;transition:border-color 0.15s;margin-bottom:16px;position:relative;}
.mh-upload-zone i{font-size:32px;color:var(--text3);}
.mh-upload-zone-label{font-size:14px;font-weight:600;color:var(--text2);}
.mh-upload-zone-sub{font-size:12px;color:var(--text3);}
.mh-upload-zone input[type=file]{position:absolute;inset:0;opacity:0;cursor:pointer;}

.mh-preview-strip{display:flex;gap:8px;overflow-x:auto;padding-bottom:4px;margin-bottom:16px;scrollbar-width:none;}
.mh-preview-strip::-webkit-scrollbar{display:none;}
.mh-preview-thumb{position:relative;flex-shrink:0;width:72px;height:72px;border-radius:10px;overflow:hidden;background:var(--bg3);}
.mh-preview-thumb img{width:100%;height:100%;object-fit:cover;}
.mh-preview-thumb .mh-remove-img{position:absolute;top:3px;right:3px;width:18px;height:18px;border-radius:50%;background:rgba(0,0,0,0.6);border:none;cursor:pointer;display:flex;align-items:center;justify-content:center;}
.mh-preview-thumb .mh-remove-img i{font-size:11px;color:#fff;}
.mh-preview-thumb .mh-angle-badge{position:absolute;bottom:2px;left:3px;font-size:9px;font-weight:700;color:#fff;background:rgba(0,0,0,0.55);border-radius:4px;padding:1px 4px;}

.mh-field{margin-bottom:14px;}
.mh-field label{display:block;font-size:11px;font-weight:600;color:var(--text3);text-transform:uppercase;letter-spacing:0.07em;margin-bottom:7px;}
.mh-field input,.mh-field select,.mh-field textarea{width:100%;padding:12px 14px;border-radius:12px;border:0.5px solid var(--border2);background:var(--bg2);color:var(--text);font-family:inherit;font-size:14px;outline:none;transition:border-color 0.15s;}
.mh-field select{background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 24 24' fill='none' stroke='%23888' stroke-width='2'%3E%3Cpolyline points='6 9 12 15 18 9'%3E%3C/polyline%3E%3C/svg%3E");background-repeat:no-repeat;background-position:right 14px center;padding-right:36px;}
.mh-field input:focus,.mh-field select:focus{border-color:var(--opt-on-border);}
.mh-field textarea{resize:none;}
.mh-submit-btn{width:100%;padding:15px;background:var(--lock-bg);color:var(--lock-text);border:none;border-radius:12px;font-size:15px;font-weight:700;cursor:pointer;display:flex;align-items:center;justify-content:center;gap:9px;font-family:inherit;transition:opacity 0.15s;margin-top:4px;}
.mh-submit-btn:active{opacity:0.8;}
.mh-submit-btn:disabled{opacity:0.4;cursor:not-allowed;}

/* Library tab */
.mh-cut-card{background:var(--bg2);border:0.5px solid var(--border);border-radius:14px;overflow:hidden;margin-bottom:10px;position:relative;}
.mh-cut-card-img{width:100%;height:180px;object-fit:cover;display:block;opacity:0;transition:opacity 0.3s;}
.mh-cut-card-img.loaded{opacity:1;}
.mh-cut-card-body{padding:12px 14px 14px;}
.mh-cut-card-name{font-size:15px;font-weight:700;color:var(--text);}
.mh-cut-card-meta{font-size:12px;color:var(--text2);margin-top:3px;}
.mh-cut-card-desc{font-size:12px;color:var(--text3);margin-top:4px;line-height:1.4;}
.mh-cut-actions{display:flex;gap:8px;margin-top:12px;}
.mh-delete-btn{flex:1;padding:10px;background:rgba(229,85,85,0.1);color:#e55;border:0.5px solid rgba(229,85,85,0.25);border-radius:10px;font-size:13px;font-weight:600;cursor:pointer;font-family:inherit;display:flex;align-items:center;justify-content:center;gap:6px;transition:opacity 0.15s;}
.mh-delete-btn:active{opacity:0.7;}

.mh-empty{text-align:center;padding:48px 24px;flex:1;display:flex;flex-direction:column;align-items:center;justify-content:center;gap:12px;}
.mh-empty i{font-size:36px;color:var(--text3);}
.mh-empty-title{font-size:16px;font-weight:700;color:var(--text);}
.mh-empty-sub{font-size:13px;color:var(--text3);line-height:1.6;}

.mh-loading{text-align:center;padding:48px 24px;color:var(--text3);font-size:13px;display:flex;flex-direction:column;align-items:center;gap:12px;}
.mh-spinner{width:24px;height:24px;border:2px solid var(--border2);border-top-color:var(--text2);border-radius:50%;animation:spin 0.7s linear infinite;}
@keyframes spin{to{transform:rotate(360deg);}}

.mh-toast{position:fixed;bottom:90px;left:50%;transform:translateX(-50%);background:var(--text);color:var(--bg);padding:11px 20px;border-radius:10px;font-size:13px;font-weight:600;z-index:300;opacity:0;transition:opacity 0.25s;pointer-events:none;white-space:nowrap;}
.mh-toast.show{opacity:1;}
```

---

## Step 5: Add Settings Link

Update your **Settings screen (s7)** to add a link to manage haircuts:

```html
<div class="settings-section">
  <div class="settings-section-label">Content Management</div>
  <div class="settings-row" style="cursor:pointer;" onclick="go('s16')">
    <div class="settings-row-left">
      <div class="settings-row-icon"><i class="ti ti-database"></i></div>
      <div>
        <div class="settings-row-label">Supabase Setup</div>
        <div class="settings-row-sub">Connect your database</div>
      </div>
    </div>
    <i class="ti ti-chevron-right" style="color:var(--text3);font-size:16px;"></i>
  </div>
  <div class="settings-row" style="cursor:pointer;" onclick="go('s15')">
    <div class="settings-row-left">
      <div class="settings-row-icon"><i class="ti ti-upload"></i></div>
      <div>
        <div class="settings-row-label">Manage Haircuts</div>
        <div class="settings-row-sub">Upload & manage images</div>
      </div>
    </div>
    <i class="ti ti-chevron-right" style="color:var(--text3);font-size:16px;"></i>
  </div>
</div>
```

---

## Usage

1. **Go to Settings** → **Supabase Setup**
2. **Enter your Project URL and API Key** from Supabase
3. **Click Connect**
4. **Go to Settings** → **Manage Haircuts**
5. **Upload Tab**: Select images, fill details, click "Upload Haircut"
6. **Library Tab**: View all uploaded haircuts, delete if needed
7. **Haircuts automatically load** from Supabase when app starts

---

## Troubleshooting

- **"Connection failed"** → Check URL and key are correct
- **Images not uploading** → Ensure bucket is named `haircut-images`
- **No haircuts showing** → Make sure you're connected and have uploaded some

Done! Your app now dynamically manages haircuts via Supabase. 🎉
