<template>
  <div class="game-container">
    <!-- 标题 -->
    <header class="game-header">
      <h1 class="game-title">🎮 推箱子</h1>
      <p class="game-subtitle">Sokoban</p>
    </header>

    <!-- 游戏信息 -->
    <div class="game-info" v-if="isPlaying">
      <div class="level-info">
        <span class="level-badge">第 {{ isPlayingCustom ? '自' + (customLevelIndex + 1) : currentLevel + 1 }} 关</span>
        <span class="level-name">{{ isPlayingCustom ? customLevels[customLevelIndex]?.name : currentLevelData?.name }}</span>
        <span class="level-total" v-if="!isPlayingCustom">{{ currentLevel + 1 }}/{{ totalLevels }}</span>
      </div>
      <div class="stats">
        <div class="stat">
          <span class="stat-icon">👣</span>
          <span class="stat-value">{{ moves }}</span>
          <span class="stat-label">步数</span>
        </div>
        <div class="stat">
          <span class="stat-icon">📦</span>
          <span class="stat-value">{{ pushes }}</span>
          <span class="stat-label">推动</span>
        </div>
        <div class="stat">
          <span class="stat-icon">✅</span>
          <span class="stat-value">{{ boxesOnTarget }}/{{ boxCount }}</span>
          <span class="stat-label">完成</span>
        </div>
        <div class="stat">
          <span class="stat-icon">⏱</span>
          <span class="stat-value">{{ formattedTime }}</span>
          <span class="stat-label">用时</span>
        </div>
      </div>
    </div>

    <!-- 游戏地图 -->
    <div class="map-container">
      <div class="map" :style="mapStyle">
        <div v-for="(row, y) in gameMap" :key="'row-' + y" class="row">
          <template v-for="(cell, x) in row" :key="'cell-' + y + '-' + x">
            <div
              v-if="cell !== 0 || visibleCells[y]?.[x]"
              :class="getCellClass(cell, y, x)"
              @click="showTutorial = false"
            >
              <span class="cell-content">{{ getCellContent(cell) }}</span>
            </div>
            <div v-else class="cell void"></div>
          </template>
        </div>
      </div>
    </div>

    <!-- 提示消息 -->
    <transition name="fade">
      <div v-if="showMessage" class="game-message">
        {{ message }}
      </div>
    </transition>

    <!-- 方向控制（移动端） -->
    <div class="direction-controls">
      <div class="d-pad">
        <button class="d-btn d-up" @click="movePlayer(0, -1)" @touchstart.prevent="movePlayer(0, -1)">▲</button>
        <button class="d-btn d-left" @click="movePlayer(-1, 0)" @touchstart.prevent="movePlayer(-1, 0)">◀</button>
        <button class="d-btn d-center" @click="restartLevel">🔄</button>
        <button class="d-btn d-right" @click="movePlayer(1, 0)" @touchstart.prevent="movePlayer(1, 0)">▶</button>
        <button class="d-btn d-down" @click="movePlayer(0, 1)" @touchstart.prevent="movePlayer(0, 1)">▼</button>
      </div>
    </div>

    <!-- 胜利弹窗 -->
    <transition name="popup">
      <div v-if="isWin" class="overlay">
        <div class="win-modal">
          <div class="win-header">
            <span class="win-icon">🎉</span>
            <h2>恭喜通关！</h2>
          </div>
          <div class="win-stats">
            <div class="win-stat">
              <span class="win-stat-value">{{ moves }}</span>
              <span class="win-stat-label">总步数</span>
            </div>
            <div class="win-stat">
              <span class="win-stat-value">{{ pushes }}</span>
              <span class="win-stat-label">推动次数</span>
            </div>
            <div class="win-stat">
              <span class="win-stat-value">{{ formattedTime }}</span>
              <span class="win-stat-label">用时</span>
            </div>
            <div class="win-stat" v-if="!isPlayingCustom && bestRecords[currentLevel]">
              <span class="win-stat-value" :class="{ 'new-record': isNewRecord }">{{ bestRecords[currentLevel] }}</span>
              <span class="win-stat-label">{{ isNewRecord ? '🏆 新纪录！' : '最佳记录' }}</span>
            </div>
          </div>
          <div class="win-actions">
            <button class="btn btn-secondary" @click="restartLevel">
              再玩一次
            </button>
            <button
              class="btn btn-primary"
              @click="nextLevel"
              v-if="!isPlayingCustom && currentLevel < totalLevels - 1"
            >
              下一关 →
            </button>
            <button
              class="btn btn-primary"
              @click="showWin = false; isWin = false"
              v-if="isPlayingCustom"
            >
              确定
            </button>
          </div>
        </div>
      </div>
    </transition>

    <!-- 关卡选择弹窗 -->
    <transition name="popup">
      <div v-if="showLevelSelect" class="overlay" @click.self="showLevelSelect = false">
        <div class="level-select-modal">
          <div class="modal-header">
            <h2>选择关卡 ({{ unlockedLevels }}/{{ totalLevels }} 已解锁)</h2>
            <button class="close-btn" @click="showLevelSelect = false">✕</button>
          </div>
          <div class="level-grid">
            <button
              v-for="(level, index) in levels"
              :key="index"
              class="level-btn"
              :class="{
                'locked': index >= unlockedLevels,
                'current': index === currentLevel,
                'completed': bestRecords[index]
              }"
              @click="selectLevel(index)"
              :disabled="index >= unlockedLevels"
            >
              <span class="level-num">{{ index + 1 }}</span>
              <span class="level-name-text">{{ level.name }}</span>
              <span v-if="bestRecords[index]" class="best-score">⭐{{ bestRecords[index] }}</span>
              <span v-if="index >= unlockedLevels" class="lock-icon">🔒</span>
            </button>
          </div>
        </div>
      </div>
    </transition>

    <!-- 教程弹窗 -->
    <transition name="popup">
      <div v-if="showTutorial && !isWin" class="overlay">
        <div class="tutorial-modal">
          <div class="tutorial-header">
            <h2>🎮 推箱子</h2>
            <p class="tutorial-sub">Sokoban</p>
          </div>

          <div class="tutorial-grid">
            <div class="tutorial-card">
              <span class="card-emoji">👷</span>
              <span class="card-text">你</span>
              <span class="card-desc">推动箱子到目标</span>
            </div>
            <div class="tutorial-card">
              <span class="card-emoji">📦</span>
              <span class="card-text">箱子</span>
              <span class="card-desc">推到 ○ 处</span>
            </div>
            <div class="tutorial-card">
              <span class="card-emoji">○</span>
              <span class="card-text" style="color:#64ffda">目标点</span>
              <span class="card-desc">箱子的终点</span>
            </div>
            <div class="tutorial-card">
              <span class="card-emoji">✅</span>
              <span class="card-text" style="color:#48bb78">完成</span>
              <span class="card-desc">箱子在目标上</span>
            </div>
          </div>

          <div class="tutorial-keys-row">
            <div class="key-chip"><kbd>↑↓←→</kbd> 移动</div>
            <div class="key-chip"><kbd>WASD</kbd> 移动</div>
            <div class="key-chip"><kbd>Ctrl+Z</kbd> 撤销</div>
            <div class="key-chip"><kbd>R</kbd> 重置</div>
          </div>

          <button class="btn btn-primary btn-large tutorial-start" @click="startGame">
            开始游戏
          </button>
        </div>
      </div>
    </transition>

    <!-- 关卡编辑器弹窗 -->
    <transition name="popup">
      <div v-if="showLevelEditor" class="overlay">
        <div class="level-editor-modal">
          <div class="modal-header">
            <h2>创建自定义关卡</h2>
            <button class="close-btn" @click="showLevelEditor = false">✕</button>
          </div>
          <div class="editor-content">
            <div class="editor-toolbar">
              <div class="tool-group">
                <span class="tool-label">元素</span>
                <div class="tool-buttons">
                  <button
                    v-for="tool in editorTools"
                    :key="tool.type"
                    class="tool-btn"
                    :class="{ active: selectedTool === tool.type }"
                    @click="selectedTool = tool.type"
                    :title="tool.name"
                  >
                    {{ tool.symbol }}
                  </button>
                </div>
              </div>
              <div class="editor-divider"></div>
              <div class="editor-row">
                <label>大小</label>
                <select v-model="editorHeight">
                  <option v-for="h in [5,6,7,8,9,10,11,12]" :key="h" :value="h">{{ h }}行</option>
                </select>
                <span style="color:#5a6a7a">×</span>
                <select v-model="editorWidth">
                  <option v-for="w in [5,6,7,8,9,10,11,12]" :key="w" :value="w">{{ w }}列</option>
                </select>
                <button class="btn btn-secondary btn-sm" @click="initEditorMap">应用</button>
              </div>
              <div class="editor-divider"></div>
              <div class="editor-row">
                <label>名称</label>
                <input type="text" v-model="customLevelName" placeholder="关卡名称" maxlength="10">
              </div>
            </div>
            <div class="editor-map">
              <div class="map editor-grid" :style="editorMapStyle">
                <div v-for="(row, y) in editorMap" :key="'edit-row-' + y" class="row">
                  <div
                    v-for="(cell, x) in row"
                    :key="'edit-cell-' + y + '-' + x"
                    :class="getEditorCellClass(cell)"
                    @mousedown.prevent="startPaint(y, x)"
                    @mouseenter="dragPaint(y, x)"
                    @touchstart.prevent="startPaint(y, x)"
                  >
                    <span class="cell-content">{{ getEditorCellContent(cell) }}</span>
                  </div>
                </div>
              </div>
            </div>
            <div class="editor-validation" v-if="editorError">
              <span class="error-text">{{ editorError }}</span>
            </div>
            <div class="editor-actions">
              <button class="btn btn-secondary" @click="clearEditor">清空</button>
              <button class="btn btn-secondary" @click="loadRandomTemplate">随机模板</button>
              <button class="btn btn-primary" @click="saveCustomLevel">保存关卡</button>
            </div>
          </div>
        </div>
      </div>
    </transition>

    <!-- 自定义关卡选择弹窗 -->
    <transition name="popup">
      <div v-if="showCustomLevels" class="overlay" @click.self="showCustomLevels = false">
        <div class="level-select-modal">
          <div class="modal-header">
            <h2>我的关卡 ({{ customLevels.length }})</h2>
            <button class="close-btn" @click="showCustomLevels = false">✕</button>
          </div>
          <div class="level-grid" v-if="customLevels.length > 0">
            <div
              v-for="(level, index) in customLevels"
              :key="'custom-' + index"
              class="level-item"
            >
              <button class="level-btn" @click="playCustomLevel(index)">
                <span class="level-num">自{{ index + 1 }}</span>
                <span class="level-name-text">{{ level.name }}</span>
              </button>
              <button class="delete-btn" @click.stop="deleteCustomLevel(index)" title="删除">✕</button>
            </div>
          </div>
          <div v-else class="no-levels">
            <p>还没有自定义关卡</p>
            <p class="hint">点击"自定义"按钮创建新关卡</p>
          </div>
        </div>
      </div>
    </transition>

    <!-- 页脚 -->
    <footer class="game-footer">
      <div class="toolbar">
        <button class="toolbar-btn" @click="undo" :disabled="!canUndo" title="撤销 (Ctrl+Z)">
          <span class="toolbar-icon">↩</span>
          <span class="toolbar-label">撤销</span>
          <span v-if="undoCount > 0" class="toolbar-badge">{{ undoCount }}</span>
        </button>
        <button class="toolbar-btn" @click="restartLevel" title="重新开始 (R)">
          <span class="toolbar-icon">⟳</span>
          <span class="toolbar-label">重置</span>
        </button>
        <button class="toolbar-btn toolbar-btn-primary" @click="showLevelSelect = true" title="关卡选择 (Esc)">
          <span class="toolbar-icon">☰</span>
          <span class="toolbar-label">关卡</span>
        </button>
        <button class="toolbar-btn" @click="openEditor" title="自定义关卡">
          <span class="toolbar-icon">✚</span>
          <span class="toolbar-label">自定义</span>
        </button>
        <button class="toolbar-btn" @click="showCustomLevels = true" title="我的关卡">
          <span class="toolbar-icon">📁</span>
          <span class="toolbar-label">我的</span>
        </button>
        <button class="toolbar-btn" @click="resetToInitial" title="返回主菜单">
          <span class="toolbar-icon">🏠</span>
          <span class="toolbar-label">菜单</span>
        </button>
      </div>
      <p class="footer-hint">ESC 关卡 | R 重置 | Ctrl+Z 撤销</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, watch } from 'vue'
