<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>周报看板 - Weekly Report Board</title>
  <style>
    * {
      box-sizing: border-box;
      margin: 0;
      padding: 0;
      font-family: system-ui, -apple-system, "Segoe UI", Roboto, "PingFang SC", "Microsoft YaHei", sans-serif;
    }
    body {
      background-color: #f0f2f5;
      padding: 16px;
      color: #1f2937;
    }
    .container {
      max-width: 1400px;
      margin: 0 auto;
    }
    .header-bar {
      display: flex;
      align-items: center;
      gap: 12px;
      margin-bottom: 16px;
      flex-wrap: wrap;
    }
    h1 {
      font-size: 20px;
      font-weight: 600;
    }
    button {
      padding: 6px 12px;
      border-radius: 6px;
      border: none;
      cursor: pointer;
      font-size: 14px;
      transition: opacity 0.15s, background 0.15s;
    }
    button:hover {
      opacity: 0.9;
    }
    button:active {
      opacity: 1;
    }
    .btn-primary {
      background: #2563eb;
      color: white;
    }
    .btn-secondary {
      background: #6b7280;
      color: white;
    }
    .tip-text {
      color: #6b7280;
      font-size: 13px;
      margin-bottom: 12px;
      line-height: 1.5;
    }
    .empty-tip {
      color: #9ca3af;
      font-size: 14px;
      text-align: center;
      padding: 60px 20px;
      grid-column: 1 / -1;
      background: #fff;
      border-radius: 10px;
      border: 1px dashed #e5e7eb;
    }
    .board-wrap {
      display: grid;
      grid-template-columns: repeat(auto-fill, minmax(320px, 1fr));
      gap: 14px;
      min-height: 200px;
    }
    .card {
      background: #fff;
      border-radius: 10px;
      padding: 14px;
      box-shadow: 0 2px 6px rgba(0,0,0,0.08);
      display: flex;
      flex-direction: column;
      transition: background 0.15s, box-shadow 0.15s;
    }
    .card:hover {
      box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    }
    .card.drag-over {
      background: #dbeafe;
      border: 2px dashed #2563eb;
      padding: 12px;
    }
    .card-header {
      display: flex;
      justify-content: space-between;
      align-items: center;
      gap: 8px;
      margin-bottom: 10px;
      cursor: grab;
      user-select: none;
    }
    .card-header:active {
      cursor: grabbing;
    }
    .card-title {
      font-weight: 600;
      font-size: 15px;
      flex: 1;
      min-width: 0;
      overflow: hidden;
      text-overflow: ellipsis;
      white-space: nowrap;
    }
    .card-actions {
      flex-shrink: 0;
      display: flex;
      gap: 4px;
    }
    .card-actions button {
      padding: 3px 8px;
      font-size: 12px;
      background: #f3f4f6;
      color: #374151;
      border-radius: 4px;
    }
    .card-actions button:hover {
      background: #e5e7eb;
    }
    .card-actions .del-card {
      background: #fee2e2;
      color: #dc2626;
    }
    .card-actions .del-card:hover {
      background: #fecaca;
    }
    /* 改用textarea，彻底解决换行、XSS、格式问题 */
    .card-content {
      width: 100%;
      min-height: 80px;
      max-height: 420px;
      font-size: 14px;
      line-height: 1.6;
      color: #374151;
      border: 1px solid #e5e7eb;
      border-radius: 6px;
      padding: 8px 10px;
      resize: vertical;
      outline: none;
      font-family: inherit;
      word-break: break-word;
      overflow-wrap: break-word;
      background: #fcfcfd;
      transition: border-color 0.15s, background 0.15s;
    }
    .card-content:focus {
      border-color: #2563eb;
      background: #fff;
      box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
    }
    #file-input {
      display: none;
    }

    /* 打印样式优化 */
    @media print {
      body {
        background: #fff;
        padding: 0;
      }
      .header-bar, .tip-text {
        display: none;
      }
      .card {
        box-shadow: none;
        border: 1px solid #ddd;
        page-break-inside: avoid;
      }
      .card-content {
        border: none;
        resize: none;
        max-height: none;
        background: #fff;
      }
    }
  </style>
</head>
<body>
<div class="container">
  <div class="header-bar">
    <h1>周报看板</h1>
    <button class="btn-primary" id="addCardBtn">新增周报卡片</button>
    <button class="btn-primary" id="exportBtn">导出备份(JSON)</button>
    <button class="btn-secondary" id="importBtn">导入备份</button>
    <input type="file" id="file-input" accept=".json">
  </div>
  <div class="tip-text">⚠️ 数据默认保存在浏览器本地存储；清除缓存、无痕模式会丢失，记得定期导出备份文件 | 拖拽卡片头部可调整排序</div>
  <div class="board-wrap" id="board"></div>
