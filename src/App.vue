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
      </div>
    </div>

    <!-- 游戏地图 -->
    <div class="map-container">
      <div class="map" :style="mapStyle">
        <div v-for="(row, y) in gameMap" :key="'row-' + y" class="row">
          <template v-for="(cell, x) in row" :key="'cell-' + y + '-' + x">
            <div
              v-if="cell !== 0 || hasNonZeroNeighbor(y, x)"
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

    <!-- 提示消息（非居中，顶部显示） -->
    <transition name="fade">
      <div v-if="showMessage" class="game-message">
        {{ message }}
      </div>
    </transition>

    <!-- 控制按钮 -->
    <div class="controls">
      <button class="btn btn-secondary" @click="undo" title="撤销 (Ctrl+Z)">
        <span class="btn-icon">↩</span>
        撤销
      </button>
      <button class="btn btn-secondary" @click="restartLevel" title="重新开始 (R)">
        <span class="btn-icon">⟳</span>
        重置
      </button>
      <button class="btn btn-primary" @click="showLevelSelect = true" title="关卡选择 (Esc)">
        <span class="btn-icon">☰</span>
        关卡
      </button>
      <button class="btn btn-secondary" @click="showLevelEditor = true" title="自定义关卡">
        <span class="btn-icon">✚</span>
        自定义
      </button>
      <button class="btn btn-secondary" @click="showCustomLevels = true" title="我的关卡">
        <span class="btn-icon">📁</span>
        我的关卡
      </button>
      <button class="btn btn-secondary" @click="resetToInitial" title="返回主菜单">
        <span class="btn-icon">🏠</span>
        主菜单
      </button>
    </div>

    <!-- 方向控制（移动端） -->
    <div class="direction-controls">
      <div class="d-pad">
        <button class="d-btn d-up" @click="movePlayer(0, -1)">▲</button>
        <button class="d-btn d-left" @click="movePlayer(-1, 0)">◀</button>
        <button class="d-btn d-center" @click="restartLevel">🔄</button>
        <button class="d-btn d-right" @click="movePlayer(1, 0)">▶</button>
        <button class="d-btn d-down" @click="movePlayer(0, 1)">▼</button>
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
            <div class="win-stat" v-if="bestRecords[currentLevel]">
              <span class="win-stat-value">{{ bestRecords[currentLevel] }}</span>
              <span class="win-stat-label">最佳记录</span>
            </div>
          </div>
          <div class="win-actions">
            <button class="btn btn-secondary" @click="restartLevel">
              再玩一次
            </button>
            <button
              class="btn btn-primary"
              @click="nextLevel"
              v-if="currentLevel < totalLevels - 1"
            >
              下一关 →
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
            <h2>选择关卡</h2>
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
            <span class="tutorial-icon">▲</span>
            <h2>推箱子</h2>
          </div>
          <div class="tutorial-content">
            <div class="tutorial-item">
              <span class="tutorial-symbol player-icon">👷</span>
              <span class="tutorial-text">这是你，推动箱子到目标点</span>
            </div>
            <div class="tutorial-item">
              <span class="tutorial-symbol box-icon">📦</span>
              <span class="tutorial-text">箱子，需要推到目标点</span>
            </div>
            <div class="tutorial-item">
              <span class="tutorial-symbol target-icon">○</span>
              <span class="tutorial-text">目标点，箱子需要放这里</span>
            </div>
            <div class="tutorial-item">
              <span class="tutorial-symbol complete-icon">📦</span>
              <span class="tutorial-text">箱子已在目标点上</span>
            </div>
            <div class="tutorial-keys">
              <h3>操作方式</h3>
              <div class="key-group">
                <span class="key">↑↓←→</span>
                <span class="key-desc">方向键移动</span>
              </div>
              <div class="key-group">
                <span class="key">WASD</span>
                <span class="key-desc">字母键移动</span>
              </div>
              <div class="key-group">
                <span class="key">Ctrl+Z</span>
                <span class="key-desc">撤销</span>
              </div>
              <div class="key-group">
                <span class="key">R</span>
                <span class="key-desc">重新开始</span>
              </div>
            </div>
          </div>
          <button class="btn btn-primary btn-large center" @click="showTutorial = false">
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
            <div class="editor-tools">
              <span class="tool-label">选择元素：</span>
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
            <div class="editor-size">
              <label>地图大小：</label>
              <select v-model="editorHeight">
                <option v-for="h in [5,6,7,8,9,10]" :key="h" :value="h">{{ h }}行</option>
              </select>
              ×
              <select v-model="editorWidth">
                <option v-for="w in [5,6,7,8,9,10,11,12]" :key="w" :value="w">{{ w }}列</option>
              </select>
            </div>
            <div class="editor-name">
              <label>关卡名称：</label>
              <input type="text" v-model="customLevelName" placeholder="输入关卡名称" maxlength="10">
            </div>
            <div class="editor-map">
              <div class="map" :style="editorMapStyle">
                <div v-for="(row, y) in editorMap" :key="'edit-row-' + y" class="row">
                  <div
                    v-for="(cell, x) in row"
                    :key="'edit-cell-' + y + '-' + x"
                    :class="getEditorCellClass(cell)"
                    @click="paintCell(y, x)"
                    @mouseenter="paintCell(y, x, true)"
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
            <h2>我的关卡</h2>
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
      <p>按 ESC 打开关卡选择 | 按 R 重新开始</p>
    </footer>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'
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
const history = ref([])
const animatingCells = ref(new Set())
const message = ref('')
const showMessage = ref(false)

