# 🎮 游戏项目看板

> [!INFO] 系统使用说明
> 本系统通过关联 **项目档案** 与 **每日日记** 来自动追踪进度。
> 
> **3步操作流程：**
> 1. **建档**：点击下方按钮创建游戏档案（必须存在文件才能被追踪）。
> 2. **记录**：在日记中输入链接+标签，例如：`- [[游戏名]] 打过了第一章 #进行中`。
> 3. **查看**：本页面会自动根据标签归类游戏状态。
> 
> 📘 **详细教程**：[[03Resource/Game_Guide_System_Manual|📖 游戏攻略系统：全流程操作指南]]

```dataviewjs
// 快速新建按钮
const { createButton } = app.plugins.plugins["buttons"] || {};
const wrap = dv.el("div", "", { cls: "project-actions" });

const btn = document.createElement("button");
btn.textContent = "➕ 新建游戏项目";
btn.className = "mod-cta";
btn.onclick = async () => {
    const name = await app.plugins.plugins["templater-obsidian"].templater.functions_generator.internal_functions.modules.system.prompt("请输入游戏名称");
    if (name) {
        const templatePath = "03Resource/templates/Game Project Template.md";
        const folder = "01Project";
        const filename = `${folder}/${name}.md`;
        const exists = await app.vault.adapter.exists(filename);
        if (!exists) {
           const templateFile = app.vault.getAbstractFileByPath(templatePath);
           if(templateFile) {
               // 创建文件并应用模板 (简化版，直接调用Templater API可能较复杂，这里使用Vault API创建)
               // 为了触发Templater，最好使用该插件的API，或者让用户手动应用。
               // 这里尝试创建文件后打开。
               await app.vault.create(filename, "");
               const newFile = app.vault.getAbstractFileByPath(filename);
               await app.workspace.getLeaf(true).openFile(newFile);
               // 尝试触发模板插入
               app.commands.executeCommandById("templater-obsidian:insert-templater-source-to-active-file");
           } else {
               new Notice("未找到模板文件！");
           }
        } else {
            new Notice("项目已存在！");
        }
    }
};
wrap.appendChild(btn);
```

```dataviewjs
// 定义状态标签映射
const STATUS_MAP = {
    "新开": "active",
    "进行中": "active",
    "待玩": "backlog",
    "通关": "completed",
    "全成就": "completed",
    "弃坑": "dropped"
};

// 获取所有游戏项目
const projects = dv.pages('"01Project"').where(p => p.file.name !== "01Project");

// 扫描所有日记，找到针对这些项目的最新标签
// 注意：这可能在大型库中较慢。如果卡顿，可以限制扫描最近30天的日记。
// 优化策略：Project本身不需要存Status，我们实时计算。

// 1. 构建项目映射
let projectStatus = {}; // { path: { status: 'backlog', date: ... } }

// 初始化所有项目
for (let p of projects) {
    if (!p || !p.file) continue; // 安全检查
    projectStatus[p.file.path] = { 
        status: 'backlog', // 默认为待玩
        lastDate: p.file.ctime,
        file: p // 这里存的是 dv.page() 对象
    };
}

// 辅助函数：标准化路径比较
function isSameFile(path1, path2) {
    if (!path1 || !path2) return false;
    // 移除扩展名和文件夹，只比较文件名
    const name1 = path1.split('/').pop().replace(/\.md$/, '');
    const name2 = path2.split('/').pop().replace(/\.md$/, '');
    return name1 === name2;
}

// 2. 扫描最近 90 天的日记 (性能优化)
// 使用 dv.date("now") 获取当前时间，并通过 minus 推算起始日期
const logs = dv.pages('"Diary"').where(p => p.file.day >= dv.date("now").minus({ days: 90 })).sort(p => p.file.day, 'asc');

for (let log of logs) {
    // 获取当天的所有列表项
    const listItems = log.file.lists;
    for (let item of listItems) {
        // 检查是否有链接到 Known Projects
        for (let link of item.outlinks) {
            
            // 尝试查找匹配的项目
            let matchedKey = Object.keys(projectStatus).find(k => isSameFile(k, link.path));
            let matchedProject = matchedKey ? projectStatus[matchedKey] : null;

            if (matchedProject) { // 是一个游戏项目
                // 检查标签
                if (!item.tags) continue; // 防御性检查
                for (let tag of item.tags) {
                    const tagClean = tag.replace('#', '');
                    if (STATUS_MAP[tagClean]) {
                        // 确保 projectStatus[matchedKey] 存在且 valid
                        if (projectStatus[matchedKey]) {
                             projectStatus[matchedKey] = {
                                status: STATUS_MAP[tagClean],
                                lastDate: log.file.day,
                                file: projectStatus[matchedKey].file, // 保持引用
                                latestLog: item.text,
                                logLink: log.file.link
                            };
                        }
                    }
                }
            }
        }
    }
}

// 3. 分组展示
const groups = {
    active: [],
    backlog: [],
    completed: [],
    dropped: []
};

for (let path in projectStatus) {
    const p = projectStatus[path];
    if (groups[p.status]) {
        groups[p.status].push(p);
    }
}

// 渲染函数
function renderTable(title, items) {
    if (items.length === 0) return;
    dv.header(2, title + ` (${items.length})`);
    
    // 按最后更新时间倒序
    items.sort((a, b) => b.lastDate - a.lastDate);

    dv.table(
        ["🎮 游戏", "📅 最新日志", "🗒️ 摘要"],
        items.map(i => {
           // 安全地获取链接：i.file 可能是一个 Page 对象，或者只是个简单的包装
           let link = i.file && i.file.file && i.file.file.link ? i.file.file.link : dv.fileLink(i.file.file.path);
           
           return [
            link,
            i.logLink || "-",
            i.latestLog ? i.latestLog.replace(/\[\[.*?\]\]/g, "").replace(/#\S+/g, "").trim() : "暂无最近日志"
           ];
        })
    );
}

renderTable("🔥 正在攻略 (Active)", groups.active);
renderTable("📦 待玩清单 (Backlog)", groups.backlog);
renderTable("🏆 已完成 / 全成就 (Completed)", groups.completed);

```
