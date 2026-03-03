<!DOCTYPE html>
<html lang="zh-TW">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>彈性輪班生活排程器</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://unpkg.com/lucide@latest"></script>
    <style>
        [x-cloak] { display: none !important; }
        .custom-scrollbar::-webkit-scrollbar { width: 4px; }
        .custom-scrollbar::-webkit-scrollbar-track { background: transparent; }
        .custom-scrollbar::-webkit-scrollbar-thumb { background: #cbd5e1; border-radius: 10px; }
    </style>
    <script>
        function plannerApp() {
            return {
                daysOfWeek: ['週一', '週二', '週三', '週四', '週五', '週六', '週日'],
                showPresetModal: false,
                showAddModal: false,
                activeDayIdx: 0,
                toast: { show: false, message: '' },
                weekData: Array.from({ length: 7 }, () => ({ shiftName: '', items: [] })),
                
                // 新增項目的暫存資料
                newItem: { time: '08:00', task: '' },

                presets: {
                    morning: {
                        name: '☀️ 早班',
                        time: '06:30-14:30',
                        color: 'bg-orange-500',
                        items: [
                            { time: '05:30', task: '起床與早餐' },
                            { time: '06:30', task: '🏢 開始上班' },
                            { time: '14:30', task: '🏠 下班/午睡 20m' },
                            { time: '15:30', task: '🏋️ 運動 (重訓/有氧)' },
                            { time: '18:30', task: '📖 深度學習 (語言/技術)' },
                            { time: '21:30', task: '💤 準備就寢' }
                        ]
                    },
                    middle: {
                        name: '🕛 中班',
                        time: '11:00-19:00',
                        color: 'bg-emerald-500',
                        items: [
                            { time: '08:00', task: '🏋️ 晨間運動' },
                            { time: '09:00', task: '📖 輕度學習 (聽力/閱讀)' },
                            { time: '11:00', task: '🏢 開始上班' },
                            { time: '19:00', task: '🏠 下班與晚餐' },
                            { time: '20:30', task: '💻 專題實作' },
                            { time: '23:30', task: '💤 睡覺' }
                        ]
                    },
                    afternoon: {
                        name: '🌤️ 午班',
                        time: '14:20-22:20',
                        color: 'bg-blue-500',
                        items: [
                            { time: '07:30', task: '📖 深度學習 (黃金時段)' },
                            { time: '09:30', task: '🏋️ 運動時間' },
                            { time: '14:20', task: '🏢 開始上班' },
                            { time: '22:20', task: '🏠 下班放鬆' },
                            { time: '23:30', task: '💤 睡覺' }
                        ]
                    },
                    night: {
                        name: '🌙 夜班',
                        time: '22:10-06:40',
                        color: 'bg-indigo-500',
                        items: [
                            { time: '08:00', task: '💤 進入睡眠 (避光)' },
                            { time: '15:30', task: '起床與第一餐' },
                            { time: '16:30', task: '🏋️ 運動時間 (喚醒身體)' },
                            { time: '17:30', task: '📖 學習時間' },
                            { time: '22:10', task: '🏢 開始上班' }
                        ]
                    },
                    off: {
                        name: '🌴 休假',
                        time: '全天',
                        color: 'bg-rose-500',
                        items: [
                            { time: '09:00', task: '自然醒 + 豐盛早餐' },
                            { time: '10:00', task: '🏋️ 戶外運動' },
                            { time: '13:00', task: '📖 專長衝刺 (4小時)' },
                            { time: '19:00', task: '🎬 娛樂放鬆時間' }
                        ]
                    }
                },

                init() {
                    const hash = window.location.hash;
                    if (hash && hash.length > 1) {
                        try {
                            const decoded = decodeURIComponent(escape(atob(hash.substring(1))));
                            this.weekData = JSON.parse(decoded);
                            this.showToast('成功匯入分享的班表！');
                            window.location.hash = '';
                        } catch (e) {
                            this.loadLocalData();
                        }
                    } else {
                        this.loadLocalData();
                    }

                    this.$nextTick(() => { lucide.createIcons(); });
                    
                    this.$watch('weekData', (value) => {
                        localStorage.setItem('shift_planner_data_v4', JSON.stringify(value));
                        this.$nextTick(() => { lucide.createIcons(); });
                    });
                },

                loadLocalData() {
                    const saved = localStorage.getItem('shift_planner_data_v4');
                    if (saved) {
                        try { this.weekData = JSON.parse(saved); } catch (e) {}
                    }
                },

                openPresetModal(idx) {
                    this.activeDayIdx = idx;
                    this.showPresetModal = true;
                },

                applyPreset(dayIdx, presetId) {
                    const preset = this.presets[presetId];
                    this.weekData[dayIdx] = {
                        shiftName: preset.name,
                        items: JSON.parse(JSON.stringify(preset.items))
                    };
                    this.showPresetModal = false;
                    this.showToast(`已套用 ${preset.name}`);
                },

                openAddModal(dayIdx) {
                    this.activeDayIdx = dayIdx;
                    this.newItem = { time: '08:00', task: '' };
                    this.showAddModal = true;
                },

                saveNewItem() {
                    if (!this.newItem.task.trim()) return;
                    
                    this.weekData[this.activeDayIdx].items.push({ ...this.newItem });
                    // 自動排序時間
                    this.weekData[this.activeDayIdx].items.sort((a, b) => a.time.localeCompare(b.time));
                    
                    this.showAddModal = false;
                    this.showToast('行程已新增');
                },

                removeItem(dayIdx, itemIdx) {
                    this.weekData[dayIdx].items.splice(itemIdx, 1);
                },

                copyShareLink() {
                    try {
                        const dataString = JSON.stringify(this.weekData);
                        const encoded = btoa(unescape(encodeURIComponent(dataString)));
                        const url = window.location.origin + window.location.pathname + '#' + encoded;
                        
                        const dummy = document.createElement('textarea');
                        document.body.appendChild(dummy);
                        dummy.value = url;
                        dummy.select();
                        document.execCommand('copy');
                        document.body.removeChild(dummy);
                        
                        this.showToast('分享連結已複製到剪貼簿！');
                    } catch (e) {
                        this.showToast('產生連結失敗');
                    }
                },

                showToast(msg) {
                    this.toast.message = msg;
                    this.toast.show = true;
                    setTimeout(() => { this.toast.show = false; }, 3000);
                }
            };
        }
    </script>
    <script src="https://unpkg.com/alpinejs@3.x.x/dist/cdn.min.js"></script>
</head>
<body class="bg-slate-50 min-h-screen text-slate-900 font-sans pb-20" x-data="plannerApp()">

    <!-- Header -->
    <header class="bg-white border-b border-slate-200 sticky top-0 z-30 shadow-sm">
        <div class="max-w-6xl mx-auto px-4 py-4 flex justify-between items-center">
            <div>
                <h1 class="text-xl font-bold text-blue-600 flex items-center gap-2">
                    <i data-lucide="calendar-days"></i> 輪班生活排程器
                </h1>
                <p class="text-xs text-slate-500">規劃屬於你的彈性作息</p>
            </div>
            <button @click="copyShareLink()" class="bg-blue-600 hover:bg-blue-700 text-white px-4 py-2 rounded-lg text-sm font-medium flex items-center gap-2 transition-all shadow-md active:scale-95">
                <i data-lucide="share-2" class="w-4 h-4"></i>
                <span class="hidden sm:inline">分享目前進度</span>
            </button>
        </div>
    </header>

    <main class="max-w-6xl mx-auto px-4 py-6">
        <!-- 班別色塊提示 -->
        <div class="mb-8 flex flex-wrap gap-3">
            <template x-for="(p, id) in presets" :key="id">
                <div class="bg-white px-3 py-1.5 rounded-full border border-slate-200 shadow-sm flex items-center gap-2">
                    <span :class="p.color + ' w-2.5 h-2.5 rounded-full'"></span>
                    <span class="text-xs font-medium text-slate-600" x-text="p.name"></span>
                </div>
            </template>
        </div>

        <!-- 週間行程 -->
        <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-7 gap-4">
            <template x-for="(day, dayIdx) in weekData" :key="dayIdx">
                <div class="bg-white rounded-2xl shadow-sm border border-slate-200 flex flex-col overflow-hidden h-fit transition-all hover:shadow-md">
                    <!-- Day Header -->
                    <div class="p-4 border-b border-slate-100 flex justify-between items-center bg-slate-50">
                        <div>
                            <span class="text-xs font-bold text-slate-400 uppercase tracking-wider" x-text="daysOfWeek[dayIdx]"></span>
                            <div class="font-bold text-slate-800" x-text="day.shiftName || '未排班'"></div>
                        </div>
                        <button @click="openPresetModal(dayIdx)" class="p-1.5 hover:bg-blue-100 text-blue-600 rounded-md transition-colors">
                            <i data-lucide="settings-2" class="w-4 h-4"></i>
                        </button>
                    </div>

                    <!-- Day Content -->
                    <div class="p-4 space-y-4 min-h-[140px]">
                        <div class="relative border-l-2 border-slate-100 ml-2 pl-4 space-y-4">
                            <template x-for="(item, itemIdx) in day.items" :key="itemIdx">
                                <div class="relative group">
                                    <div class="absolute -left-[21px] top-1.5 w-2.5 h-2.5 rounded-full bg-slate-300 border-2 border-white"></div>
                                    <div class="flex flex-col">
                                        <span class="text-[10px] font-mono text-slate-400" x-text="item.time"></span>
                                        <span class="text-sm font-medium text-slate-700" x-text="item.task"></span>
                                    </div>
                                    <button @click="removeItem(dayIdx, itemIdx)" class="absolute right-0 top-0 opacity-0 group-hover:opacity-100 text-red-300 hover:text-red-500 transition-opacity">
                                        <i data-lucide="trash-2" class="w-3.5 h-3.5"></i>
                                    </button>
                                </div>
                            </template>
                        </div>
                        
                        <button @click="openAddModal(dayIdx)" class="w-full py-2 border-2 border-dashed border-slate-100 rounded-lg text-slate-400 hover:text-blue-500 hover:border-blue-200 hover:bg-blue-50 text-xs flex items-center justify-center gap-1 transition-all">
                            <i data-lucide="plus" class="w-3 h-3"></i> 新增行程
                        </button>
                    </div>
                </div>
            </template>
        </div>
    </main>

    <!-- Modal 1: 班別範本設定 -->
    <div x-show="showPresetModal" x-cloak class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm" x-transition.opacity>
        <div @click.away="showPresetModal = false" class="bg-white rounded-3xl shadow-2xl w-full max-w-md overflow-hidden" x-transition.scale>
            <div class="p-6 border-b border-slate-100 flex justify-between items-center">
                <h2 class="text-lg font-bold flex items-center gap-2">
                    <i data-lucide="layout" class="text-blue-600"></i>
                    設定 <span x-text="daysOfWeek[activeDayIdx]"></span> 的班別
                </h2>
                <button @click="showPresetModal = false" class="text-slate-400 hover:text-slate-600">
                    <i data-lucide="x"></i>
                </button>
            </div>
            
            <div class="p-6 space-y-3">
                <template x-for="(p, id) in presets" :key="id">
                    <button @click="applyPreset(activeDayIdx, id)" class="w-full flex items-center justify-between p-4 rounded-xl border-2 border-slate-100 hover:border-blue-500 hover:bg-blue-50 transition-all text-left group">
                        <div class="flex items-center gap-3">
                            <span :class="p.color + ' w-4 h-4 rounded-full'"></span>
                            <div>
                                <div class="font-bold text-slate-800" x-text="p.name"></div>
                                <div class="text-xs text-slate-500" x-text="p.time"></div>
                            </div>
                        </div>
                        <i data-lucide="chevron-right" class="text-slate-300 group-hover:text-blue-500 transition-colors"></i>
                    </button>
                </template>
            </div>
        </div>
    </div>

    <!-- Modal 2: 手動新增行程 -->
    <div x-show="showAddModal" x-cloak class="fixed inset-0 z-50 flex items-center justify-center p-4 bg-slate-900/60 backdrop-blur-sm" x-transition.opacity>
        <div @click.away="showAddModal = false" class="bg-white rounded-3xl shadow-2xl w-full max-w-sm overflow-hidden" x-transition.scale>
            <div class="p-6 border-b border-slate-100">
                <h2 class="text-lg font-bold text-slate-800">新增行程項目</h2>
            </div>
            
            <div class="p-6 space-y-4">
                <div>
                    <label class="block text-xs font-bold text-slate-400 uppercase mb-1">時間</label>
                    <input type="time" x-model="newItem.time" class="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl outline-none focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 transition-all">
                </div>
                <div>
                    <label class="block text-xs font-bold text-slate-400 uppercase mb-1">活動內容</label>
                    <input type="text" x-model="newItem.task" placeholder="例如：運動、閱讀..." class="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl outline-none focus:ring-2 focus:ring-blue-500/20 focus:border-blue-500 transition-all" @keyup.enter="saveNewItem()">
                </div>
            </div>
            
            <div class="p-6 bg-slate-50 flex gap-3">
                <button @click="showAddModal = false" class="flex-1 px-4 py-2.5 text-slate-600 font-medium hover:bg-slate-200 rounded-xl transition-colors">取消</button>
                <button @click="saveNewItem()" class="flex-1 px-4 py-2.5 bg-blue-600 text-white font-bold rounded-xl shadow-md shadow-blue-200 hover:bg-blue-700 active:scale-95 transition-all">儲存項目</button>
            </div>
        </div>
    </div>

    <!-- Toast Notification -->
    <div x-show="toast.show" x-cloak 
         x-transition:enter="transition ease-out duration-300"
         x-transition:enter-start="opacity-0 translate-y-10"
         x-transition:enter-end="opacity-100 translate-y-0"
         x-transition:leave="transition ease-in duration-200"
         x-transition:leave-start="opacity-100 translate-y-0"
         x-transition:leave-end="opacity-0 translate-y-10"
         class="fixed bottom-8 left-1/2 -translate-x-1/2 z-50 px-6 py-3 bg-slate-800 text-white rounded-full shadow-lg flex items-center gap-2">
        <i data-lucide="check-circle" class="w-4 h-4 text-green-400"></i>
        <span x-text="toast.message" class="text-sm"></span>
    </div>

</body>
</html>