import { levels, CELL_TYPES } from './levels.js'

// 游戏状态
const currentLevel = ref(0)
const gameMap = ref([])
const playerPos = reactive({ x: 0, y: 0 })
const moves = ref(0)
const pushes = ref(0)
const isWin = ref(false)
const isPlaying = ref(false)
const showLevelSelect = ref(false)
const showTutorial = ref(true)
const showLevelEditor = ref(false)
const showCustomLevels = ref(false)
const showWin = ref(false)
const history = ref([])
const animatingCells = reactive(new Set())
const message = ref('')
const showMessage = ref(false)
const isNewRecord = ref(false)

// 计时器
const elapsedTime = ref(0)
let timerInterval = null

const formattedTime = computed(() => {
  const m = Math.floor(elapsedTime.value / 60)
  const s = elapsedTime.value % 60
  return `${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`
})

const startTimer = () => {
  stopTimer()
  elapsedTime.value = 0
  timerInterval = setInterval(() => {
    elapsedTime.value++
  }, 1000)
}

const stopTimer = () => {
  if (timerInterval) {
    clearInterval(timerInterval)
    timerInterval = null
  }
}

// 关卡编辑器状态
const editorWidth = ref(8)
const editorHeight = ref(8)
const editorMap = ref([])
const selectedTool = ref(CELL_TYPES.WALL)
const customLevelName = ref('')
const editorError = ref('')
const isPainting = ref(false)
const isPlayingCustom = ref(false)
const customLevelIndex = ref(-1)

