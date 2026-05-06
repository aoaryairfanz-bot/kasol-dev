<script>
    import { fade, slide } from 'svelte/transition';

    let inventoryList = [];
    let isLoading = false;
    let errorMessage = "";
    let searchQuery = "";
    let sortBy = "newest"; // State untuk urutan data

    const API_BASE = "https://aryairfanz-kasol-dev.hf.space";

    async function fetchDataAPI() {
        isLoading = true;
        errorMessage = "";
        try {
            const response = await fetch(`${API_BASE}/api/stock/all-data-json`);
            if (!response.ok) throw new Error("Gagal terhubung ke server.");
            
            inventoryList = await response.json();
        } catch (err) {
            errorMessage = err.message;
        } finally {
            isLoading = false;
        }
    }

    // PEMANGGILAN LANGSUNG (Tanpa onMount)
    // Script akan langsung menarik data dari API sesaat setelah file dibaca
    fetchDataAPI();

    // Reaktif: Menggabungkan Filter Pencarian & Pengurutan Data
    $: filteredData = inventoryList
        .filter(item => 
            item.sku.toLowerCase().includes(searchQuery.toLowerCase()) || 
            item.nama.toLowerCase().includes(searchQuery.toLowerCase())
        )
        .sort((a, b) => {
            if (sortBy === "highest") {
                return b.stok - a.stok; // Urutkan dari stok paling banyak
            }
            return 0; // Pertahankan urutan asli dari API (Data terbaru)
        });
</script>