</div>

<script>
const STORAGE_KEY = "weekly_report_board_data";
const boardEl = document.getElementById('board');
const fileInput = document.getElementById('file-input');

// 全局唯一数据源，所有操作基于此数组
let currentList = [];
let saveTimer = null;
let isUpdatingFromStorage = false; // 多标签同步锁

// ========== 数据读写层 ==========
function loadData() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (raw) {
      const data = JSON.parse(raw);
      if (Array.isArray(data)) {
        return data.map(item => ({
          title: typeof item.title === 'string' ? item.title : '未命名周报',
          content: typeof item.content === 'string' ? item.content : ''
        }));
      }
    }
  } catch (e) {
    console.warn('本地数据读取失败，使用空数据', e);
  }
  return [];
}

function saveData(immediate = false) {
  const doSave = () => {
    try {
      localStorage.setItem(STORAGE_KEY, JSON.stringify(currentList));
    } catch (e) {
      alert('❌ 保存失败：浏览器本地存储不可用，请立即导出备份后再继续操作');
      console.error(e);
    }
  };

  clearTimeout(saveTimer);
  if (immediate) {
    doSave();
  } else {
    saveTimer = setTimeout(doSave, 300);
  }
}

// 强制立即落盘并清理定时器（结构操作前必须调用）
function flushSave() {
  clearTimeout(saveTimer);
  saveTimer = null;
  try {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(currentList));
  } catch (e) {
    alert('❌ 保存失败：浏览器本地存储不可用');
    console.error(e);
  }
}

// ========== 工具函数 ==========
function formatLocalDate() {
  const d = new Date();
  const year = d.getFullYear();
  const month = String(d.getMonth() + 1).padStart(2, '0');
  const day = String(d.getDate()).padStart(2, '0');
  return `${year}-${month}-${day}`;
}

// textarea 自动高度适配
function autoResize(textarea) {
  textarea.style.height = 'auto';
  textarea.style.height = Math.min(textarea.scrollHeight, 420) + 'px';
}

// ========== 渲染层 ==========
function render() {
  // 重渲染前清理旧定时器，防止旧数据覆盖
  clearTimeout(saveTimer);
  saveTimer = null;
  boardEl.innerHTML = '';

  // 空状态
  if (currentList.length === 0) {
    const empty = document.createElement('div');
    empty.className = 'empty-tip';
    empty.textContent = '暂无周报卡片，点击上方「新增周报卡片」开始使用';
    boardEl.appendChild(empty);
    return;
  }

  currentList.forEach((item, index) => {
    const card = document.createElement('div');
    card.className = 'card';
    card.dataset.index = index;

    card.innerHTML = `
      <div class="card-header" draggable="true">
        <div class="card-title" title="${item.title || '未命名周报'}"></div>
        <div class="card-actions">
          <button class="edit-title" type="button">改标题</button>
          <button class="del-card" type="button">删除</button>
        </div>
      </div>
      <textarea class="card-content" spellcheck="false"></textarea>
    `;

    const titleEl = card.querySelector('.card-title');
    const contentEl = card.querySelector('.card-content');
    const headerEl = card.querySelector('.card-header');
    const actionsEl = card.querySelector('.card-actions');

    // 文本赋值，安全无XSS
    titleEl.textContent = item.title || '未命名周报';
    contentEl.value = item.content || '';

    // 初始化高度
    requestAnimationFrame(() => autoResize(contentEl));

    // 内容编辑：仅更新数据，不重渲染
    contentEl.addEventListener('input', () => {
      currentList[index].content = contentEl.value;
      autoResize(contentEl);
      saveData(false);
    });

    // 阻止按钮区域触发拖拽
    actionsEl.addEventListener('mousedown', (e) => {
      e.stopPropagation();
    });

    // 修改标题
    card.querySelector('.edit-title').addEventListener('click', () => {
      const oldTitle = item.title || '未命名周报';
      const newTitle = prompt("输入卡片标题", oldTitle);
      if(newTitle !== null) {
        flushSave();
        currentList[index].title = newTitle.trim() || '未命名周报';
        saveData(true);
        render();
      }
    });

    // 删除卡片
    card.querySelector('.del-card').addEventListener('click', () => {
      const cardTitle = item.title || '未命名周报';
      if(confirm(`确定删除「${cardTitle}」这条卡片吗？此操作无法撤销。`)){
        flushSave();
        currentList.splice(index, 1);
        saveData(true);
        render();
      }
    });

    // ========== 拖拽排序逻辑 ==========
    headerEl.addEventListener('dragstart', (e) => {
      e.dataTransfer.setData('text/plain', index);
      e.dataTransfer.effectAllowed = 'move';
      // 拖拽预览使用整张卡片
      if (e.dataTransfer.setDragImage) {
        e.dataTransfer.setDragImage(card, 20, 20);
      }
      contentEl.blur();
    });

    card.addEventListener('dragover', (e) => {
      e.preventDefault();
      e.dataTransfer.dropEffect = 'move';
    });

    card.addEventListener('dragenter', (e) => {
      e.preventDefault();
      card.classList.add('drag-over');
    });

    card.addEventListener('dragleave', (e) => {
      // 仅真正离开卡片时移除高亮，解决子元素冒泡闪烁
      if (!card.contains(e.relatedTarget)) {
        card.classList.remove('drag-over');
      }
    });

    card.addEventListener('drop', (e) => {
      e.preventDefault();
      card.classList.remove('drag-over');
      
      const from = Number(e.dataTransfer.getData('text/plain'));
      const to = index;
      if(from === to || isNaN(from)) return;

      flushSave();

      // 修复索引偏移：向下拖拽时插入位置减1，保证始终落在目标卡片前方
      const [movedItem] = currentList.splice(from, 1);
      const insertIndex = from < to ? to - 1 : to;
      currentList.splice(insertIndex, 0, movedItem);

      saveData(true);
      render();
    });

    boardEl.appendChild(card);
  })
}

