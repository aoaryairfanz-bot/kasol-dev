<script>
    function tilt(node) {
        const handleMouseMove = (e) => {
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
</script>

<section class="feature-split">
    <div class="container split-layout">
        <div class="img-container">
            <img src="https://images.unsplash.com/photo-1515886657613-9f3515b0c78f?auto=format&fit=crop&q=80&w=600" alt="Model" class="rounded-img" />
        </div>
        <div class="text-container">
            <h2>Remove Background from Images Instantly with Clean, High-Resolution Results</h2>
            <p>Remove the background from any image in seconds and download a clean, high-resolution transparent PNG. Get sharp edges with no rough outlines, ready for web and social media. No signup required.</p>
            <button class="btn-3d blue">Try it now</button>
        </div>
    </div>
</section>

<section class="value-props">
    <div class="container">
        <h2>Background Removal for Photos, Products, and Design Projects</h2>
        <p class="sub-heading">Turn images into high-resolution transparent PNGs in seconds.</p>
        <div class="card-grid">
            <div class="prop-card" use:tilt>
                <div class="icon-box">🖼️</div><h3>Photos for products</h3><p>Clean up product images for websites, marketplaces, packaging, and catalogs.</p>
            </div>
            <div class="prop-card" use:tilt>
                <div class="icon-box">✨</div><h3>Logos and graphics</h3><p>Turn logos, illustrations, and design files into transparent PNGs.</p>
            </div>
            <div class="prop-card" use:tilt>
                <div class="icon-box">🖨️</div><h3>Stickers and print</h3><p>Create clean cutouts for stickers, labels, and print-ready artwork.</p>
            </div>
        </div>
    </div>
</section>

<style>
    .container { max-width: 1200px; margin: 0 auto; padding: 80px 20px; }
    .split-layout { display: flex; align-items: center; gap: 60px; }
    .text-container { flex: 1; }
    .text-container h2 { font-size: 2.5rem; font-weight: 800; margin-bottom: 20px; line-height: 1.2; letter-spacing: -1px; color: #111827;}
    .text-container p { font-size: 1.1rem; color: #6b7280; line-height: 1.6; margin-bottom: 30px; }
    .img-container { flex: 1; }
    .rounded-img { width: 100%; border-radius: 40px; box-shadow: 0 20px 40px rgba(0,0,0,0.1); transition: transform 0.3s;}
    .rounded-img:hover { transform: translateY(-10px); }

    .value-props { background: linear-gradient(135deg, #8b5cf6 0%, #3b82f6 100%); color: white; text-align: center; }
    .value-props h2 { font-size: 2.5rem; font-weight: 800; margin-bottom: 10px; }
    .sub-heading { font-size: 1.2rem; font-weight: 500; margin-bottom: 50px; opacity: 0.9;}
    .card-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 30px; perspective: 1000px;}
    .prop-card { background: white; color: #111827; padding: 40px 30px; border-radius: 24px; text-align: left; box-shadow: 0 10px 30px rgba(0,0,0,0.1); transition: transform 0.1s ease-out; transform-style: preserve-3d; }
    .icon-box { border: 2px solid #111827; display: inline-block; padding: 10px; border-radius: 12px; font-size: 1.5rem; margin-bottom: 20px; box-shadow: 2px 2px 0px #111827; transform: translateZ(20px);}
    .prop-card h3 { font-size: 1.3rem; font-weight: 800; margin-bottom: 15px; transform: translateZ(10px);}
    .prop-card p { color: #6b7280; line-height: 1.6; font-size: 0.95rem; transform: translateZ(5px);}

    .btn-3d { padding: 12px 24px; border: 2px solid #111827; border-radius: 12px; font-weight: bold; background: white; cursor: pointer; box-shadow: 4px 4px 0px #111827; transition: transform 0.1s, box-shadow 0.1s; }
    .btn-3d:active { transform: translate(4px, 4px); box-shadow: 0px 0px 0px #111827; }
    .btn-3d.blue { background: #3b82f6; color: white; }

    @media (max-width: 768px) {
        .split-layout { flex-direction: column; text-align: center; }
    }
</style>