// 关卡编辑器状态
const editorWidth = ref(8)
const editorHeight = ref(8)
const editorMap = ref([])
const selectedTool = ref(CELL_TYPES.WALL)
const customLevelName = ref('')
const editorError = ref('')
const isMouseDown = ref(false)
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
const boxCount = computed(() => {
  let count = 0
  gameMap.value.forEach(row => {
    row.forEach(cell => {
      if (cell === CELL_TYPES.BOX || cell === CELL_TYPES.BOX_ON_TARGET) count++
    })
  })
  return count
})

const boxesOnTarget = computed(() => {
  let count = 0
  gameMap.value.forEach(row => {
    row.forEach(cell => {
      if (cell === CELL_TYPES.BOX_ON_TARGET) count++
    })
  })
  return count
})

const mapStyle = computed(() => {
  if (gameMap.value.length === 0) return {}
  const cols = gameMap.value[0].length
  return {
    gridTemplateColumns: `repeat(${cols}, 1fr)`,
    maxWidth: `${cols * 50 + 20}px`
  }
})

// 编辑器地图样式
const editorMapStyle = computed(() => {
  return {
    gridTemplateColumns: `repeat(${editorWidth.value}, 1fr)`,
    maxWidth: `${editorWidth.value * 50 + 20}px`
  }
})

// 检查周围是否有非空格子
const hasNonZeroNeighbor = (y, x) => {
  const map = gameMap.value
  const directions = [[-1, 0], [1, 0], [0, -1], [0, 1]]
  for (const [dy, dx] of directions) {
    const ny = y + dy
    const nx = x + dx
    if (ny >= 0 && ny < map.length && nx >= 0 && nx < map[ny].length) {
      if (map[ny][nx] !== 0) return true
    }
  }
  return false
}

