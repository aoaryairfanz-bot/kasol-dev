<script>
    import { fade, fly, slide, scale } from 'svelte/transition';
    import { quintOut, cubicOut } from 'svelte/easing';
    import { tweened } from 'svelte/motion';
    
    import Header from './lib/Header.svelte';
    import Marketing from './lib/Marketing.svelte';
    import Footer from './lib/Footer.svelte';

    let activeTab = 'remove-bg'; 
    let loggedInUser = null;
    let file = null;
    let videoUrl = "";
    let isLoading = false;
    let resultUrl = null;
    let resultType = null;
    let errorMessage = "";
    let successMessage = ""; 
    let isDragging = false;

    // State Stock Opname
    let itemId = ""; 
    let internalId = null; 
    let currentStock = null;
    let productDetails = null; 
    let qtyInput = "";
    let stockMode = "add"; 

    // State Master Barang (Superadmin)
    let showMasterModal = false;
    let newBarangNama = "";
    let newBarangSku = "";
    let isAddingManual = false;
    let excelFile = null;
    let isImporting = false;
    let masterMessage = ""; 
    let isMasterError = false;

    let previewImage = null;
    let previewTimer; 
    const downloadProgress = tweened(0, { duration: 400, easing: cubicOut });
    let progressInterval;

    const API_BASE = "https://aryairfanz-kasol-dev.hf.space";

    const pageContent = {
        'remove-bg': { title: "Remove Background", desc: "Remove image backgrounds in seconds.", formats: "JPG, PNG, or WebP - Up to 20MB", btnText: "Upload Image", accept: "image/*" },
        'stock-opname': { title: "Stock Opname Gudang", desc: "Input stok barang secara real-time.", formats: "Masukkan SKU Barang", btnText: "Cek Stok Barang", accept: "" },
        'download-video': { title: "Video Downloader", desc: "Download video YouTube, TikTok, atau Instagram.", formats: "Paste link video di bawah", btnText: "Download Video", accept: "" }
    };

    $: if (activeTab === 'download-video') {
        if (!videoUrl || videoUrl.trim() === "") {
            previewImage = null;
        } else {
            clearTimeout(previewTimer);
            previewTimer = setTimeout(() => { fetchThumbnailPreview(videoUrl); }, 300);
        }
    }

    async function fetchThumbnailPreview(url) {
        if (resultType === 'video') { resultUrl = null; resultType = null; }
        previewImage = null;
        const ytMatch = url.match(/(?:youtu\.be\/|youtube\.com\/(?:embed\/|v\/|shorts\/|watch\?v=|watch\?.+&v=))((\w|-){11})/);
        if (ytMatch && ytMatch[1]) { previewImage = `https://img.youtube.com/vi/${ytMatch[1]}/hqdefault.jpg`; return; }
        if (url.includes('tiktok.com')) {
            try {
                const response = await fetch(`https://www.tiktok.com/oembed?url=${encodeURIComponent(url)}`);
                if (response.ok) {
                    const data = await response.json();
                    if (data.thumbnail_url) { previewImage = data.thumbnail_url; return; }
                }
            } catch (err) { console.log("Gagal memuat preview", err); }
        }
    }

    function tilt(node) {
        const handleMouseMove = (e) => {
            if (window.innerWidth < 768) return; 
            const rect = node.getBoundingClientRect();
            const x = e.clientX - rect.left; const y = e.clientY - rect.top;
            const centerX = rect.width / 2; const centerY = rect.height / 2;
            const rotateX = ((y - centerY) / centerY) * -5; const rotateY = ((x - centerX) / centerX) * 5;
            node.style.transform = `perspective(1000px) rotateX(${rotateX}deg) rotateY(${rotateY}deg) scale3d(1.01, 1.01, 1.01)`;
        };
        const handleMouseLeave = () => { node.style.transform = `perspective(1000px) rotateX(0deg) rotateY(0deg) scale3d(1, 1, 1)`; };
        node.addEventListener('mousemove', handleMouseMove); node.addEventListener('mouseleave', handleMouseLeave);
        return { destroy() { node.removeEventListener('mousemove', handleMouseMove); node.removeEventListener('mouseleave', handleMouseLeave); } };
    }

    function handleDragOver(e) { e.preventDefault(); isDragging = true; }
    function handleDragLeave() { isDragging = false; }
    function handleDrop(e) { e.preventDefault(); isDragging = false; if (e.dataTransfer.files && e.dataTransfer.files.length > 0) handleFileSelect(e.dataTransfer.files[0]); }
    function handleFileInput(e) { if (e.target.files && e.target.files.length > 0) handleFileSelect(e.target.files[0]); }

    function handleFileSelect(selectedFile) {
        if (activeTab === 'remove-bg' && !selectedFile.type.startsWith('image/')) { errorMessage = "Harap upload file gambar!"; return; }
        file = selectedFile; processFile();
    }

    function startProgress() {
        downloadProgress.set(0, {duration: 0});
        progressInterval = setInterval(() => {
            downloadProgress.update(n => {
                if (n >= 90) { clearInterval(progressInterval); return 90; } 
                return n + (Math.random() * 8); 
            });
        }, 800);
    }

    $: activeTab, resetStateOnTabChange();
    function resetStateOnTabChange() {
        file = null; videoUrl = ""; resultUrl = null; errorMessage = ""; successMessage = ""; isLoading = false; previewImage = null;
        itemId = ""; internalId = null; currentStock = null; productDetails = null; qtyInput = ""; stockMode = "add";
        downloadProgress.set(0, {duration: 0}); clearInterval(progressInterval);
    }

    async function processFile() {
        isLoading = true; errorMessage = ""; resultUrl = null;
        const formData = new FormData(); formData.append("file", file);
        try {
            const response = await fetch(`${API_BASE}/remove-bg`, { method: "POST", body: formData });
            if (!response.ok) throw new Error("Gagal memproses file.");
            const blob = await response.blob();
            resultUrl = URL.createObjectURL(blob);
            resultType = 'image';
        } catch (err) { errorMessage = err.message; } finally { isLoading = false; }
    }

    async function processVideo() {
        if (!videoUrl) { errorMessage = "Masukkan link video dulu!"; return; }
        isLoading = true; errorMessage = ""; resultUrl = null;
        startProgress(); 
        const formData = new FormData(); formData.append("url", videoUrl);
        try {
            const response = await fetch(`${API_BASE}/download-video`, { method: "POST", body: formData });
            if (!response.ok) throw new Error("Gagal mendownload.");
            const blob = await response.blob();
            resultUrl = URL.createObjectURL(blob);
            resultType = 'video';
            clearInterval(progressInterval);
            await downloadProgress.set(100, { duration: 200 }); 
            const contentDisposition = response.headers.get('Content-Disposition');
            let filename = "video_terunduh.mp4";
            if (contentDisposition && contentDisposition.includes('filename=')) filename = contentDisposition.split('filename=')[1].replace(/"/g, '');
            triggerDownload(resultUrl, filename);
        } catch (err) { clearInterval(progressInterval); downloadProgress.set(0, {duration: 0}); errorMessage = err.message; } finally { isLoading = false; }
    }

    async function fetchCurrentStock() {
        if (!itemId) return;
        isLoading = true; errorMessage = ""; successMessage = ""; currentStock = null; productDetails = null; internalId = null;
        try {
            const response = await fetch(`${API_BASE}/api/stock/${itemId}`);
            const data = await response.json();
            if (response.ok) {
                currentStock = data.current_total;
                internalId = data.id; 
                productDetails = { nama: data.nama, sku: data.sku };
            } else { errorMessage = data.detail || "Barang tidak ditemukan"; }
        } catch (err) { errorMessage = "Gagal terhubung ke database."; }
        finally { isLoading = false; }
    }

    async function submitStock() {
        if (qtyInput === "" || !internalId) return;
        isLoading = true; errorMessage = ""; successMessage = "";
        const payload = { barang_id: internalId, user_id: loggedInUser.id, action: stockMode, input_qty: parseInt(qtyInput) };
        try {
            const response = await fetch(`${API_BASE}/api/stock/input`, {
                method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload)
            });
            if (response.ok) {
                successMessage = "Stok berhasil diperbarui!";
                await fetchCurrentStock(); 
                qtyInput = "";
                setTimeout(() => successMessage = "", 3000);
            } else { throw new Error("Gagal menyimpan data stok."); }
        } catch (err) { errorMessage = err.message; }
        finally { isLoading = false; }
    }

    async function addManualBarang() {
        if (!newBarangNama) { masterMessage = "Nama barang wajib diisi!"; isMasterError = true; return; }
        isAddingManual = true; masterMessage = "";
        try {
            const response = await fetch(`${API_BASE}/api/stock/barang`, {
                method: "POST", headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ nama: newBarangNama, sku: newBarangSku, user_id: loggedInUser.id })
            });
            const data = await response.json();
            if (response.ok) { masterMessage = data.message; isMasterError = false; newBarangNama = ""; newBarangSku = ""; } 
            else { throw new Error(data.detail || "Gagal menambah barang."); }
        } catch (err) { masterMessage = err.message; isMasterError = true; } 
        finally { isAddingManual = false; }
    }

    function handleExcelInput(e) {
        if (e.target.files && e.target.files.length > 0) { excelFile = e.target.files[0]; uploadExcel(); }
    }

    async function uploadExcel() {
        if (!excelFile) return;
        isImporting = true; masterMessage = "";
        const formData = new FormData();
        formData.append("file", excelFile);
        formData.append("user_id", loggedInUser.id);
        try {
            const response = await fetch(`${API_BASE}/api/stock/import-barang`, { method: "POST", body: formData });
            const data = await response.json();
            if (response.ok) { masterMessage = data.message; isMasterError = false; } 
            else { throw new Error(data.detail || "Gagal mengimport file."); }
        } catch (err) { masterMessage = err.message; isMasterError = true; } 
        finally { isImporting = false; excelFile = null; }
    }

    function triggerDownload(url, filename) {
        const a = document.createElement('a'); a.href = url; a.download = filename || 'download';
        document.body.appendChild(a); a.click(); document.body.removeChild(a);
    }