// 编辑器工具
const editorTools = [
  { type: CELL_TYPES.EMPTY, name: '空地', symbol: '·' },
  { type: CELL_TYPES.WALL, name: '墙', symbol: '■' },
  { type: CELL_TYPES.TARGET, name: '目标点', symbol: '○' },
  { type: CELL_TYPES.BOX, name: '箱子', symbol: '📦' },
  { type: CELL_TYPES.PLAYER, name: '玩家', symbol: '👷' }
]

// 已解锁关卡
const unlockedLevels = ref(parseInt(localStorage.getItem('sokoban_unlocked') || '1'))

// 最佳记录
const bestRecords = ref(JSON.parse(localStorage.getItem('sokoban_records') || '{}'))

// 自定义关卡
const customLevels = ref(JSON.parse(localStorage.getItem('sokoban_custom_levels') || '[]'))

// 计算属性
const currentLevelData = computed(() => levels[currentLevel.value])
const totalLevels = computed(() => levels.length)
const canUndo = computed(() => history.value.length > 0)
const undoCount = computed(() => history.value.length)

const boxCount = computed(() => {
  let count = 0
  const map = gameMap.value
  for (let y = 0; y < map.length; y++) {
    const row = map[y]
    for (let x = 0; x < row.length; x++) {
      if (row[x] === CELL_TYPES.BOX || row[x] === CELL_TYPES.BOX_ON_TARGET) count++
    }
  }
  return count
})