// 初始化编辑器地图
const initEditorMap = () => {
  const newMap = []
  for (let y = 0; y < editorHeight.value; y++) {
    const row = []
    for (let x = 0; x < editorWidth.value; x++) {
      // 默认边缘是墙，中间是空地
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

// 绘制格子
const paintCell = (y, x, isDrag = false) => {
  if (isDrag && !isMouseDown.value) return

  // 如果放置玩家，先移除现有玩家
  if (selectedTool.value === CELL_TYPES.PLAYER) {
    for (let row of editorMap.value) {
      for (let i = 0; i < row.length; i++) {
        if (row[i] === CELL_TYPES.PLAYER) {
          row[i] = CELL_TYPES.EMPTY
        }
      }
    }
  }

  // 如果放置箱子到目标点
  if (selectedTool.value === CELL_TYPES.BOX && editorMap.value[y][x] === CELL_TYPES.TARGET) {
    editorMap.value[y][x] = CELL_TYPES.BOX_ON_TARGET
  }
  // 如果放置目标点且上面有箱子
  else if (selectedTool.value === CELL_TYPES.TARGET && editorMap.value[y][x] === CELL_TYPES.BOX) {
    editorMap.value[y][x] = CELL_TYPES.BOX_ON_TARGET
  }
  // 如果放置玩家到目标点
  else if (selectedTool.value === CELL_TYPES.PLAYER && editorMap.value[y][x] === CELL_TYPES.TARGET) {
    editorMap.value[y][x] = CELL_TYPES.PLAYER_ON_TARGET
  }
  else {
    editorMap.value[y][x] = selectedTool.value
  }

  editorError.value = ''
}

// 清空编辑器
const clearEditor = () => {
  initEditorMap()
  customLevelName.value = ''
}

// 获取编辑器格子类名
const getEditorCellClass = (cell) => {
  const classes = ['cell']
  switch (cell) {
    case CELL_TYPES.WALL:
      classes.push('wall')
      break
    case CELL_TYPES.TARGET:
      classes.push('target')
      break
    case CELL_TYPES.BOX:
      classes.push('box')
      break
    case CELL_TYPES.BOX_ON_TARGET:
      classes.push('box', 'on-target')
      break
    case CELL_TYPES.PLAYER:
      classes.push('player')
      break
    case CELL_TYPES.PLAYER_ON_TARGET:
      classes.push('player', 'on-target')
      break
    default:
      classes.push('empty')
  }
  return classes
}

// 获取编辑器格子内容
const getEditorCellContent = (cell) => {
  switch (cell) {
    case CELL_TYPES.WALL:
      return ''
    case CELL_TYPES.TARGET:
      return '○'
    case CELL_TYPES.BOX:
      return '📦'
    case CELL_TYPES.BOX_ON_TARGET:
      return '📦'
    case CELL_TYPES.PLAYER:
    case CELL_TYPES.PLAYER_ON_TARGET:
      return '👷'
    default:
      return '·'
  }
}

// 验证关卡
const validateLevel = () => {
  let playerCount = 0
  let boxCount = 0
  let targetCount = 0

  for (let row of editorMap.value) {
    for (let cell of row) {
      if (cell === CELL_TYPES.PLAYER || cell === CELL_TYPES.PLAYER_ON_TARGET) playerCount++
      if (cell === CELL_TYPES.BOX || cell === CELL_TYPES.BOX_ON_TARGET) boxCount++
      if (cell === CELL_TYPES.TARGET || cell === CELL_TYPES.BOX_ON_TARGET || cell === CELL_TYPES.PLAYER_ON_TARGET) targetCount++
    }
  }

  if (playerCount !== 1) {
    return '必须且只能有一个玩家'
  }
  if (boxCount === 0) {
    return '至少需要一个箱子'
  }
  if (targetCount === 0) {
    return '至少需要一个目标点'
  }
  if (boxCount !== targetCount) {
    return `箱子数量(${boxCount})必须等于目标点数量(${targetCount})`
  }

  return null
}

// 保存自定义关卡
const saveCustomLevel = () => {
  const error = validateLevel()
  if (error) {
    editorError.value = error
    return
  }

  const newLevel = {
    name: customLevelName.value || `关卡${customLevels.value.length + 1}`,
    map: editorMap.value.map(row => [...row])
  }

  customLevels.value.push(newLevel)
  localStorage.setItem('sokoban_custom_levels', JSON.stringify(customLevels.value))

  showLevelEditor.value = false
  showGameMessage('关卡已保存！')
}

// 删除自定义关卡
const deleteCustomLevel = (index) => {
  customLevels.value.splice(index, 1)
  localStorage.setItem('sokoban_custom_levels', JSON.stringify(customLevels.value))
}

// 游玩自定义关卡
const playCustomLevel = (index) => {
  isPlayingCustom.value = true
  customLevelIndex.value = index
  const level = customLevels.value[index]
  gameMap.value = level.map.map(row => [...row])
  moves.value = 0
  pushes.value = 0
  isWin.value = false
  history.value = []
  isPlaying.value = true

  // 找到玩家位置
  for (let y = 0; y < gameMap.value.length; y++) {
    for (let x = 0; x < gameMap.value[y].length; x++) {
      if (gameMap.value[y][x] === CELL_TYPES.PLAYER || gameMap.value[y][x] === CELL_TYPES.PLAYER_ON_TARGET) {
        playerPos.x = x
        playerPos.y = y
      }
    }
  }

  showCustomLevels.value = false
}

// 加载随机模板
const loadRandomTemplate = () => {
  const templates = [
    // 简单关卡模板
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
    // 双箱关卡模板
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
    // L形关卡模板
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

// 初始化关卡
const initLevel = (levelIndex) => {
  currentLevel.value = levelIndex
  const level = levels[levelIndex]
  gameMap.value = level.map.map(row => [...row])
  moves.value = 0
  pushes.value = 0
  isWin.value = false
  history.value = []
  isPlaying.value = true

  // 找到玩家位置
  for (let y = 0; y < gameMap.value.length; y++) {
    for (let x = 0; x < gameMap.value[y].length; x++) {
      if (gameMap.value[y][x] === CELL_TYPES.PLAYER || gameMap.value[y][x] === CELL_TYPES.PLAYER_ON_TARGET) {
        playerPos.x = x
        playerPos.y = y
      }
    }
  }
}

// 保存状态到历史
const saveState = () => {
  history.value.push({
    map: gameMap.value.map(row => [...row]),
    playerPos: { ...playerPos },
    moves: moves.value,
    pushes: pushes.value
  })
  // 限制历史记录数量
  if (history.value.length > 100) {
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
}

// 重置到初始状态
const resetToInitial = () => {
  currentLevel.value = 0
  initLevel(0)
  showTutorial.value = true
  isPlayingCustom.value = false
  customLevelIndex.value = -1
  showLevelSelect.value = false
  showLevelEditor.value = false
  showCustomLevels.value = false
  showGameMessage('已返回主菜单')
}

// 显示提示消息
const showGameMessage = (msg) => {
  message.value = msg
  showMessage.value = true
  setTimeout(() => {
    showMessage.value = false
  }, 1500)
}

// 移动玩家
const movePlayer = (dx, dy) => {
  if (isWin.value) return

  const newX = playerPos.x + dx
  const newY = playerPos.y + dy

  // 检查边界
  if (newY < 0 || newY >= gameMap.value.length || newX < 0 || newX >= gameMap.value[newY].length) {
    return
  }

  const targetCell = gameMap.value[newY][newX]

  // 撞墙
  if (targetCell === CELL_TYPES.WALL) {
    return
  }

  // 添加动画
  const animKey = `${newY}-${newX}`
  animatingCells.value.add(animKey)

  setTimeout(() => {
    animatingCells.value.delete(animKey)
  }, 150)

  // 空地或目标点
  if (targetCell === CELL_TYPES.EMPTY || targetCell === CELL_TYPES.TARGET) {
    saveState()

    // 更新原位置
    const oldCell = gameMap.value[playerPos.y][playerPos.x]
    gameMap.value[playerPos.y][playerPos.x] = oldCell === CELL_TYPES.PLAYER_ON_TARGET ? CELL_TYPES.TARGET : CELL_TYPES.EMPTY

    // 更新新位置
    gameMap.value[newY][newX] = targetCell === CELL_TYPES.TARGET ? CELL_TYPES.PLAYER_ON_TARGET : CELL_TYPES.PLAYER

    playerPos.x = newX
    playerPos.y = newY
    moves.value++
  }
  // 推箱子
  else if (targetCell === CELL_TYPES.BOX || targetCell === CELL_TYPES.BOX_ON_TARGET) {
    const boxNewX = newX + dx
    const boxNewY = newY + dy

    // 检查箱子移动目标
    if (boxNewY < 0 || boxNewY >= gameMap.value.length || boxNewX < 0 || boxNewX >= gameMap.value[boxNewY].length) {
      return
    }

    const boxTargetCell = gameMap.value[boxNewY][boxNewX]

    // 箱子不能推到墙或另一个箱子
    if (boxTargetCell === CELL_TYPES.WALL || boxTargetCell === CELL_TYPES.BOX || boxTargetCell === CELL_TYPES.BOX_ON_TARGET) {
      return
    }

    saveState()

    // 添加箱子动画
    const boxAnimKey = `${boxNewY}-${boxNewX}`
    animatingCells.value.add(boxAnimKey)
    setTimeout(() => {
      animatingCells.value.delete(boxAnimKey)
    }, 150)

    // 更新原位置
    const oldCell = gameMap.value[playerPos.y][playerPos.x]
    gameMap.value[playerPos.y][playerPos.x] = oldCell === CELL_TYPES.PLAYER_ON_TARGET ? CELL_TYPES.TARGET : CELL_TYPES.EMPTY

    // 更新玩家新位置
    gameMap.value[newY][newX] = targetCell === CELL_TYPES.BOX_ON_TARGET ? CELL_TYPES.PLAYER_ON_TARGET : CELL_TYPES.PLAYER

    // 更新箱子新位置
    gameMap.value[boxNewY][boxNewX] = boxTargetCell === CELL_TYPES.TARGET ? CELL_TYPES.BOX_ON_TARGET : CELL_TYPES.BOX

    playerPos.x = newX
    playerPos.y = newY
    moves.value++
    pushes.value++

    // 检查胜利
    checkWin()
  }
}

// 检查胜利条件
const checkWin = () => {
  let allBoxesOnTarget = true
  for (let y = 0; y < gameMap.value.length; y++) {
    for (let x = 0; x < gameMap.value[y].length; x++) {
      if (gameMap.value[y][x] === CELL_TYPES.BOX) {
        allBoxesOnTarget = false
        break
      }
    }
    if (!allBoxesOnTarget) break
  }

  if (allBoxesOnTarget) {
    isWin.value = true

    // 如果不是自定义关卡，解锁下一关并保存记录
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
      }
    }
  }
}

// 重新开始当前关卡
const restartLevel = () => {
  initLevel(currentLevel.value)
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
    showTutorial.value = false
    return
  }

  switch (e.key) {
    case 'ArrowUp':
    case 'w':
    case 'W':
      e.preventDefault()
      movePlayer(0, -1)
      break
    case 'ArrowDown':
    case 's':
    case 'S':
      e.preventDefault()
      movePlayer(0, 1)
      break
    case 'ArrowLeft':
    case 'a':
    case 'A':
      e.preventDefault()
      movePlayer(-1, 0)
      break
    case 'ArrowRight':
    case 'd':
    case 'D':
      e.preventDefault()
      movePlayer(1, 0)
      break
    case 'z':
    case 'Z':
      if (e.ctrlKey || e.metaKey) {
        e.preventDefault()
        undo()
      }
      break
    case 'r':
    case 'R':
      restartLevel()
      break
    case 'Escape':
      showLevelSelect.value = true
      break
  }
}

// 触摸控制
let touchStartX = 0
let touchStartY = 0

const handleTouchStart = (e) => {
  touchStartX = e.touches[0].clientX
  touchStartY = e.touches[0].clientY
}

const handleTouchEnd = (e) => {
  if (!isPlaying.value || isWin.value) return

  const touchEndX = e.changedTouches[0].clientX
  const touchEndY = e.changedTouches[0].clientY

  const dx = touchEndX - touchStartX
  const dy = touchEndY - touchStartY

  const minSwipe = 30

  if (Math.abs(dx) > Math.abs(dy) && Math.abs(dx) > minSwipe) {
    movePlayer(dx > 0 ? 1 : -1, 0)
  } else if (Math.abs(dy) > Math.abs(dx) && Math.abs(dy) > minSwipe) {
    movePlayer(0, dy > 0 ? 1 : -1)
  }
}

// 获取单元格类名
const getCellClass = (cell, y, x) => {
  const classes = ['cell']

  const animKey = `${y}-${x}`
  if (animatingCells.value.has(animKey)) {
    classes.push('animating')
  }

  switch (cell) {
    case CELL_TYPES.WALL:
      classes.push('wall')
      break
    case CELL_TYPES.TARGET:
      classes.push('target')
      break
    case CELL_TYPES.BOX:
      classes.push('box')
      break
    case CELL_TYPES.BOX_ON_TARGET:
      classes.push('box', 'on-target')
      break
    case CELL_TYPES.PLAYER:
      classes.push('player')
      break
    case CELL_TYPES.PLAYER_ON_TARGET:
      classes.push('player', 'on-target')
      break
    default:
      classes.push('empty')
  }

  return classes
}

// 获取单元格内容
const getCellContent = (cell) => {
  switch (cell) {
    case CELL_TYPES.WALL:
      return ''
    case CELL_TYPES.TARGET:
      return '○'
    case CELL_TYPES.BOX:
      return '📦'
    case CELL_TYPES.BOX_ON_TARGET:
      return '📦'
    case CELL_TYPES.PLAYER:
    case CELL_TYPES.PLAYER_ON_TARGET:
      return '👷'
    default:
      return ''
  }
}

// 生命周期
onMounted(() => {
  window.addEventListener('keydown', handleKeydown)
  document.addEventListener('touchstart', handleTouchStart, { passive: true })
  document.addEventListener('touchend', handleTouchEnd, { passive: true })
  document.addEventListener('mousedown', () => isMouseDown.value = true)
  document.addEventListener('mouseup', () => isMouseDown.value = false)
  initLevel(0)
  initEditorMap()
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown)
  document.removeEventListener('touchstart', handleTouchStart)
  document.removeEventListener('touchend', handleTouchEnd)
  document.removeEventListener('mousedown', () => isMouseDown.value = true)
  document.removeEventListener('mouseup', () => isMouseDown.value = false)
})
</script>

<style scoped>
/* 组件特定样式 */
</style>
