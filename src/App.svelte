<script>
    import { fade, fly, slide, scale } from 'svelte/transition';
    import { quintOut, cubicOut } from 'svelte/easing';
    import { tweened } from 'svelte/motion';
    import * as XLSX from 'xlsx'; // Library untuk generate file Excel .xlsx
    
    import Header from './lib/Header.svelte';
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

    let itemId = ""; 
    let internalId = null; 
    let currentStock = null;
    let productDetails = null; 
    let qtyInput = "";
    let stockMode = "add"; 

    let showMasterModal = false;
    let newBarangNama = "";
    let newBarangSku = "";
    let isAddingManual = false;
    let excelFile = null;
    let isImporting = false;
    let masterMessage = ""; 
    let isMasterError = false;

    // State Tabel Inventaris
    let inventoryList = [];
    let isTableLoading = false;
    let isDataFetched = false;
    let isTableVisible = false; // Pengendali Hide/Show Tabel
    let currentPage = 1;
    let itemsPerPage = 10;
    let searchTableQuery = "";

    // State Modal Download
    let showDownloadModal = false;
    let downloadFileName = "Data_Gudang";
    let downloadFormat = "xlsx"; // Default format Excel

    let previewImage = null;
    let previewTimer; 
    const downloadProgress = tweened(0, { duration: 400, easing: cubicOut });
    let progressInterval;

    // State Login Teks
    let showLoginModal = false; // Tambahkan variabel ini di App.svelte

    const API_BASE = "https://aryairfanz-kasol-dev.hf.space";

    const pageContent = {
        'remove-bg': { title: "Remove Background", desc: "Remove image backgrounds in seconds.", formats: "JPG, PNG, or WebP - Up to 20MB", btnText: "Upload Image", accept: "image/*" },
        'stock-opname': { title: "Stock Opname Gudang", desc: "Input stok barang secara real-time.", formats: "Masukkan SKU Barang", btnText: "Cek Stok Barang", accept: "" },
        'download-video': { title: "Video Downloader", desc: "Download video YouTube, TikTok, atau Instagram.", formats: "Paste link video di bawah", btnText: "Download Video", accept: "" }
    };

    $: if (activeTab === 'download-video') {
        if (!videoUrl || videoUrl.trim() === "") { previewImage = null; } 
        else { clearTimeout(previewTimer); previewTimer = setTimeout(() => { fetchThumbnailPreview(videoUrl); }, 300); }
    }

    async function fetchThumbnailPreview(url) {
        if (resultType === 'video') { resultUrl = null; resultType = null; }
        previewImage = null;
        const ytMatch = url.match(/(?:youtu\.be\/|youtube\.com\/(?:embed\/|v\/|shorts\/|watch\?v=|watch\?.+&v=))((\w|-){11})/);
        if (ytMatch && ytMatch[1]) { previewImage = `https://img.youtube.com/vi/${ytMatch[1]}/hqdefault.jpg`; return; }
        if (url.includes('tiktok.com')) {
            try {
                const response = await fetch(`https://www.tiktok.com/oembed?url=${encodeURIComponent(url)}`);
                if (response.ok) { const data = await response.json(); if (data.thumbnail_url) { previewImage = data.thumbnail_url; return; } }
            } catch (err) { console.log("Gagal preview", err); }
        }
    }

    function handleDragOver(e) { e.preventDefault(); isDragging = true; }
    function handleDragLeave() { isDragging = false; }
    function handleDrop(e) { e.preventDefault(); isDragging = false; if (e.dataTransfer.files && e.dataTransfer.files.length > 0) handleFileSelect(e.dataTransfer.files[0]); }
    function handleFileInput(e) { if (e.target.files && e.target.files.length > 0) handleFileSelect(e.target.files[0]); }

    function handleFileSelect(selectedFile) {
        if (activeTab === 'remove-bg' && !selectedFile.type.startsWith('image/')) { errorMessage = "Harap upload gambar!"; return; }
        file = selectedFile; processFile();
    }

    function startProgress() {
        downloadProgress.set(0, {duration: 0});
        progressInterval = setInterval(() => {
            downloadProgress.update(n => { if (n >= 90) { clearInterval(progressInterval); return 90; } return n + (Math.random() * 8); });
        }, 800);
    }

    $: activeTab, resetStateOnTabChange();
    function resetStateOnTabChange() {
        file = null; videoUrl = ""; resultUrl = null; errorMessage = ""; successMessage = ""; isLoading = false; previewImage = null;
        itemId = ""; internalId = null; currentStock = null; productDetails = null; qtyInput = ""; stockMode = "add";
        downloadProgress.set(0, {duration: 0}); clearInterval(progressInterval);
        isDataFetched = false; isTableVisible = false; inventoryList = []; currentPage = 1; searchTableQuery = "";
    }

    async function processFile() {
        isLoading = true; errorMessage = ""; resultUrl = null;
        const formData = new FormData(); formData.append("file", file);
        try {
            const response = await fetch(`${API_BASE}/remove-bg`, { method: "POST", body: formData });
            if (!response.ok) throw new Error("Gagal memproses file.");
            const blob = await response.blob(); resultUrl = URL.createObjectURL(blob); resultType = 'image';
        } catch (err) { errorMessage = err.message; } finally { isLoading = false; }
    }

    async function processVideo() {
        if (!videoUrl) { errorMessage = "Masukkan link video dulu!"; return; }
        isLoading = true; errorMessage = ""; resultUrl = null; startProgress(); 
        const formData = new FormData(); formData.append("url", videoUrl);
        try {
            const response = await fetch(`${API_BASE}/download-video`, { method: "POST", body: formData });
            if (!response.ok) throw new Error("Gagal mendownload.");
            const blob = await response.blob(); resultUrl = URL.createObjectURL(blob); resultType = 'video';
            clearInterval(progressInterval); await downloadProgress.set(100, { duration: 200 }); 
            const contentDisposition = response.headers.get('Content-Disposition');
            let filename = "video_terunduh.mp4";
            if (contentDisposition && contentDisposition.includes('filename=')) filename = contentDisposition.split('filename=')[1].replace(/"/g, '');
            triggerDownload(resultUrl, filename);
        } catch (err) { clearInterval(progressInterval); downloadProgress.set(0, {duration: 0}); errorMessage = err.message; } finally { isLoading = false; }
    }

    async function fetchCurrentStock() {
        if (!itemId) return;
        isLoading = true; errorMessage = ""; successMessage = ""; currentStock = null; productDetails = null; internalId = null;
        const skuPencarian = itemId.trim().toUpperCase();
        try {
            const response = await fetch(`${API_BASE}/api/stock/${skuPencarian}`);
            const data = await response.json();
            if (response.ok) {
                currentStock = data.current_total; internalId = data.id; productDetails = { nama: data.nama, sku: data.sku };
                itemId = skuPencarian; 
            } else { errorMessage = data.detail || "Barang tidak ditemukan"; }
        } catch (err) { errorMessage = "Gagal terhubung ke database."; }
        finally { isLoading = false; }
    }

    async function submitStock() {
        if (qtyInput === "" || qtyInput === null || !internalId) return;
        isLoading = true; errorMessage = ""; successMessage = "";
        const payload = { barang_id: internalId, user_id: loggedInUser.id, action: stockMode, input_qty: parseInt(qtyInput) };
        try {
            const response = await fetch(`${API_BASE}/api/stock/input`, {
                method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify(payload)
            });
            if (response.ok) {
                successMessage = "Stok berhasil diperbarui!";
                await fetchCurrentStock(); qtyInput = "";
                if (isDataFetched && isTableVisible) fetchTableData(); 
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
            if (response.ok) { masterMessage = data.message; isMasterError = false; newBarangNama = ""; newBarangSku = ""; if (isDataFetched && isTableVisible) fetchTableData(); } 
            else { throw new Error(data.detail || "Gagal menambah barang."); }
        } catch (err) { masterMessage = err.message; isMasterError = true; } 
        finally { isAddingManual = false; }
    }

    function handleExcelInput(e) { if (e.target.files && e.target.files.length > 0) { excelFile = e.target.files[0]; uploadExcel(); } }

    async function uploadExcel() {
        if (!excelFile) return;
        isImporting = true; masterMessage = ""; const formData = new FormData();
        formData.append("file", excelFile); formData.append("user_id", loggedInUser.id);
        try {
            const response = await fetch(`${API_BASE}/api/stock/import-barang`, { method: "POST", body: formData });
            const data = await response.json();
            if (response.ok) { masterMessage = data.message; isMasterError = false; if (isDataFetched && isTableVisible) fetchTableData(); } 
            else { throw new Error(data.detail || "Gagal mengimport file."); }
        } catch (err) { masterMessage = err.message; isMasterError = true; } 
        finally { isImporting = false; excelFile = null; }
    }

    function triggerDownload(url, filename) {
        const a = document.createElement('a'); a.href = url; a.download = filename || 'download';
        document.body.appendChild(a); a.click(); document.body.removeChild(a);
    }

    async function toggleTable() {
        if (!isTableVisible) {
            isTableVisible = true;
            if (!isDataFetched) await fetchTableData();
        } else {
            isTableVisible = false;
        }
    }

    async function fetchTableData() {
        isTableLoading = true;
        try {
            const response = await fetch(`${API_BASE}/api/stock/all-data-json`);
            if (!response.ok) throw new Error("Gagal mengambil data dari server");
            inventoryList = await response.json();
            isDataFetched = true; currentPage = 1;
        } catch (err) { console.error("Gagal load tabel", err); } 
        finally { isTableLoading = false; }
    }

    // Fungsi Proses Download Berdasarkan Format
    function processDownloadData() {
        if (!inventoryList || inventoryList.length === 0) return;

        let finalName = downloadFileName.trim() === "" ? "Data_Gudang" : downloadFileName.trim();

        if (downloadFormat === 'csv') {
            // Logika CSV
            const headers = "SKU/ID,Nama,Total Stok,User Terakhir\n";
            const rows = inventoryList.map(item => `${item.sku},"${item.nama}",${item.stok},${item.user}`).join("\n");
            const blob = new Blob([headers + rows], { type: 'text/csv;charset=utf-8;' });
            triggerDownload(URL.createObjectURL(blob), `${finalName}.csv`);
        } else {
            // Logika Excel (.xlsx) menggunakan SheetJS
            const wsData = inventoryList.map(item => ({
                "SKU/ID": item.sku,
                "Nama": item.nama,
                "Total Stok": item.stok,
                "User Terakhir": item.user
            }));
            const ws = XLSX.utils.json_to_sheet(wsData);
            const wb = XLSX.utils.book_new();
            XLSX.utils.book_append_sheet(wb, ws, "Data Inventaris");
            XLSX.writeFile(wb, `${finalName}.xlsx`);
        }

        showDownloadModal = false; // Tutup popup setelah download
    }

    $: filteredInventory = inventoryList.filter(item => 
        item.sku.toLowerCase().includes(searchTableQuery.toLowerCase()) || 
        item.nama.toLowerCase().includes(searchTableQuery.toLowerCase())
    );

    $: if (searchTableQuery !== undefined) { currentPage = 1; }
    $: totalPages = Math.ceil(filteredInventory.length / itemsPerPage) || 1;
    $: paginatedData = filteredInventory.slice((currentPage - 1) * itemsPerPage, currentPage * itemsPerPage);

    function changePage(delta) { currentPage += delta; if (currentPage < 1) currentPage = 1; if (currentPage > totalPages) currentPage = totalPages; }
    function handleItemsPerPageChange() { currentPage = 1; }
</script>

<Header bind:activeTab={activeTab} bind:loggedInUser={loggedInUser} bind:showLoginModal={showLoginModal} />

<section class="hero">
    <div class="breadcrumb">Home / Tools / {pageContent[activeTab].title}</div>
    
    {#key activeTab}
        <div class="hero-text" in:fly={{ y: 20, duration: 400, delay: 200, easing: quintOut }} out:fade={{ duration: 150 }}>
            <h1>{pageContent[activeTab].title}</h1>
            <p>{pageContent[activeTab].desc}</p>
        </div>
    {/key}

    <div class="upload-card {isDragging ? 'drag-active' : ''}">
        {#if errorMessage} <div class="error-msg" transition:slide={{ duration: 300 }}>{errorMessage}</div> {/if}
        {#if successMessage} <div class="success-msg" transition:slide={{ duration: 300 }}>{successMessage}</div> {/if}

        {#if isLoading && activeTab === 'remove-bg'}
            <div class="loading-state" in:fade={{ duration: 200 }}><div class="spinner"></div><h2>Memproses...</h2></div>
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
                                    {#if isLoading || $downloadProgress > 0} <div class="progress-bar-bg"><div class="progress-bar-fill" style="width: {$downloadProgress}%"></div></div> {/if}
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
                                    <div class="login-alert">
                                        Akses terbatas. Silakan <span class="login-link" on:click={() => showLoginModal = true}>Login</span>.
                                    </div>
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
                                            <button class="btn-3d blue w-full" on:click={submitStock} disabled={isLoading || qtyInput === "" || qtyInput === null}>Simpan</button>
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
                                        </div>
                                        <div class="sync-info"><span class="dot"></span> Sync Aktif</div>
                                    </div>
                                {:else}
                                    <div class="placeholder-icon">📦</div>
                                {/if}
                            </div>
                        </div>

                        {#if loggedInUser}
                            <div class="table-container" in:fade>
                                <div class="table-header-flex">
                                    <h3>Data Inventaris</h3>
                                    
                                    <div class="table-controls">
                                        {#if isTableVisible}
                                            <input type="text" bind:value={searchTableQuery} placeholder="Cari SKU/Nama..." class="table-search-input" />
                                            <select class="select-dropdown" bind:value={itemsPerPage} on:change={handleItemsPerPageChange}>
                                                <option value={10}>10 Baris</option>
                                                <option value={15}>15 Baris</option>
                                                <option value={50}>50 Baris</option>
                                                <option value={100}>100 Baris</option>
                                                <option value={300}>300 Baris</option>
                                                <option value={500}>500 Baris</option>
                                                <option value={999999}>Semua Data</option>
                                            </select>
                                        {/if}

                                        <button class="btn-3d {isTableVisible ? 'blue' : 'pink'} btn-small" on:click={toggleTable} disabled={isTableLoading}>
                                            {isTableLoading ? 'Memuat...' : (isTableVisible ? 'Hide Data' : 'Show Data')}
                                        </button>

                                        <div class="btn-wrapper" title={loggedInUser.role !== 'superadmin' ? 'Hanya Superadmin yang dapat mengunduh data.' : 'Download ke Excel/CSV'}>
                                            <button class="btn-3d green btn-small" 
                                                    on:click={() => { downloadFileName = "Data_Gudang"; showDownloadModal = true; }} 
                                                    disabled={loggedInUser.role !== 'superadmin'}>
                                                Download Data
                                            </button>
                                        </div>
                                    </div>
                                </div>

                                {#if isTableVisible}
                                    <div class="table-wrapper" in:slide={{ duration: 300 }}>
                                        <table class="data-table">
                                            <thead>
                                                <tr>
                                                    <th style="width: 20%;">SKU/ID</th>
                                                    <th style="width: 45%;">Nama</th>
                                                    <th style="width: 15%; text-align: center;">Total Stok</th>
                                                    <th style="width: 20%; text-align: center;">User Terakhir</th>
                                                </tr>
                                            </thead>
                                            <tbody>
                                                {#if filteredInventory.length === 0}
                                                    <tr><td colspan="4" class="empty-td">Data tidak ditemukan.</td></tr>
                                                {:else}
                                                    {#each paginatedData as item (item.sku)}
                                                        <tr>
                                                            <td class="font-mono text-blue">{item.sku}</td>
                                                            <td class="font-bold text-dark">{item.nama}</td>
                                                            <td style="text-align: center;">
                                                                <span class="stock-badge {item.stok > 0 ? 'safe' : 'danger'}">{item.stok}</span>
                                                            </td>
                                                            <td style="text-align: center;">
                                                                <span class="user-badge">{item.user}</span>
                                                            </td>
                                                        </tr>
                                                    {/each}
                                                {/if}
                                            </tbody>
                                        </table>
                                    </div>

                                    {#if isDataFetched && filteredInventory.length > 0}
                                        <div class="pagination-bar" in:fade>
                                            <button class="page-btn" disabled={currentPage === 1} on:click={() => changePage(-1)}>Previous</button>
                                            <span class="page-info">Halaman {currentPage} dari {totalPages}</span>
                                            <button class="page-btn" disabled={currentPage === totalPages} on:click={() => changePage(1)}>Next</button>
                                        </div>
                                    {/if}
                                {/if}
                            </div>
                        {/if}

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

{#if showDownloadModal}
    <div class="modal-overlay" on:click|self={() => showDownloadModal = false}>
        <div class="modal-box" transition:scale>
            <h2>Download File</h2>
            <p class="sub-text">Pilih format dan beri nama file sebelum mendownload agar tidak terduplikat.</p>
            
            <div class="form-group">
                <label class="input-label">Nama File</label>
                <input type="text" placeholder="Contoh: Laporan_Bulan_Ini" bind:value={downloadFileName} class="modal-input" />
                
                <label class="input-label" style="margin-top: 15px;">Format File</label>
                <select bind:value={downloadFormat} class="modal-input select-dropdown w-full">
                    <option value="xlsx">Excel (.xlsx) - Rekomendasi</option>
                    <option value="csv">CSV (.csv)</option>
                </select>

                <div class="btn-group-modal">
                    <button class="btn-3d outline w-full" on:click={() => showDownloadModal = false}>Batal</button>
                    <button class="btn-3d green w-full" on:click={processDownloadData}>Download</button>
                </div>
            </div>
        </div>
    </div>
{/if}

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

<Footer />

<style>
    :global(:root) { --pink-btn: #f472b6; --text-dark: #111827; --text-gray: #6b7280; --blue-btn: #3b82f6; --green-btn: #10b981; }
    
    .hero { background: linear-gradient(135deg, #fff0f8 0%, #e0f2fe 100%); padding: 20px 20px 80px 20px; display: flex; flex-direction: column; align-items: center; text-align: center; }
    .hero-text h1 { font-size: clamp(2rem, 5vw, 3.5rem); font-weight: 800; margin-bottom: 20px; }
    
    .upload-card { background: white; width: 90%; max-width: 1000px; border-radius: 40px; box-shadow: 0 20px 40px rgba(0,0,0,0.08); padding: 40px; margin: 0 auto; }
    
    .responsive-layout { display: flex; gap: 40px; align-items: flex-start; text-align: left; width: 100%; }
    .layout-left { flex: 1.2; padding-left: 10px; }
    .layout-right { flex: 0.8; background: #f9fafb; border-radius: 24px; min-height: 300px; display: flex; align-items: center; justify-content: center; border: 2px dashed #e5e7eb; }
    
    .form-stack { display: flex; flex-direction: column; gap: 15px; width: 100%; }
    .search-form { display: flex; gap: 10px; margin-bottom: 20px; }
    
    .url-input { width: 100%; padding: 14px 20px; border: 2px solid #e5e7eb; border-radius: 99px; outline: none; transition: border-color 0.3s; font-family: inherit; }
    .url-input:focus { border-color: var(--pink-btn); }

    .product-card { padding: 20px; text-align: center; width: 100%; }
    .sku-tag { background: #f3f4f6; padding: 4px 10px; border-radius: 99px; font-size: 0.75rem; font-weight: 700; color: var(--text-gray); }
    .name { font-size: 1.2rem; font-weight: 800; margin: 10px 0 20px; color: var(--text-dark); }
    .stock-circle { background: white; border: 2px solid var(--pink-btn); width: 120px; height: 120px; border-radius: 50%; margin: 0 auto; display: flex; flex-direction: column; align-items: center; justify-content: center; box-shadow: 0 8px 15px rgba(244, 114, 182, 0.1); }
    .stock-val { font-size: 2.2rem; font-weight: 900; color: var(--pink-btn); line-height: 1; }
    .stock-unit { font-size: 0.8rem; font-weight: 700; color: var(--text-gray); }
    .sync-info { margin-top: 15px; font-size: 0.75rem; color: var(--green-btn); font-weight: 700; display: flex; align-items: center; justify-content: center; gap: 5px; }
    .dot { width: 6px; height: 6px; background: var(--green-btn); border-radius: 50%; display: inline-block; animation: pulse 1.5s infinite; }

    .btn-3d { padding: 12px 24px; border: 2px solid #111827; border-radius: 12px; font-weight: bold; background: white; cursor: pointer; box-shadow: 4px 4px 0px #111827; transition: all 0.1s; text-align: center; }
    .btn-3d.pink { background: var(--pink-btn); color: white; border-color: #be185d; box-shadow: 4px 4px 0px #be185d; }
    .btn-3d.blue { background: var(--blue-btn); color: white; border-color: #1e3a8a; box-shadow: 4px 4px 0px #1e3a8a; }
    .btn-3d.green { background: var(--green-btn); color: white; border-color: #047857; box-shadow: 4px 4px 0px #047857; }
    .btn-3d.outline { background: white; color: var(--text-dark); border-color: #d1d5db; box-shadow: 4px 4px 0px #d1d5db; }
    
    .btn-3d:not(:disabled):active { transform: translate(4px, 4px); box-shadow: 0px 0px 0px transparent; }
    
    .btn-wrapper { display: inline-block; }
    .btn-3d:disabled { opacity: 0.5; cursor: not-allowed; box-shadow: 2px 2px 0px rgba(0,0,0,0.5); transform: none; pointer-events: none; }
    
    .w-full { width: 100%; }
    .btn-small { padding: 8px 16px; font-size: 0.85rem; }
    .btn-master { width: 100%; margin-top: 20px; padding: 12px; background: #fffbeb; color: #854d0e; border: 2px solid #ca8a04; border-radius: 12px; font-weight: 800; cursor: pointer; }

    .mode-toggle { display: flex; gap: 8px; margin-bottom: 12px; }
    .mode-toggle label { flex: 1; text-align: center; padding: 8px; border: 2px solid #e5e7eb; border-radius: 10px; cursor: pointer; font-size: 0.85rem; font-weight: 700; }
    .mode-toggle label.active { border-color: var(--pink-btn); background: #fdf2f8; color: var(--pink-btn); }

    .placeholder-icon { font-size: 3rem; color: #d1d5db; }
    .preview-media { max-width: 100%; max-height: 250px; border-radius: 12px; }

    /* Modal Styling */
    .modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.5); display: flex; align-items: center; justify-content: center; z-index: 1000; }
    .modal-box { background: white; padding: 30px; border-radius: 20px; width: 90%; max-width: 400px; }
    .modal-input { width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px; margin-bottom: 10px; font-family: inherit; font-size: 0.9rem;}
    .sub-text { font-size: 0.85rem; color: var(--text-gray); margin-bottom: 20px; line-height: 1.4;}
    .input-label { display: block; font-size: 0.85rem; font-weight: 700; color: var(--text-dark); margin-bottom: 5px; text-align: left; }
    .btn-group-modal { display: flex; gap: 10px; margin-top: 15px; }

    /* TABEL INVENTARIS LEBIH KECIL & PLAIN TEXT */
    .table-container { margin-top: 40px; padding-top: 30px; border-top: 2px dashed #e5e7eb; text-align: left; }
    .table-header-flex { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; flex-wrap: wrap; gap: 15px; }
    .table-header-flex h3 { margin: 0; font-size: 1.5rem; font-weight: 800; color: var(--text-dark); }
    .table-controls { display: flex; gap: 10px; align-items: center; flex-wrap: wrap;}
    
    .table-search-input { padding: 8px 12px; border: 2px solid #e5e7eb; border-radius: 8px; font-family: inherit; font-weight: 500; font-size: 0.8rem; outline: none; color: var(--text-dark); min-width: 180px; transition: border-color 0.2s; }
    .table-search-input:focus { border-color: var(--pink-btn); }
    
    .select-dropdown { padding: 8px 10px; border: 2px solid #e5e7eb; border-radius: 8px; font-family: inherit; font-weight: bold; font-size: 0.85rem; outline: none; cursor: pointer; color: var(--text-dark); }
    .select-dropdown:focus { border-color: var(--pink-btn); }

    .table-wrapper { 
        overflow-x: auto; 
        overflow-y: auto; /* Mengaktifkan scroll vertikal */
        max-height: 550px; /* Batas tinggi tabel (kira-kira 15 baris) */
        border: 1px solid #e5e7eb; 
        border-radius: 12px; 
        background: white; 
        
        /* Menyembunyikan tampilan scrollbar secara visual */
        -ms-overflow-style: none;  /* Untuk IE & Edge */
        scrollbar-width: none;  /* Untuk Firefox */
    }
    
    /* Menyembunyikan scrollbar untuk Chrome, Safari, dan Opera */
    .table-wrapper::-webkit-scrollbar {
        display: none; 
    }

    .data-table { width: 100%; border-collapse: collapse; min-width: 600px; }
    
    .data-table th, .data-table td { padding: 8px 12px; border-bottom: 1px solid #f3f4f6; font-size: 0.75rem; }

    /* Membuat Teks Login */
    .login-alert {
        font-weight: 700;
        color: var(--text-gray);
        margin-top: 10px;
        font-size: 0.9rem;
    }

    .login-link {
        color: var(--pink-btn); /* Warna pink sesuai brand Baginda */
        cursor: pointer;
        text-decoration: underline;
        transition: opacity 0.2s;
    }

    .login-link:hover {
        opacity: 0.8;
    }
    
    /* Membuat Header Tabel menempel di atas (Sticky) */
    .data-table th { 
        position: sticky; 
        top: 0; 
        z-index: 10; /* Agar header tidak tertimpa baris data yang lewat */
        background: #f9fafb; 
        font-weight: 800; 
        color: var(--text-gray); 
        text-transform: uppercase; 
        font-size: 0.65rem; 
        box-shadow: 0 1px 0 #f3f4f6; /* Menggunakan shadow sebagai ganti border bottom */
    }

    .data-table tr:hover { background: #fdf2f8; }
    .empty-td { text-align: center; padding: 30px; color: var(--text-gray); font-weight: bold; }
    
    .font-mono { font-family: monospace; font-size: 0.75rem; }
    .text-blue { color: var(--blue-btn); font-weight: bold; }
    .font-bold { font-weight: 700; }
    .text-dark { color: var(--text-dark); }

    .stock-badge { font-weight: 800; font-size: 0.75rem; }
    .stock-badge.safe { color: #16a34a; }
    .stock-badge.danger { color: #dc2626; }
    .user-badge { color: #4b5563; font-weight: 600; font-size: 0.75rem; }

    .pagination-bar { display: flex; justify-content: space-between; align-items: center; margin-top: 15px; padding: 5px 0; }
    .page-info { font-weight: 700; color: var(--text-gray); font-size: 0.85rem; }
    .page-btn { padding: 6px 14px; border: 2px solid #111827; background: white; border-radius: 8px; font-weight: bold; cursor: pointer; box-shadow: 2px 2px 0px #111827; font-size: 0.8rem; transition: all 0.1s; }
    .page-btn:disabled { opacity: 0.4; cursor: not-allowed; box-shadow: none; transform: none; }
    .page-btn:not(:disabled):active { transform: translate(2px, 2px); box-shadow: 0px 0px 0px #111827; }

    @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }

    @media (max-width: 768px) {
        .responsive-layout { flex-direction: column; gap: 20px; }
        .layout-left { padding-left: 0; width: 100%; }
        .layout-right { width: 100%; min-height: 250px; order: -1; }
        .upload-card { padding: 25px; width: 95%; border-radius: 25px; }
        .hero-text h1 { font-size: 2rem; }
        .table-header-flex { flex-direction: column; align-items: flex-start; }
        .table-controls { width: 100%; justify-content: flex-start; }
    }
</style>