const boxesOnTarget = computed(() => {
  let count = 0
  const map = gameMap.value
  for (let y = 0; y < map.length; y++) {
    const row = map[y]
    for (let x = 0; x < row.length; x++) {
      if (row[x] === CELL_TYPES.BOX_ON_TARGET) count++
    }
  }
  return count
})

const mapStyle = computed(() => {
  const map = gameMap.value
  if (map.length === 0) return {}
  const cols = map[0].length
  return {
    gridTemplateColumns: `repeat(${cols}, 1fr)`,
    maxWidth: `${cols * 47 + 20}px`
  }
})

const editorMapStyle = computed(() => ({
  gridTemplateColumns: `repeat(${editorWidth.value}, 1fr)`,
  maxWidth: `${editorWidth.value * 47 + 20}px`
}))

// 预计算可见单元格（哪些 void 格子需要渲染）
const visibleCells = computed(() => {
  const map = gameMap.value
  if (map.length === 0) return []
  const result = []
  for (let y = 0; y < map.length; y++) {
    result[y] = []
    for (let x = 0; x < map[y].length; x++) {
      if (map[y][x] !== 0) {
        result[y][x] = true
        continue
      }
      // 检查四周是否有非零格子
      let hasNeighbor = false
      const dirs = [[-1,0],[1,0],[0,-1],[0,1]]
      for (const [dy, dx] of dirs) {
        const ny = y + dy, nx = x + dx
        if (ny >= 0 && ny < map.length && nx >= 0 && nx < map[ny].length && map[ny][nx] !== 0) {
          hasNeighbor = true
          break
        }
      }
      result[y][x] = hasNeighbor
    }
  }
  return result
})

