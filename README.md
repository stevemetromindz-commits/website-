<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Pro Website Monitor</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @keyframes pulse-green { 0% { box-shadow: 0 0 0 0 rgba(16, 185, 129, 0.4); } 70% { box-shadow: 0 0 0 10px rgba(16, 185, 129, 0); } 100% { box-shadow: 0 0 0 0 rgba(16, 185, 129, 0); } }
        .online-pulse { animation: pulse-green 2s infinite; }
        .custom-scroll::-webkit-scrollbar { width: 4px; }
        .custom-scroll::-webkit-scrollbar-thumb { background: #4b5563; border-radius: 10px; }
    </style>
</head>
<body class="bg-gray-900 text-gray-100 min-h-screen p-4 md:p-8">

    <div class="max-w-6xl mx-auto">
        <div class="flex flex-col md:flex-row justify-between items-center mb-10 gap-4">
            <div>
                <h1 class="text-3xl font-black text-emerald-500 tracking-tight">SITEWATCH <span class="text-gray-500 font-light text-xl italic">v2.0</span></h1>
                <p class="text-gray-400 text-sm">Real-time status & latency tracking</p>
            </div>
            <div class="flex gap-2">
                <button onclick="checkAllNow()" class="bg-gray-700 hover:bg-gray-600 px-4 py-2 rounded-lg text-sm font-semibold transition">🔄 Refresh All</button>
                <button onclick="clearAll()" class="bg-rose-900/30 text-rose-400 hover:bg-rose-900/50 px-4 py-2 rounded-lg text-sm font-semibold transition">🗑️ Clear All</button>
            </div>
        </div>

        <div class="bg-gray-800 p-4 rounded-2xl border border-gray-700 shadow-2xl mb-8">
            <div class="flex flex-col md:flex-row gap-3">
                <input type="text" id="urlInput" placeholder="https://example.com" class="bg-gray-900 border border-gray-700 p-3 rounded-xl flex-1 outline-none focus:ring-2 focus:ring-emerald-500 transition text-emerald-400">
                <button onclick="addWebsite()" class="bg-emerald-600 hover:bg-emerald-500 px-8 py-3 rounded-xl font-bold shadow-lg shadow-emerald-900/20 transition-all active:scale-95">Add Monitor</button>
            </div>
        </div>

        <div class="mb-6">
            <input type="text" id="searchInput" onkeyup="searchWebsites()" placeholder="🔍 Filter websites..." class="w-full md:w-64 bg-transparent border-b border-gray-700 p-2 outline-none focus:border-emerald-500 transition">
        </div>

        <div id="websitesGrid" class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
            </div>
    </div>

    <audio id="alertSound"><source src="https://actions.google.com/sounds/v1/alarms/beep_short.ogg" type="audio/ogg"></audio>

    <script>
        let monitors = JSON.parse(localStorage.getItem('monitors')) || [];
        const grid = document.getElementById('websitesGrid');

        // Initialize Notification
        if ('Notification' in window && Notification.permission !== 'granted') Notification.requestPermission();

        function save() {
            localStorage.setItem('monitors', JSON.stringify(monitors));
            render();
        }

        function addWebsite() {
            const input = document.getElementById('urlInput');
            let url = input.value.trim();
            if (!/^https?:\/\//.test(url)) return alert("Include http:// or https://");
            
            monitors.push({
                url: url,
                active: true,
                status: 'pending',
                latency: '--',
                lastCheck: '--'
            });
            input.value = '';
            save();
            checkStatus(monitors.length - 1);
        }

        async function checkStatus(index) {
            const site = monitors[index];
            if (!site || !site.active) return;

            const start = performance.now();
            try {
                // Using a cache-busting parameter to ensure fresh results
                await fetch(site.url, { mode: 'no-cors', cache: 'no-store' });
                const end = performance.now();
                
                site.status = 'online';
                site.latency = Math.round(end - start) + 'ms';
            } catch (e) {
                if (site.status === 'online') { // Alert only on transition to down
                    new Notification(`❌ Down: ${site.url}`);
                    document.getElementById('alertSound').play();
                }
                site.status = 'offline';
                site.latency = 'ERROR';
            }
            site.lastCheck = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit', second: '2-digit' });
            render();
        }

        function toggleActive(index) {
            monitors[index].active = !monitors[index].active;
            if (monitors[index].active) checkStatus(index);
            else monitors[index].status = 'pending';
            save();
        }

        function removeSite(index) {
            monitors.splice(index, 1);
            save();
        }

        function clearAll() {
            if(confirm("Delete all monitors?")) { monitors = []; save(); }
        }

        function searchWebsites() {
            const term = document.getElementById('searchInput').value.toLowerCase();
            render(term);
        }

        function checkAllNow() {
            monitors.forEach((_, i) => checkStatus(i));
        }

        function render(filter = '') {
            grid.innerHTML = '';
            monitors.forEach((site, i) => {
                if (filter && !site.url.toLowerCase().includes(filter)) return;

                const card = document.createElement('div');
                const isOnline = site.status === 'online';
                const isOffline = site.status === 'offline';
                
                card.className = `relative bg-gray-800 p-5 rounded-2xl border-2 transition-all duration-500 ${!site.active ? 'border-gray-700 opacity-60' : isOnline ? 'border-emerald-500/30' : isOffline ? 'border-rose-500/50 shadow-lg shadow-rose-900/20' : 'border-gray-700'}`;
                
                card.innerHTML = `
                    <button onclick="removeSite(${i})" class="absolute top-3 right-3 text-gray-500 hover:text-white transition">✖</button>
                    <div class="flex items-center gap-3 mb-4">
                        <div class="w-3 h-3 rounded-full ${!site.active ? 'bg-gray-600' : isOnline ? 'bg-emerald-500 online-pulse' : 'bg-rose-500'}"></div>
                        <span class="text-[10px] font-bold uppercase tracking-widest text-gray-500">${site.status}</span>
                    </div>
                    <h3 class="font-bold truncate pr-6 mb-1 text-sm md:text-base" title="${site.url}">${site.url.replace('https://','').replace('http://','')}</h3>
                    <div class="flex justify-between items-end mt-6">
                        <div>
                            <p class="text-[10px] text-gray-500 uppercase font-bold">Latency</p>
                            <p class="font-mono text-lg ${isOnline ? 'text-emerald-400' : 'text-gray-400'}">${site.latency}</p>
                        </div>
                        <div class="text-right">
                            <p class="text-[10px] text-gray-500 uppercase font-bold italic">Last Check</p>
                            <p class="text-xs text-gray-400">${site.lastCheck}</p>
                        </div>
                    </div>
                    <button onclick="toggleActive(${i})" class="mt-4 w-full py-2 rounded-lg text-[10px] font-bold uppercase tracking-tighter transition ${site.active ? 'bg-gray-700 hover:bg-rose-900/40 text-gray-300' : 'bg-emerald-900/40 text-emerald-400 hover:bg-emerald-900/60'}">
                        ${site.active ? 'Pause Monitor' : 'Resume Monitor'}
                    </button>
                `;
                grid.appendChild(card);
            });
        }

        // Background Check Loop (Every 5 minutes)
        setInterval(checkAllNow, 300000);
        
        // Initial Render
        render();
        checkAllNow();
    </script>
</body>
</html>