</script>

<Header bind:activeTab={activeTab} bind:loggedInUser={loggedInUser} />

<section class="hero">
    <div class="breadcrumb">Home / Tools / {pageContent[activeTab].title}</div>
    
    {#key activeTab}
        <div class="hero-text" in:fly={{ y: 20, duration: 400, delay: 200, easing: quintOut }} out:fade={{ duration: 150 }}>
            <h1>{pageContent[activeTab].title}</h1>
            <p>{pageContent[activeTab].desc}</p>
        </div>
    {/key}

    <div class="upload-card {isDragging ? 'drag-active' : ''}" use:tilt>
        
        {#if errorMessage} <div class="error-msg" transition:slide={{ duration: 300 }}>{errorMessage}</div> {/if}
        {#if successMessage} <div class="success-msg" transition:slide={{ duration: 300 }}>{successMessage}</div> {/if}

        {#if isLoading && activeTab === 'remove-bg'}
            <div class="loading-state" in:fade={{ duration: 200 }}>
                <div class="spinner"></div><h2>Memproses...</h2>
            </div>
        {:else if resultUrl && resultType === 'image'}
            <div class="result-state" in:scale={{ duration: 400, start: 0.9, easing: quintOut }}>
                <img src={resultUrl} alt="Hasil" class="checkerboard-bg result-img"/>
                <div class="btn-group">
                    <button class="btn-3d pink" on:click={() => triggerDownload(resultUrl, 'no-bg.png')}>Download</button>
                    <button class="btn-3d blue" on:click={() => {activeTab = activeTab; resultUrl = null;}}>Ulangi</button>
                </div>
            </div>
        {:else}
            {#key activeTab}
                <div in:fly={{ y: 15, duration: 400, delay: 100 }} class="input-wrapper">
                    
                    {#if activeTab === 'download-video'}
                        <div class="responsive-layout">
                            <div class="layout-left">
                                <h2>Masukkan Link Video</h2>
                                <p class="sub">{pageContent[activeTab].formats}</p>
                                <div class="form-stack">
                                    <input type="text" bind:value={videoUrl} placeholder="Paste link video..." class="url-input" />
                                    <button class="btn-3d pink w-full" on:click={processVideo} disabled={isLoading || !videoUrl}>{isLoading ? 'Loading...' : 'Download'}</button>
                                    {#if isLoading || $downloadProgress > 0}
                                        <div class="progress-bar-bg"><div class="progress-bar-fill" style="width: {$downloadProgress}%"></div></div>
                                    {/if}
                                </div>
                            </div>
                            <div class="layout-right video-box">
                                {#if resultUrl && resultType === 'video'} <video src={resultUrl} controls autoplay class="preview-media"></video>
                                {:else if previewImage} <img src={previewImage} alt="Preview" class="preview-media" />
                                {:else} <div class="placeholder-icon">Video</div> {/if}
                            </div>
                        </div>

                    {:else if activeTab === 'stock-opname'}
                        <div class="responsive-layout">
                            <div class="layout-left">
                                <h2>Cari Data Barang</h2>
                                <p class="sub">{pageContent[activeTab].formats}</p>
                                
                                {#if !loggedInUser}
                                    <div class="login-alert">Fitur terkunci. Silakan Login.</div>
                                {:else}
                                    <div class="search-form">
                                        <input type="text" bind:value={itemId} placeholder="ID atau SKU" class="url-input" />
                                        <button class="btn-3d pink" on:click={fetchCurrentStock} disabled={isLoading || !itemId}>Cek</button>
                                    </div>

                                    {#if internalId !== null}
                                        <div class="stock-form" in:slide>
                                            <div class="mode-toggle">
                                                <label class:active={stockMode === 'add'}><input type="radio" bind:group={stockMode} value="add" hidden> Tambah</label>
                                                <label class:active={stockMode === 'update'}><input type="radio" bind:group={stockMode} value="update" hidden> Update</label>
                                            </div>
                                            <input type="number" bind:value={qtyInput} placeholder="Angka..." class="url-input" />
                                            <button class="btn-3d blue w-full" on:click={submitStock} disabled={isLoading || !qtyInput}>Simpan</button>
                                        </div>
                                    {/if}

                                    {#if loggedInUser.role === 'superadmin'}
                                        <button class="btn-master" on:click={() => showMasterModal = true}>Kelola Master Barang</button>
                                    {/if}
                                {/if}
                            </div>
                            <div class="layout-right">
                                {#if productDetails && loggedInUser} 
                                    <div class="product-card" in:fade>
                                        <span class="sku-tag">SKU: {productDetails.sku}</span>
                                        <h3 class="name">{productDetails.nama}</h3>
                                        <div class="stock-circle">
                                            <span class="stock-val">{currentStock}</span>
                                            <span class="stock-unit">pcs</span>
                                        </div>
                                        <div class="sync-info"><span class="dot"></span> Sync Aktif</div>
                                    </div>
                                {:else}
                                    <div class="placeholder-icon">📦</div>
                                {/if}
                            </div>
                        </div>

                    {:else}
                        <div class="dropzone" on:dragover={handleDragOver} on:dragleave={handleDragLeave} on:drop={handleDrop}>
                            <h2>Upload Gambar</h2>
                            <p>{pageContent[activeTab].formats}</p>
                            <label class="btn-3d pink">Pilih File <input type="file" accept={pageContent[activeTab].accept} on:change={handleFileInput} hidden /></label>
                        </div>
                    {/if}
                </div>
            {/key}
        {/if}
    </div>
</section>

{#if showMasterModal}
    <div class="modal-overlay" on:click|self={() => showMasterModal = false}>
        <div class="modal-box" transition:scale>
            <h2>Master Barang</h2>
            {#if masterMessage}<div class="msg-box {isMasterError ? 'error' : 'success'}">{masterMessage}</div>{/if}
            <div class="form-group">
                <input type="text" placeholder="Nama Barang" bind:value={newBarangNama} class="modal-input" />
                <input type="text" placeholder="SKU" bind:value={newBarangSku} class="modal-input" />
                <button class="btn-3d pink w-full" on:click={addManualBarang}>Simpan</button>
                <div class="divider">OR</div>
                <label class="btn-3d blue w-full">Excel Import <input type="file" accept=".xlsx, .csv" on:change={handleExcelInput} hidden /></label>
            </div>
        </div>
    </div>
{/if}

<Marketing />
<Footer />

<style>
    :global(:root) { --pink-btn: #f472b6; --text-dark: #111827; --text-gray: #6b7280; --blue-btn: #3b82f6; }
    
    .hero { background: linear-gradient(135deg, #fff0f8 0%, #e0f2fe 100%); padding: 20px 20px 80px 20px; display: flex; flex-direction: column; align-items: center; text-align: center; }
    .hero-text h1 { font-size: clamp(2rem, 5vw, 3.5rem); font-weight: 800; margin-bottom: 20px; }
    
    .upload-card { background: white; width: 90%; max-width: 1000px; border-radius: 40px; box-shadow: 0 20px 40px rgba(0,0,0,0.08); padding: 40px; margin: 0 auto; transition: transform 0.3s; }
    
    .responsive-layout { display: flex; gap: 40px; align-items: flex-start; text-align: left; width: 100%; }
    .layout-left { flex: 1.2; padding-left: 10px; }
    .layout-right { flex: 0.8; background: #f9fafb; border-radius: 24px; min-height: 300px; display: flex; align-items: center; justify-content: center; border: 2px dashed #e5e7eb; }
    
    .form-stack { display: flex; flex-direction: column; gap: 15px; width: 100%; }
    .search-form { display: flex; gap: 10px; margin-bottom: 20px; }
    
    .url-input { width: 100%; padding: 14px 20px; border: 2px solid #e5e7eb; border-radius: 99px; outline: none; transition: border-color 0.3s; font-family: inherit; }
    .url-input:focus { border-color: var(--pink-btn); }

    /* Product Card di Kanan */
    .product-card { padding: 20px; text-align: center; width: 100%; }
    .sku-tag { background: #f3f4f6; padding: 4px 10px; border-radius: 99px; font-size: 0.75rem; font-weight: 700; color: var(--text-gray); }
    .name { font-size: 1.2rem; font-weight: 800; margin: 10px 0 20px; color: var(--text-dark); }
    .stock-circle { background: white; border: 2px solid var(--pink-btn); width: 120px; height: 120px; border-radius: 50%; margin: 0 auto; display: flex; flex-direction: column; align-items: center; justify-content: center; box-shadow: 0 8px 15px rgba(244, 114, 182, 0.1); }
    .stock-val { font-size: 2.2rem; font-weight: 900; color: var(--pink-btn); line-height: 1; }
    .stock-unit { font-size: 0.8rem; font-weight: 700; color: var(--text-gray); }
    .sync-info { margin-top: 15px; font-size: 0.75rem; color: #10b981; font-weight: 700; display: flex; align-items: center; justify-content: center; gap: 5px; }
    .dot { width: 6px; height: 6px; background: #10b981; border-radius: 50%; display: inline-block; animation: pulse 1.5s infinite; }

    .btn-3d { padding: 12px 24px; border: 2px solid #111827; border-radius: 12px; font-weight: bold; background: white; cursor: pointer; box-shadow: 4px 4px 0px #111827; transition: all 0.1s; text-align: center; }
    .btn-3d.pink { background: var(--pink-btn); color: white; border-color: #be185d; box-shadow: 4px 4px 0px #be185d; }
    .btn-3d.blue { background: var(--blue-btn); color: white; border-color: #1e3a8a; box-shadow: 4px 4px 0px #1e3a8a; }
    .btn-3d:active { transform: translate(4px, 4px); box-shadow: 0px 0px 0px #111827; }
    .w-full { width: 100%; }

    .btn-master { width: 100%; margin-top: 20px; padding: 12px; background: #fffbeb; color: #854d0e; border: 2px solid #ca8a04; border-radius: 12px; font-weight: 800; cursor: pointer; }

    .mode-toggle { display: flex; gap: 8px; margin-bottom: 12px; }
    .mode-toggle label { flex: 1; text-align: center; padding: 8px; border: 2px solid #e5e7eb; border-radius: 10px; cursor: pointer; font-size: 0.85rem; font-weight: 700; }
    .mode-toggle label.active { border-color: var(--pink-btn); background: #fdf2f8; color: var(--pink-btn); }

    .placeholder-icon { font-size: 3rem; color: #d1d5db; }
    .preview-media { max-width: 100%; max-height: 250px; border-radius: 12px; }

    .modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 1000; }
    .modal-box { background: white; padding: 30px; border-radius: 20px; width: 90%; max-width: 400px; }
    .modal-input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px; margin-bottom: 10px; }

    @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }

    /* RESPONSIVE HP */
    @media (max-width: 768px) {
        .responsive-layout { flex-direction: column; gap: 20px; }
        .layout-left { padding-left: 0; width: 100%; }
        .layout-right { width: 100%; min-height: 250px; order: -1; } /* Info Produk pindah ke atas di HP */
        .upload-card { padding: 25px; width: 95%; border-radius: 25px; }
        .hero-text h1 { font-size: 2rem; }
        .stock-circle { width: 100px; height: 100px; }
        .stock-val { font-size: 1.8rem; }
    }
</style>