// 开始游戏（从教程进入）
const startGame = () => {
  showTutorial.value = false
  initLevel(currentLevel.value)
}

// 初始化编辑器地图
const initEditorMap = () => {
  const newMap = []
  for (let y = 0; y < editorHeight.value; y++) {
    const row = []
    for (let x = 0; x < editorWidth.value; x++) {
      if (y === 0 || y === editorHeight.value - 1 || x === 0 || x === editorWidth.value - 1) {
        row.push(CELL_TYPES.WALL)
      } else {
        row.push(CELL_TYPES.EMPTY)
      }
    }
    newMap.push(row)
  }
  editorMap.value = newMap
  editorError.value = ''
}

// 编辑器绘制
const startPaint = (y, x) => {
  isPainting.value = true
  paintCell(y, x)
}

const dragPaint = (y, x) => {
  if (isPainting.value) {
    paintCell(y, x)
  }
}

const paintCell = (y, x) => {
  if (selectedTool.value === CELL_TYPES.PLAYER) {
    for (let row of editorMap.value) {
      for (let i = 0; i < row.length; i++) {
        if (row[i] === CELL_TYPES.PLAYER || row[i] === CELL_TYPES.PLAYER_ON_TARGET) {
          row[i] = CELL_TYPES.EMPTY
        }
      }
    }
  }

  if (selectedTool.value === CELL_TYPES.BOX && editorMap.value[y][x] === CELL_TYPES.TARGET) {
    editorMap.value[y][x] = CELL_TYPES.BOX_ON_TARGET
  } else if (selectedTool.value === CELL_TYPES.TARGET && editorMap.value[y][x] === CELL_TYPES.BOX) {
    editorMap.value[y][x] = CELL_TYPES.BOX_ON_TARGET
  } else if (selectedTool.value === CELL_TYPES.PLAYER && editorMap.value[y][x] === CELL_TYPES.TARGET) {
    editorMap.value[y][x] = CELL_TYPES.PLAYER_ON_TARGET
  } else {
    editorMap.value[y][x] = selectedTool.value
  }

  editorError.value = ''
}

const clearEditor = () => {
  initEditorMap()
  customLevelName.value = ''
}

const getEditorCellClass = (cell) => {
  const classes = ['cell']
  switch (cell) {
    case CELL_TYPES.WALL: classes.push('wall'); break
    case CELL_TYPES.TARGET: classes.push('target'); break
    case CELL_TYPES.BOX: classes.push('box'); break
    case CELL_TYPES.BOX_ON_TARGET: classes.push('box', 'on-target'); break
    case CELL_TYPES.PLAYER: classes.push('player'); break
    case CELL_TYPES.PLAYER_ON_TARGET: classes.push('player', 'on-target'); break
    default: classes.push('empty')
  }
  return classes
}

const getEditorCellContent = (cell) => {
  switch (cell) {
    case CELL_TYPES.TARGET: return '○'
    case CELL_TYPES.BOX:
    case CELL_TYPES.BOX_ON_TARGET: return '📦'
    case CELL_TYPES.PLAYER:
    case CELL_TYPES.PLAYER_ON_TARGET: return '👷'
    default: return ''
  }
}

const validateLevel = () => {
  let playerCount = 0, boxCount = 0, targetCount = 0
  for (let row of editorMap.value) {
    for (let cell of row) {
      if (cell === CELL_TYPES.PLAYER || cell === CELL_TYPES.PLAYER_ON_TARGET) playerCount++
      if (cell === CELL_TYPES.BOX || cell === CELL_TYPES.BOX_ON_TARGET) boxCount++
      if (cell === CELL_TYPES.TARGET || cell === CELL_TYPES.BOX_ON_TARGET || cell === CELL_TYPES.PLAYER_ON_TARGET) targetCount++
    }
  }
  if (playerCount !== 1) return '必须且只能有一个玩家'
  if (boxCount === 0) return '至少需要一个箱子'
  if (targetCount === 0) return '至少需要一个目标点'
  if (boxCount !== targetCount) return `箱子数量(${boxCount})必须等于目标点数量(${targetCount})`
  return null
}