<section class="inventory-section">
    <div class="container">
        
        <div class="header-row" in:fade>
            <div class="header-text">
                <h2 class="title">Database Live Gudang</h2>
                <p class="subtitle">Menampilkan 100 data pembaruan terakhir.</p>
            </div>
            
            <div class="action-group">
                <!-- Dropdown Filter Urutan -->
                <select bind:value={sortBy} class="sort-select">
                    <option value="newest">Terbaru Diinput</option>
                    <option value="highest">Stok Terbanyak</option>
                </select>

                <div class="search-wrapper">
                    <span class="search-icon">
                        <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="11" cy="11" r="8"></circle><line x1="21" y1="21" x2="16.65" y2="16.65"></line></svg>
                    </span>
                    <input 
                        type="text" 
                        bind:value={searchQuery}
                        placeholder="Cari SKU atau Nama..." 
                        class="search-input"
                    />
                </div>
                <button on:click={fetchDataAPI} class="btn-outline" title="Muat Ulang Data">
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polyline points="23 4 23 10 17 10"></polyline><polyline points="1 20 1 14 7 14"></polyline><path d="M3.51 9a9 9 0 0 1 14.85-3.36L23 10M1 14l4.64 4.36A9 9 0 0 0 20.49 15"></path></svg>
                    <span>Refresh</span>
                </button>
            </div>
        </div>

        <div class="table-card">
            {#if isLoading}
                <div class="state-box" in:fade>
                    <div class="spinner"></div>
                    <p class="state-text">Menyinkronkan data...</p>
                </div>
            
            {:else if errorMessage}
                <div class="state-box" in:slide>
                    <div class="error-badge">
                        <svg xmlns="http://www.w3.org/2000/svg" width="18" height="18" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><circle cx="12" cy="12" r="10"></circle><line x1="12" y1="8" x2="12" y2="12"></line><line x1="12" y1="16" x2="12.01" y2="16"></line></svg>
                        Kesalahan Sistem: {errorMessage}
                    </div>
                </div>
            
            {:else if filteredData.length === 0}
                <div class="state-box">
                    <div class="empty-icon">
                        <svg xmlns="http://www.w3.org/2000/svg" width="48" height="48" viewBox="0 0 24 24" fill="none" stroke="#d1d5db" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"><rect x="2" y="3" width="20" height="14" rx="2" ry="2"></rect><line x1="8" y1="21" x2="16" y2="21"></line><line x1="12" y1="17" x2="12" y2="21"></line></svg>
                    </div>
                    <p class="state-text">Tidak ada data yang sesuai dengan kriteria.</p>
                </div>
            
            {:else}
                <div class="table-responsive">
                    <table class="data-table">
                        <thead>
                            <tr>
                                <th style="width: 25%;">Nomor SKU</th>
                                <th style="width: 50%;">Nama Produk</th>
                                <th style="width: 25%; text-align: center;">Total Fisik</th>
                            </tr>
                        </thead>
                        <tbody>
                            {#each filteredData as item (item.sku)}
                                <tr transition:fade>
                                    <td><span class="sku-text">{item.sku}</span></td>
                                    <td><div class="product-name">{item.nama}</div></td>
                                    <td style="text-align: center;">
                                        {#if item.stok > 10}
                                            <span class="stock-status status-safe">{item.stok} Pcs</span>
                                        {:else if item.stok > 0}
                                            <span class="stock-status status-warn">{item.stok} Pcs</span>
                                        {:else}
                                            <span class="stock-status status-danger">Habis</span>
                                        {/if}
                                    </td>
                                </tr>
                            {/each}
                        </tbody>
                    </table>
                </div>
            {/if}
        </div>
    </div>
</section>

<style>
    /* --- Layout Dasar --- */
    .inventory-section {
        padding: 4rem 1rem;
        background-color: #f9fafb;
        min-height: auto;
        border-top: 1px solid #e5e7eb;
        font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
    }
    
    .container {
        max-width: 1152px;
        margin: 0 auto;
    }

    /* --- Header Profesional --- */
    .header-row {
        display: flex;
        flex-direction: column;
        justify-content: space-between;
        align-items: flex-start;
        margin-bottom: 2rem;
        gap: 1.5rem;
    }
    
    @media (min-width: 768px) {
        .header-row {
            flex-direction: row;
            align-items: flex-end;
        }
    }

    .title {
        font-size: 1.75rem;
        font-weight: 800;
        color: #111827;
        margin: 0 0 0.5rem 0;
        letter-spacing: -0.025em;
    }

    .subtitle {
        color: #6b7280;
        font-size: 0.95rem;
        margin: 0;
    }

    .action-group {
        display: flex;
        align-items: center;
        gap: 1rem;
        width: 100%;
        flex-wrap: wrap; /* Agar rapi jika layar menyempit */
    }
    
    @media (min-width: 768px) {
        .action-group {
            width: auto;
            flex-wrap: nowrap;
        }
    }

    /* --- Input & Dropdown Select --- */
    .sort-select {
        padding: 0.65rem 1rem;
        border: 1px solid #d1d5db;
        border-radius: 8px;
        outline: none;
        font-size: 0.9rem;
        color: #111827;
        background-color: white;
        cursor: pointer;
        font-family: inherit;
        transition: all 0.2s;
        min-width: 160px;
    }

    .sort-select:focus {
        border-color: #3b82f6;
        box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }

    .search-wrapper {
        position: relative;
        flex-grow: 1;
        width: 100%;
    }
    
    @media (min-width: 768px) {
        .search-wrapper {
            width: 300px;
            flex-grow: 0;
        }
    }

    .search-icon {
        position: absolute;
        left: 1rem;
        top: 50%;
        transform: translateY(-50%);
        color: #9ca3af;
        display: flex;
        align-items: center;
    }

    .search-input {
        width: 100%;
        padding: 0.65rem 1rem 0.65rem 2.75rem;
        border: 1px solid #d1d5db;
        border-radius: 8px;
        outline: none;
        font-size: 0.9rem;
        color: #111827;
        box-sizing: border-box;
        transition: all 0.2s;
        font-family: inherit;
    }

    .search-input:focus {
        border-color: #3b82f6;
        box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
    }

    .btn-outline {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        padding: 0.65rem 1.25rem;
        background: white;
        border: 1px solid #d1d5db;
        border-radius: 8px;
        color: #374151;
        font-weight: 600;
        font-size: 0.9rem;
        cursor: pointer;
        transition: all 0.2s;
        white-space: nowrap;
    }

    .btn-outline:hover {
        background: #f3f4f6;
        color: #111827;
    }

    /* --- Tabel Data --- */
    .table-card {
        background: white;
        border-radius: 12px;
        border: 1px solid #e5e7eb;
        box-shadow: 0 1px 3px rgba(0,0,0,0.05);
        overflow: hidden;
    }

    .table-responsive {
        overflow-x: auto;
    }

    .data-table {
        width: 100%;
        text-align: left;
        border-collapse: collapse;
        min-width: 600px;
    }

    .data-table th {
        padding: 1rem 1.5rem;
        font-size: 0.75rem;
        font-weight: 700;
        color: #4b5563;
        text-transform: uppercase;
        background: #f9fafb;
        border-bottom: 1px solid #e5e7eb;
    }

    .data-table td {
        padding: 1rem 1.5rem;
        border-bottom: 1px solid #e5e7eb;
        vertical-align: middle;
    }

    .data-table tbody tr {
        transition: background-color 0.15s;
    }

    .data-table tbody tr:hover {
        background-color: #f8fafc;
    }

    /* --- Tipografi Data --- */
    .sku-text {
        font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace;
        font-size: 0.85rem;
        color: #4b5563;
        font-weight: 500;
    }

    .product-name {
        font-weight: 600;
        color: #111827;
        font-size: 0.95rem;
    }

    /* --- Status Indikator Minimalis --- */
    .stock-status {
        display: inline-block;
        padding: 0.25rem 0.75rem;
        border-radius: 999px;
        font-size: 0.8rem;
        font-weight: 600;
        border: 1px solid transparent;
    }

    .status-safe {
        background-color: #f0fdf4;
        color: #15803d;
        border-color: #bbf7d0;
    }

    .status-warn {
        background-color: #fefce8;
        color: #a16207;
        border-color: #fef08a;
    }

    .status-danger {
        background-color: #fef2f2;
        color: #b91c1c;
        border-color: #fecaca;
    }

    /* --- States (Loading & Error) --- */
    .state-box {
        padding: 4rem;
        text-align: center;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
    }

    .spinner {
        width: 2rem;
        height: 2rem;
        border: 3px solid #f3f4f6;
        border-top-color: #3b82f6;
        border-radius: 50%;
        animation: spin 0.8s linear infinite;
        margin-bottom: 1rem;
    }

    @keyframes spin {
        to { transform: rotate(360deg); }
    }

    .state-text {
        color: #6b7280;
        font-size: 0.9rem;
        margin: 0;
    }

    .error-badge {
        display: inline-flex;
        align-items: center;
        gap: 0.5rem;
        background: #fef2f2;
        color: #991b1b;
        padding: 0.75rem 1.25rem;
        border-radius: 8px;
        font-size: 0.9rem;
        font-weight: 500;
        border: 1px solid #fecaca;
    }

    .empty-icon {
        margin-bottom: 1rem;
    }
</style>