// 全局拖拽结束，清理所有残留高亮
document.addEventListener('dragend', () => {
  document.querySelectorAll('.card.drag-over').forEach(el => {
    el.classList.remove('drag-over');
  });
});

// 页面关闭/刷新前强制落盘
window.addEventListener('beforeunload', () => {
  flushSave();
});

// 多标签页数据同步
window.addEventListener('storage', (e) => {
  if (e.key === STORAGE_KEY && !isUpdatingFromStorage) {
    try {
      const newData = JSON.parse(e.newValue);
      if (Array.isArray(newData)) {
        currentList = newData.map(item => ({
          title: typeof item.title === 'string' ? item.title : '未命名周报',
          content: typeof item.content === 'string' ? item.content : ''
        }));
        render();
      }
    } catch (err) {
      console.warn('同步其他标签页数据失败', err);
    }
  }
});

// ========== 顶部操作按钮 ==========
// 新增卡片
document.getElementById('addCardBtn').addEventListener('click', () => {
  flushSave();
  currentList.push({
    title: "新周报",
    content: ""
  });
  saveData(true);
  render();
  // 滚动到最新卡片
  requestAnimationFrame(() => {
    const lastCard = boardEl.lastElementChild;
    if (lastCard) {
      lastCard.scrollIntoView({ behavior: 'smooth', block: 'center' });
      lastCard.querySelector('.card-content').focus();
    }
  });
});

// 导出备份
document.getElementById('exportBtn').addEventListener('click', () => {
  flushSave();
  const blob = new Blob([JSON.stringify(currentList, null, 2)], { type: "application/json" });
  const url = URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `周报看板备份-${formatLocalDate()}.json`;
  a.click();
  URL.revokeObjectURL(url);
});

// 导入备份
document.getElementById('importBtn').addEventListener('click', () => {
  fileInput.click();
});

fileInput.addEventListener('change', (e) => {
  const f = e.target.files[0];
  if(!f) return;

  const reader = new FileReader();
  reader.onload = ev => {
    try {
      const arr = JSON.parse(ev.target.result);
      if (!Array.isArray(arr)) throw new Error('数据格式错误');

      const confirmMsg = `导入将覆盖当前所有 ${currentList.length} 条周报数据，确定继续吗？`;
      if(!confirm(confirmMsg)) {
        fileInput.value = '';
        return;
      }

      isUpdatingFromStorage = true;
      currentList = arr.map(item => ({
        title: typeof item.title === 'string' ? item.title : '未命名周报',
        content: typeof item.content === 'string' ? item.content : ''
      }));

      saveData(true);
      render();
      alert("✅ 导入成功！");
    } catch(err) {
      alert("❌ 文件解析失败，不是合法的备份文件");
    } finally {
      fileInput.value = '';
      isUpdatingFromStorage = false;
    }
  };
  reader.readAsText(f);
});

// ========== 初始化 ==========
currentList = loadData();
render();
</script>
</body>
</html>