const saveCustomLevel = () => {
  const error = validateLevel()
  if (error) { editorError.value = error; return }
  const newLevel = {
    name: customLevelName.value || `关卡${customLevels.value.length + 1}`,
    map: editorMap.value.map(row => [...row])
  }
  customLevels.value.push(newLevel)
  localStorage.setItem('sokoban_custom_levels', JSON.stringify(customLevels.value))
  showLevelEditor.value = false
  showGameMessage('关卡已保存！')
}

const deleteCustomLevel = (index) => {
  customLevels.value.splice(index, 1)
  localStorage.setItem('sokoban_custom_levels', JSON.stringify(customLevels.value))
}

const playCustomLevel = (index) => {
  isPlayingCustom.value = true
  customLevelIndex.value = index
  loadMap(customLevels.value[index].map)
  showCustomLevels.value = false
  startTimer()
}

const openEditor = () => {
  showLevelEditor.value = true
  editorError.value = ''
  initEditorMap()
}

const loadRandomTemplate = () => {
  const templates = [
    {
      name: '简单房间',
      map: [
        [1,1,1,1,1,1,1],
        [1,0,0,0,0,0,1],
        [1,0,0,2,3,0,1],
        [1,0,0,0,0,0,1],
        [1,0,4,0,0,0,1],
        [1,1,1,1,1,1,1]
      ]
    },
    {
      name: '双箱挑战',
      map: [
        [1,1,1,1,1,1,1,1],
        [1,0,0,0,0,0,0,1],
        [1,0,2,0,3,0,2,1],
        [1,0,0,0,3,0,0,1],
        [1,0,0,0,4,0,0,1],
        [1,1,1,1,1,1,1,1]
      ]
    },
    {
      name: 'L形走廊',
      map: [
        [1,1,1,1,1,1,1],
        [1,2,0,0,0,0,1],
        [1,0,1,0,0,0,1],
        [1,0,3,0,0,0,1],
        [1,0,0,4,0,0,1],
        [1,1,1,1,1,1,1]
      ]
    }
  ]
  const template = templates[Math.floor(Math.random() * templates.length)]
  editorHeight.value = template.map.length
  editorWidth.value = template.map[0].length
  editorMap.value = template.map.map(row => [...row])
  customLevelName.value = template.name
  editorError.value = ''
}

// 加载地图到游戏
const loadMap = (mapData) => {
  gameMap.value = mapData.map(row => [...row])
  moves.value = 0
  pushes.value = 0
  isWin.value = false
  history.value = []
  isPlaying.value = true
  isNewRecord.value = false

  for (let y = 0; y < gameMap.value.length; y++) {
    for (let x = 0; x < gameMap.value[y].length; x++) {
      const cell = gameMap.value[y][x]
      if (cell === CELL_TYPES.PLAYER || cell === CELL_TYPES.PLAYER_ON_TARGET) {
        playerPos.x = x
        playerPos.y = y
      }
    }
  }
}

// 初始化关卡
const initLevel = (levelIndex) => {
  currentLevel.value = levelIndex
  isPlayingCustom.value = false
  customLevelIndex.value = -1
  loadMap(levels[levelIndex].map)
  startTimer()
}

// 保存状态到历史
const saveState = () => {
  history.value.push({
    map: gameMap.value.map(row => [...row]),
    playerPos: { x: playerPos.x, y: playerPos.y },
    moves: moves.value,
    pushes: pushes.value
  })
  if (history.value.length > 200) {
    history.value.shift()
  }
}

// 撤销
const undo = () => {
  if (history.value.length === 0) return
  const state = history.value.pop()
  gameMap.value = state.map
  playerPos.x = state.playerPos.x
  playerPos.y = state.playerPos.y
  moves.value = state.moves
  pushes.value = state.pushes
  isWin.value = false
}

// 重置到初始状态
const resetToInitial = () => {
  stopTimer()
  isPlaying.value = false
  showTutorial.value = true
  isPlayingCustom.value = false
  customLevelIndex.value = -1
  showLevelSelect.value = false
  showLevelEditor.value = false
  showCustomLevels.value = false
  isWin.value = false
  currentLevel.value = 0
  gameMap.value = []
  showGameMessage('已返回主菜单')
}

