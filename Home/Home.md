---
banner: "https://api.dujin.org/bing/1366.php"
banner_y: 0.5
banner_x: 0.5
cssclasses: [home-dashboard]
---

# 🎮 游戏攻略管理主页

> 欢迎回到您的游戏世界！记录每一个成就，珍藏每一份感动。

---

## 🕹️ 快速操作栏

```dataviewjs
const appRef = app;
const adapter = appRef.vault.adapter;

// 辅助函数：确保文件/文件夹存在，若不存在则创建
async function ensureFile(path, fallback) {
    const dir = path.split('/').slice(0, -1).join('/');
    if (!(await adapter.exists(dir))) {
        try { await appRef.vault.createFolder(dir); } catch (e) {}
    }
    if (!(await adapter.exists(path))) {
        try {
            await appRef.vault.create(path, fallback);
            new Notice(`已创建文件: ${path}`);
        } catch (e) { new Notice('创建失败: ' + e.message, 4000); }
    }
    try { await appRef.workspace.openLinkText(path, '', true); } catch (e) {}
}

function makeBtn(label, path, fallback) {
    const b = dv.el('button', label, { cls: 'btn-nav' });
    b.addEventListener('click', () => ensureFile(path, fallback));
    return b;
}

// 定义路径
const now = new Date();
const yyyy = now.getFullYear();
const mm = (now.getMonth() + 1).toString().padStart(2, '0');
const dd = now.getDate().toString().padStart(2, '0');
const dailyPath = `Diary/${yyyy}/Daily/${yyyy}-${mm}-${dd}.md`;
const projectPath = `01Project/01Project.md`;
const notePath = `02Note/02Note.md`;
const resourcePath = `03Resource/03Resource.md`;

// 创建按钮容器
const wrap = dv.el('div', '', { cls: 'dv-nav-buttons' });

// 添加按钮
wrap.appendChild(makeBtn('📅 今日游玩日志', dailyPath, `# ${yyyy}-${mm}-${dd} 游玩日志\n\n## 🎮 今日进度\n- \n\n## 🏆 成就解锁\n- \n`));
wrap.appendChild(makeBtn('⚔️ 活跃项目', projectPath, `# 正在进行的项目\n`));
wrap.appendChild(makeBtn('📝 攻略笔记', notePath, `# 攻略笔记主页\n`));
wrap.appendChild(makeBtn('🧰 资源工具', resourcePath, `# 资源与工具\n`));

// 样式
const style = document.createElement('style');
style.innerHTML = `
.dv-nav-buttons { display: flex; flex-wrap: wrap; gap: 8px; margin: 8px 0; }
.dv-nav-buttons .btn-nav { padding: 8px 16px; border: 1px solid var(--background-modifier-border); border-radius: 6px; background: var(--background-secondary); cursor: pointer; font-weight: bold; }
.dv-nav-buttons .btn-nav:hover { background: var(--background-primary); box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
`;
wrap.appendChild(style);
```

---

## 📊 使用时长与统计

```dataviewjs
let ftMd = dv.pages("").file.sort(t => t.cday)[0];
let total = 0;
if (ftMd) {
    total = parseInt((new Date() - ftMd.ctime) / (60*60*24*1000));
}
let totalDays = "您已在这里记录了 **" + (total || 0) + "** 天的游戏旅程，";
let allFile = dv.pages('!"03Resource/templates"').file; // 排除模板
let totalMd = "共创建 **" + allFile.length + "** 篇文档，";
let totalTag = "**" + (allFile.etags.distinct().length || 0) + "** 个标签。";

dv.paragraph(totalDays + totalMd + totalTag);
```

---

## 🔥 活跃项目概览

![[01Project/01Project.md#🔥 活跃项目]]

---

## 🕒 最近更新

```dataviewjs
const pages = dv.pages('-"03Resource/templates"') // 排除模板
    .where(p => !p.file.path.includes("Home")) // 排除主页
    .sort(p => p.file.mtime, 'desc')
    .limit(10);

dv.table(
    ['📄 文件', '📂 目录', '🕒 更新时间'], 
    pages.map(p => [
        dv.fileLink(p.file.path, false, p.file.name), 
        p.file.folder,
        p.file.mtime.toFormat('yyyy-MM-dd HH:mm')
    ])
);
```
