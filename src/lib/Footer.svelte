<footer class="site-footer">
    <div class="container footer-content">
        <div class="footer-brand">
            <h2>K<span>Dev</span></h2>
            <p>Smile more, worry less🤗.</p>
        </div>
        <div class="footer-links">
            <div class="link-col">
                <h4>Tools</h4>
                <p>Remove Background</p>
                <p>Stock Opname</p>
                <p>Video Downloader</p>
            </div>
            <div class="link-col">
                <h4>Company</h4>
                <p>About Us</p>
                <p>Terms of Service</p>
                <p>Privacy Policy</p>
            </div>
        </div>
    </div>
    <div class="footer-bottom">
        <p>Made with ♡ by  KasolDev &copy; 2026. All rights reserved.</p>
    </div>
</footer>

<style>
    .container { max-width: 1200px; margin: 0 auto; padding: 0 20px; }
    .site-footer { background: #111827; color: white; padding-top: 60px; }
    .footer-content { padding-bottom: 60px; display: flex; justify-content: space-between; flex-wrap: wrap; gap: 40px;}
    .footer-brand h2 { font-size: 2rem; margin: 0 0 10px 0; color: #f472b6;}
    .footer-brand span { color: white; }
    .footer-brand p { color: #9ca3af; }
    .footer-links { display: flex; gap: 80px; }
    .link-col h4 { font-size: 1.1rem; margin-bottom: 20px; }
    .link-col p { color: #9ca3af; margin-bottom: 10px; cursor: pointer; transition: color 0.2s; }
    .link-col p:hover { color: white; }
    .footer-bottom { border-top: 1px solid #374151; padding: 20px; text-align: center; color: #9ca3af; font-size: 0.9rem;}

    @media (max-width: 768px) {
        .footer-content { flex-direction: column; text-align: center; }
        .footer-links { justify-content: center; flex-wrap: wrap; gap: 40px;}
    }
</style>