// 显示提示消息
const showGameMessage = (msg) => {
  message.value = msg
  showMessage.value = true
  setTimeout(() => { showMessage.value = false }, 1500)
}

// 添加动画
const addAnimation = (y, x, duration = 150) => {
  const key = `${y}-${x}`
  animatingCells.add(key)
  setTimeout(() => { animatingCells.delete(key) }, duration)
}

// 移动玩家
const movePlayer = (dx, dy) => {
  if (isWin.value || !isPlaying.value) return

  const newX = playerPos.x + dx
  const newY = playerPos.y + dy

  if (newY < 0 || newY >= gameMap.value.length || newX < 0 || newX >= gameMap.value[newY].length) return

  const targetCell = gameMap.value[newY][newX]

  if (targetCell === CELL_TYPES.WALL) return

  addAnimation(newY, newX)

  if (targetCell === CELL_TYPES.EMPTY || targetCell === CELL_TYPES.TARGET) {
    saveState()
    const oldCell = gameMap.value[playerPos.y][playerPos.x]
    gameMap.value[playerPos.y][playerPos.x] = oldCell === CELL_TYPES.PLAYER_ON_TARGET ? CELL_TYPES.TARGET : CELL_TYPES.EMPTY
    gameMap.value[newY][newX] = targetCell === CELL_TYPES.TARGET ? CELL_TYPES.PLAYER_ON_TARGET : CELL_TYPES.PLAYER
    playerPos.x = newX
    playerPos.y = newY
    moves.value++
  } else if (targetCell === CELL_TYPES.BOX || targetCell === CELL_TYPES.BOX_ON_TARGET) {
    const boxNewX = newX + dx
    const boxNewY = newY + dy

    if (boxNewY < 0 || boxNewY >= gameMap.value.length || boxNewX < 0 || boxNewX >= gameMap.value[boxNewY].length) return

    const boxTargetCell = gameMap.value[boxNewY][boxNewX]

    if (boxTargetCell === CELL_TYPES.WALL || boxTargetCell === CELL_TYPES.BOX || boxTargetCell === CELL_TYPES.BOX_ON_TARGET) return

    saveState()
    addAnimation(boxNewY, boxNewX)

    const oldCell = gameMap.value[playerPos.y][playerPos.x]
    gameMap.value[playerPos.y][playerPos.x] = oldCell === CELL_TYPES.PLAYER_ON_TARGET ? CELL_TYPES.TARGET : CELL_TYPES.EMPTY

    gameMap.value[newY][newX] = targetCell === CELL_TYPES.BOX_ON_TARGET ? CELL_TYPES.PLAYER_ON_TARGET : CELL_TYPES.PLAYER
    gameMap.value[boxNewY][boxNewX] = boxTargetCell === CELL_TYPES.TARGET ? CELL_TYPES.BOX_ON_TARGET : CELL_TYPES.BOX

    playerPos.x = newX
    playerPos.y = newY
    moves.value++
    pushes.value++

    checkWin()
  }
}

// 检查胜利条件
const checkWin = () => {
  const map = gameMap.value
  for (let y = 0; y < map.length; y++) {
    const row = map[y]
    for (let x = 0; x < row.length; x++) {
      if (row[x] === CELL_TYPES.BOX) return
    }
  }

  // 所有箱子都在目标点上
  isWin.value = true
  showWin.value = true
  stopTimer()

  if (!isPlayingCustom.value) {
    // 解锁下一关
    if (currentLevel.value + 2 > unlockedLevels.value) {
      unlockedLevels.value = currentLevel.value + 2
      localStorage.setItem('sokoban_unlocked', unlockedLevels.value.toString())
    }

    // 保存最佳记录
    const recordKey = currentLevel.value
    if (!bestRecords.value[recordKey] || moves.value < bestRecords.value[recordKey]) {
      bestRecords.value[recordKey] = moves.value
      localStorage.setItem('sokoban_records', JSON.stringify(bestRecords.value))
      isNewRecord.value = true
    } else {
      isNewRecord.value = false
    }
  }
}

// 重新开始当前关卡
const restartLevel = () => {
  if (isPlayingCustom.value && customLevelIndex.value >= 0) {
    loadMap(customLevels.value[customLevelIndex.value].map)
    startTimer()
  } else {
    initLevel(currentLevel.value)
  }
}

