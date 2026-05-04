<script>
    import { fade, scale, slide } from 'svelte/transition';

    export let activeTab;
    export let loggedInUser = null; 

    let showLoginModal = false;
    let isMenuOpen = false; // State untuk Hamburger Menu
    let username = "";
    let password = "";
    let loginError = "";

    function switchTab(tab) {
        activeTab = tab;
        isMenuOpen = false; // Tutup menu setelah memilih tab
    }

    function toggleMenu() {
        isMenuOpen = !isMenuOpen;
    }

    async function handleLogin() {
        if (username.trim() === "" || password.trim() === "") {
            loginError = "Username dan Password wajib diisi!";
            return;
        }
        loginError = ""; 
        try {
            const response = await fetch("https://aryairfanz-kasol-dev.hf.space/api/auth/login", {
                method: "POST",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ username, password })
            });
            const data = await response.json();
            if (response.ok) {
                loggedInUser = data.user;
                showLoginModal = false;
                username = ""; password = "";
            } else {
                loginError = data.detail || "Terjadi kesalahan saat login.";
            }
        } catch (err) {
            loginError = "Gagal terhubung ke server.";
        }
    }

    function handleLogout() {
        loggedInUser = null;
        isMenuOpen = false;
    }
</script>

<header class="site-header">
    <div class="header-top">
        <div class="logo-search-container">
            <div class="logo">K<span>Dev</span></div>
            <div class="search-bar">
                <input type="text" placeholder="Search tools..." />
            </div>
        </div>

        <div class="header-actions">
            <!-- Hamburger Button (Hanya tampil di HP) -->
            <button class="hamburger" on:click={toggleMenu} aria-label="Toggle Menu">
                <span class:open={isMenuOpen}></span>
                <span class:open={isMenuOpen}></span>
                <span class:open={isMenuOpen}></span>
            </button>

            <!-- Actions Desktop -->
            <div class="desktop-actions">
                <span class="phone">@kasolll__</span>
                {#if loggedInUser}
                    <div class="user-badge">
                        <span class="role-tag {loggedInUser.role}">{loggedInUser.role}</span>
                        <button class="header-btn logout" on:click={handleLogout}>Logout</button>
                    </div>
                {:else}
                    <button class="header-btn login" on:click={() => showLoginModal = true}>Login</button>
                {/if}
            </div>
        </div>
    </div>

    <!-- Navigasi Desktop -->
    <nav class="main-nav desktop-nav">
        <button class:active={activeTab === 'remove-bg'} on:click={() => switchTab('remove-bg')}>Remove Background</button>
        <button class:active={activeTab === 'stock-opname'} on:click={() => switchTab('stock-opname')}>Stock Opname</button>
        <button class:active={activeTab === 'download-video'} on:click={() => switchTab('download-video')}>Video Downloader</button>
    </nav>

    <!-- Navigasi Mobile (Hamburger Menu) -->
    {#if isMenuOpen}
        <div class="mobile-menu" transition:slide>
            <nav class="mobile-nav">
                <button class:active={activeTab === 'remove-bg'} on:click={() => switchTab('remove-bg')}>Remove Background</button>
                <button class:active={activeTab === 'stock-opname'} on:click={() => switchTab('stock-opname')}>Stock Opname</button>
                <button class:active={activeTab === 'download-video'} on:click={() => switchTab('download-video')}>Video Downloader</button>
                
                <hr />
                
                {#if loggedInUser}
                    <div class="mobile-user-info">
                        <span class="role-tag {loggedInUser.role}">{loggedInUser.role}</span>
                        <button class="header-btn logout w-full" on:click={handleLogout}>Logout</button>
                    </div>
                {:else}
                    <button class="header-btn login w-full" on:click={() => {showLoginModal = true; isMenuOpen = false;}}>Login</button>
                {/if}
            </nav>
        </div>
    {/if}
</header>

<!-- MODAL LOGIN TETAP SAMA -->
{#if showLoginModal}
    <div class="modal-overlay" transition:fade={{ duration: 200 }} on:click|self={() => showLoginModal = false}>
        <div class="modal-box" transition:scale={{ duration: 250, start: 0.95 }}>
            <button class="close-btn" on:click={() => showLoginModal = false}>&times;</button>
            <div class="modal-header">
                <h2>Login Sistem</h2>
                <p>Silakan masuk untuk mengelola Stock Opname.</p>
            </div>
            {#if loginError}<div class="error-msg" transition:fade>{loginError}</div>{/if}
            <div class="form-group">
                <input type="text" placeholder="Username" bind:value={username} class="modal-input" />
                <input type="password" placeholder="Password" bind:value={password} class="modal-input" on:keydown={(e) => e.key === 'Enter' && handleLogin()} />
                <button class="btn-3d pink w-full" on:click={handleLogin}>Masuk Sistem</button>
            </div>
        </div>
    </div>
{/if}

<style>
    .site-header { border-bottom: 1px solid #eaeaea; position: sticky; top: 0; background: white; z-index: 100; box-shadow: 0 4px 15px rgba(0,0,0,0.03);}
    .header-top { display: flex; justify-content: space-between; align-items: center; padding: 15px 40px; gap: 20px; }
    
    .logo-search-container { display: flex; align-items: center; gap: 20px; flex-grow: 1; }
    .logo { font-size: 1.8rem; font-weight: 800; color: #f472b6; letter-spacing: -1px; white-space: nowrap; }
    .logo span { color: #111827; }
    
    .search-bar { display: flex; align-items: center; border: 1px solid #ccc; border-radius: 99px; padding: 8px 16px; width: 100%; max-width: 300px; }
    .search-bar input { border: none; outline: none; width: 100%; font-family: inherit;}

    .header-actions { display: flex; align-items: center; }
    .desktop-actions { display: flex; align-items: center; gap: 20px; font-size: 0.9rem; font-weight: 600; }

    /* Hamburger Style */
    .hamburger { display: none; flex-direction: column; gap: 5px; background: none; border: none; cursor: pointer; padding: 5px; z-index: 110; }
    .hamburger span { display: block; width: 25px; height: 3px; background: #111827; transition: 0.3s; border-radius: 2px; }
    .hamburger span.open:nth-child(1) { transform: translateY(8px) rotate(45deg); }
    .hamburger span.open:nth-child(2) { opacity: 0; }
    .hamburger span.open:nth-child(3) { transform: translateY(-8px) rotate(-45deg); }

    .main-nav { display: flex; gap: 30px; padding: 15px 40px; border-top: 1px solid #eaeaea; font-weight: 600; color: #6b7280;}
    .main-nav button { background: none; border: none; font-size: 1rem; cursor: pointer; color: inherit; font-family: inherit; transition: 0.2s;}
    .main-nav button.active { color: #111827; border-bottom: 2px solid #111827; padding-bottom: 5px;}

    /* --- PERBAIKAN MOBILE MENU --- */
    .mobile-menu { 
        background: white; 
        border-top: 1px solid #eaeaea; 
        padding: 15px 25px; /* Padding samping disesuaikan */
        box-shadow: 0 10px 15px rgba(0,0,0,0.05); 
    }

    .mobile-nav { 
        display: flex; 
        flex-direction: column; 
        gap: 5px; /* Jarak antar menu dirapatkan */
    }

    .mobile-nav button { 
        text-align: left; 
        background: none; 
        border: none; 
        font-size: 1rem; 
        font-weight: 700; 
        color: #6b7280; 
        padding: 12px 0; /* Area klik tetap nyaman */
        transition: color 0.2s;
    }

    .mobile-nav button.active { 
        color: #f472b6; 
    }

    hr { 
        border: none; 
        border-top: 1px solid #eee; 
        margin: 10px 0; 
    }

    .mobile-user-info { 
        display: flex; 
        flex-direction: column; 
        gap: 15px; 
        padding-top: 5px;
        align-items: flex-start; /* Agar badge role tidak melebar full */
    }

    /* Perbaikan tombol agar tidak penyok */
    .mobile-nav .header-btn {
        width: 100%;
        box-sizing: border-box; /* Sangat penting agar padding tidak merusak lebar */
        padding: 12px;
        font-size: 0.95rem;
        text-align: center;
        margin-top: 5px;
    }

    /* Responsif HP */
    @media (max-width: 768px) {
        .header-top { 
            padding: 10px 20px; /* Padding header dirampingkan */
            gap: 10px;
        }
        .logo { font-size: 1.4rem; }
        .search-bar { 
            max-width: 140px; 
            padding: 5px 12px; 
        }
        .search-bar input { font-size: 0.85rem; }
        .desktop-actions, .desktop-nav { display: none; }
        .hamburger { display: flex; }
    }

    /* Buttons */
    .header-btn { border: 2px solid #111827; background: white; padding: 8px 16px; border-radius: 99px; cursor: pointer; font-weight: bold; box-shadow: 2px 2px 0px #111827;}
    .header-btn.login { background: #f472b6; color: white; border-color: #be185d; box-shadow: 2px 2px 0px #be185d;}
    .header-btn.logout { background: #f3f4f6; color: #dc2626; border-color: #fca5a5; box-shadow: none; }
    .role-tag { font-size: 0.75rem; padding: 4px 10px; border-radius: 99px; font-weight: 800; text-transform: uppercase; text-align: center; width: fit-content;}
    .role-tag.superadmin { background: #fef08a; color: #854d0e; }
    .role-tag.gudang { background: #dbeafe; color: #1e40af; }
    .w-full { width: 100%; }

    /* Modal */
    .modal-overlay { position: fixed; top: 0; left: 0; width: 100vw; height: 100vh; background: rgba(17, 24, 39, 0.6); backdrop-filter: blur(4px); display: flex; justify-content: center; align-items: center; z-index: 9999; }
    .modal-box { background: white; width: 100%; max-width: 400px; padding: 40px 30px; border-radius: 24px; position: relative; }
    .close-btn { position: absolute; top: 15px; right: 20px; background: none; border: none; font-size: 1.8rem; cursor: pointer; color: #9ca3af; }
    .modal-input { width: 100%; padding: 15px 20px; border: 2px solid #e5e7eb; border-radius: 12px; margin-bottom: 15px; box-sizing: border-box;}
    .btn-3d { padding: 14px 24px; border: 2px solid #111827; border-radius: 12px; font-weight: bold; cursor: pointer; box-shadow: 4px 4px 0px #111827;}
    .btn-3d.pink { background: #f472b6; color: white; border-color: #be185d; box-shadow: 4px 4px 0px #be185d; }

    @media (max-width: 768px) {
        .header-top { padding: 15px 20px; }
        .logo { font-size: 1.4rem; }
        .search-bar { max-width: 160px; padding: 6px 12px; }
        .desktop-actions, .desktop-nav { display: none; }
        .hamburger { display: flex; }
    }
</style>