// 选择关卡
const selectLevel = (index) => {
  if (index < unlockedLevels.value) {
    initLevel(index)
    showLevelSelect.value = false
  }
}

// 下一关
const nextLevel = () => {
  if (currentLevel.value < levels.length - 1) {
    initLevel(currentLevel.value + 1)
  }
}

// 键盘控制
const handleKeydown = (e) => {
  if (showLevelSelect.value || showLevelEditor.value || showCustomLevels.value) {
    if (e.key === 'Escape') {
      showLevelSelect.value = false
      showLevelEditor.value = false
      showCustomLevels.value = false
    }
    return
  }

  if (showTutorial.value) {
    startGame()
    return
  }

  // 关闭胜利弹窗
  if (isWin.value && e.key === 'Escape') {
    isWin.value = false
    showWin.value = false
    return
  }

  switch (e.key) {
    case 'ArrowUp': case 'w': case 'W':
      e.preventDefault(); movePlayer(0, -1); break
    case 'ArrowDown': case 's': case 'S':
      e.preventDefault(); movePlayer(0, 1); break
    case 'ArrowLeft': case 'a': case 'A':
      e.preventDefault(); movePlayer(-1, 0); break
    case 'ArrowRight': case 'd': case 'D':
      e.preventDefault(); movePlayer(1, 0); break
    case 'z': case 'Z':
      if (e.ctrlKey || e.metaKey) { e.preventDefault(); undo() }
      break
    case 'r': case 'R':
      restartLevel(); break
    case 'Escape':
      showLevelSelect.value = true; break
  }
}

// 触摸控制（滑动）
let touchStartX = 0, touchStartY = 0
const handleTouchStart = (e) => {
  touchStartX = e.touches[0].clientX
  touchStartY = e.touches[0].clientY
}
const handleTouchEnd = (e) => {
  if (!isPlaying.value || isWin.value) return
  const dx = e.changedTouches[0].clientX - touchStartX
  const dy = e.changedTouches[0].clientY - touchStartY
  const minSwipe = 30
  if (Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > minSwipe) {
    movePlayer(dx > 0 ? 1 : -1, 0)
  } else if (Math.abs(dy) > Math.abs(dx) && Math.abs(dy) > minSwipe) {
    movePlayer(0, dy > 0 ? 1 : -1)
  }
}

// 鼠标事件（编辑器绘制）
const handleMouseDown = () => { isPainting.value = true }
const handleMouseUp = () => { isPainting.value = false }

// 获取单元格类名
const getCellClass = (cell, y, x) => {
  const classes = ['cell']
  const key = `${y}-${x}`
  if (animatingCells.has(key)) classes.push('animating')
  switch (cell) {
    case CELL_TYPES.WALL: classes.push('wall'); break
    case CELL_TYPES.TARGET: classes.push('target'); break
    case CELL_TYPES.BOX: classes.push('box'); break
    case CELL_TYPES.BOX_ON_TARGET: classes.push('box', 'on-target'); break
    case CELL_TYPES.PLAYER: classes.push('player'); break
    case CELL_TYPES.PLAYER_ON_TARGET: classes.push('player', 'on-target'); break
    default: classes.push('empty')
  }
  return classes
}

// 获取单元格内容
const getCellContent = (cell) => {
  switch (cell) {
    case CELL_TYPES.TARGET: return '○'
    case CELL_TYPES.BOX: return '📦'
    case CELL_TYPES.BOX_ON_TARGET: return '📦'
    case CELL_TYPES.PLAYER:
    case CELL_TYPES.PLAYER_ON_TARGET: return '👷'
    default: return ''
  }
}

// 生命周期
onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  document.addEventListener('touchstart', handleTouchStart, { passive: true })
  document.addEventListener('touchend', handleTouchEnd, { passive: true })
  document.addEventListener('mousedown', handleMouseDown)
  document.addEventListener('mouseup', handleMouseUp)
  initLevel(0)
  initEditorMap()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  document.removeEventListener('touchstart', handleTouchStart)
  document.removeEventListener('touchend', handleTouchEnd)
  document.removeEventListener('mousedown', handleMouseDown)
  document.removeEventListener('mouseup', handleMouseUp)
  stopTimer()
})
</script>

<style scoped>
/* 组件特定样式 */
</style>