<template>
  <div class="root">
    <div class="game" :style="{height: height}">
      <canvas class="board" id="board"> </canvas>
      <div class="item-container" :style="{height: height}">
        <template v-for="(rowItems, row) in grids">
          <template v-for="(item, column) in rowItems">
            <div class="grid">
              <div v-if="item.chess && item.chess.type === 'sheep'" class="sheep" 
              :class="{'sheep-bg': item.chess.count > 0, 'sheep-hint-from': item.chess.hintFrom, 'sheep-hint-to': item.chess.hintTo, 'sheep-hint-target': item.chess.hintTarget, 'sheep-hint-disapear': item.chess.hintDisapear}"
              @click="handleItemClick(row, column)">
                <div v-if="item.chess.count > 0" class="name">羊<span v-if="item.chess.count > 1" class="count" v-text="item.chess.count + '只'"></span></div>
              </div>
              <div v-else-if="item.chess && item.chess.type === 'tiger'" 
                  class="tiger" :class="{'tiger-bg': item.chess.count > 0, 'tiger-hint-from': item.chess.hintFrom, 'tiger-hint-to': item.chess.hintTo, 'tiger-hint-target': item.chess.hintTarget}"
                  @click="handleItemClick(row, column)">
                <div v-if="item.chess.count > 0" class="name">老虎</div>
              </div>
            </div>

          </template>
        </template>
      </div>
    </div>
    <div class="seprator"></div>
    <div v-if="status === 'sheep_win'" class="win">绵羊胜</div>
    <div v-if="status === 'sheep_lose'" class="lose">老虎胜</div>
    <div class="btn-restart" @click="handleBtnRestartClick">重新开始</div>
  </div>
</template>

<script>

export default {
  name: 'Game',
  props: {
    // msg: String
  },
  data() {
    return {
      height: null,
      grids: null,
      selectedItem: null,
      turn: null,
      status: null,
      role: null,
      sheepLastAutoMove: null
    }
  },
  created() {
    this.grids = this.getGrids()
    this.height = this.getViewWidth() + 'px'
    // this.chessMap = this.getChessMap(this.grids)
  },
  mounted() {
    this.drawBoard(this.grids)
  },
  methods: {
    // 获取棋盘二维数组
    getGrids() {
      const rows = 5
      const columns = rows
      const grids = []
      for(let row = 0; row < rows; row++) {
        const rowItems = []
        grids.push(rowItems)
        for(let column = 0; column < columns; column++) {
          const grid = {row, column}
          rowItems.push(grid)
          if(  (row === 1 && column === 1) 
            || (row === 1 && column === columns - 2) 
            || (row === rows - 2 && column === 1)
            || (row === rows - 2 && column === columns - 2) ) {
            grid.chess = {type: 'sheep', count: 6}
            // grid.chess.hintFrom = true
          } 

          if( (row === 2 && column === 1) || (row === 2 && column === columns - 2) ) {
            // cosole.log('tiger')
            grid.chess = {type: 'tiger', count: 1}
          } 
        }
      }
      return grids
    },
    getViewWidth () {
      return window.outerWidth < 768 ? window.outerWidth : 768
    },
    drawBoard(grids) { 
      const canvas = document.getElementById("board")
      canvas.width = canvas.offsetWidth
      canvas.height = canvas.offsetHeight
      if(canvas.getContext) {
        const ctx = canvas.getContext('2d')
        const boardWidth = this.getViewWidth()
        const boardHeight = boardWidth
        const rows = grids.length
        const columns = rows
        const gridWidth = boardWidth / columns
        const gridHeight = gridWidth
        const startX = parseInt(gridWidth / 2)
        const startY = startX
        ctx.beginPath()
        // 绘制5行
        for (let i = 0; i < rows; i++) { 
          ctx.moveTo(startX, startY + i * gridHeight)
          ctx.lineTo(boardWidth - startX, startY + i * gridHeight)
          ctx.stroke()
        }
        // 绘制5列
        for (let i = 0; i < columns; i++) {
          ctx.moveTo(startX + i * gridWidth, startY)
          ctx.lineTo(startX + i * gridWidth, boardHeight - startY)
          ctx.stroke()
        }
        // 所以斜线的起始坐标
        const slashs = [{from: {row: 0, column: 0}, to: {row: rows - 1, column: columns - 1}},
                        {from: {row: 0, column: columns - 1}, to: {row: rows - 1, column: 0}},
                        {from: {row: 2, column: 0}, to: {row: 0, column: 2}},
                        {from: {row: 0, column: 2}, to: {row: 2, column: columns - 1}},
                        {from: {row: 2, column: 0}, to: {row: rows - 1, column: 2}},
                        {from: {row: 2, column: columns - 1}, to: {row: rows - 1, column: 2}}
                        ]
        // 绘制斜线
        slashs.forEach(item => {
          ctx.moveTo(startX + item.from.column * gridWidth, startY + item.from.row * gridHeight)
          ctx.lineTo(startX + item.to.column * gridWidth, startY + item.to.row * gridHeight)
          ctx.stroke()
        })                           
      }
    },
    // 点击棋子后，生成棋子的目标占位棋子，返回修改后的棋盘数组
    getGridsWithTarget(grids, row, column) {
      const chess = grids[row][column].chess
      const targets = this.getTargetsWith(grids, chess.type, row, column)
      targets.forEach(target => {
        grids[target.row][target.column].chess = {type: chess.type, count: 0, hintTarget: true}
      })
      return grids;
    },
    getGridsWithMove(srcGrids, fromItem, row, column) {
      const grids = []
      srcGrids.forEach(rowItems => {
        const copyRowItems = []
        grids.push(copyRowItems)
        rowItems.forEach(item => {
          const chess = item.chess ? Object.assign({}, item.chess) : undefined
          copyRowItems.push({row: item.row, column: item.column, chess})
        })
      })
      // 清除所有移动目标的提示
      grids.forEach(rowItems => {
        rowItems.forEach(item => {
          item.chess && item.chess.hintTarget && (item.chess = undefined)
        })
      })
      // 原来位置棋子数量减一
      grids[fromItem.row][fromItem.column].chess.count --
      // 设置移动起始标志hintFrom
      grids[fromItem.row][fromItem.column].chess.hintFrom = true
      // 目标位置棋子创建
      grids[row][column].chess = {type: fromItem.chess.type, count: 1, hintTo: true}
      // 如果移动两步，中间的羊消掉
      if(Math.abs(fromItem.row - row) === 2 || Math.abs(fromItem.column - column) === 2) {
        const chess = grids[fromItem.row + (row - fromItem.row) / 2][fromItem.column + (column - fromItem.column) / 2].chess
        chess.count --;
        chess.hintDisapear = true;
      }
      // console.log('grids', grids)
      return grids
    },
    // 
    getTargetsWith(grids, type, row, column) {
      const canMoveSlash = ((row % 2 === 1) && (column % 2 === 1)) || ((row % 2 === 0) && (column % 2 === 0)) // 奇数行和奇数列或者偶数行和偶数列对应的格子才可走斜线
      const maxStepLength = type === 'sheep' ? 1 : 2
      const targetsMap = {}
      for (let stepLength = 1; stepLength <= maxStepLength; stepLength++) {
        const targets = [ {row: row - stepLength, column}, // up
                        {row: row + stepLength, column}, // down
                        {row, column: column - stepLength}, // left
                        {row, column: column + stepLength}, // right
                      ]
        if(canMoveSlash) {
          targets.push({row: row - stepLength, column: column - stepLength}) // up left
          targets.push({row: row - stepLength, column: column + stepLength}) // up right
          targets.push({row: row + stepLength, column: column - stepLength}) // down left
          targets.push({row: row + stepLength, column: column + stepLength}) // down right
        }  
        targetsMap[stepLength] = targets
      }
      const oneStepLengthTargets = targetsMap[1] // 步长为1的目标
      const results = []
      if(type !== 'sheep') {
        const targets = targetsMap[2] // 步长为2的目标
        const twoStepLengthResults = targets.filter((target, index) => {
          if(this.isOut(grids, target.row, target.column)) return false // 出界
          if(!this.isEmpty(grids, target.row, target.column)) return false // 目标已经有棋子
          if(this.isEmpty(grids, oneStepLengthTargets[index].row, oneStepLengthTargets[index].column)) return false // 起始位置和目标之间没有棋子
          const oneStepLengthChess= grids[oneStepLengthTargets[index].row][oneStepLengthTargets[index].column].chess
          if(oneStepLengthChess.type !== 'sheep') return false // 起始位置和目标之间不是羊
          return true
        })
        results.push(...twoStepLengthResults)
      }
      const oneStepLengthResults = oneStepLengthTargets.filter((target) => {
        if(this.isOut(grids, target.row, target.column)) return false // 出界
        if(!this.isEmpty(grids, target.row, target.column)) return false // 目标已经有棋子
        return true
      })
      results.push(...oneStepLengthResults)
      // console.log('oneStepLengthTargets', oneStepLengthTargets)
      // console.log('results', results)
      return results
    },
    isOut(grids, row, column) {
      const rows = grids.length
      const columns = rows
      return (row < 0 || row >= rows || column < 0 || column >= columns)  // 出界了
    },
    isEmpty(grids, row, column) {
      if(!grids[row][column].chess) return true
      if(grids[row][column].chess.count === 0) return true;
      return false
    },
    sheepAutoMove() {
      const copyGrids = [];
      this.grids.forEach(rowItems => {
        const copyRowItems = []
        copyGrids.push(copyRowItems)
        rowItems.forEach(item => {
          const chess = item.chess ? Object.assign({}, item.chess): undefined
          const copyItem = {row: item.row, column: item.column, chess}
          copyRowItems.push(copyItem)
        })
      })
      const sheepItems = []
      copyGrids.forEach(rowItems => {
        rowItems.forEach(item => {
          if(item.chess && item.chess.type === 'sheep' && item.chess.count > 0) {
            sheepItems.push(item)
          }
        })
      })
      const tigerItemsFromGids = (grids) => {
        const tigerItems = []
        grids.forEach(rowItems => {
          rowItems.forEach(item => {
            if(item.chess && item.chess.type === 'tiger' && item.chess.count > 0) {
              tigerItems.push(item)
            }
          })
        })
        return tigerItems
      }
      const scoreFromGrids = (grids) => {
        const tigerItems = tigerItemsFromGids(grids)
        let score = 0
        tigerItems.forEach(tigerItem => {
          const tigerTargets = this.getTargetsWith(grids, tigerItem.chess.type, tigerItem.row, tigerItem.column)
          tigerTargets.forEach(tigerTarget => {
            if(Math.abs(tigerTarget.row - tigerItem.row) === 2 || Math.abs(tigerTarget.column - tigerItem.column) === 2) {
              const count = grids[tigerItem.row + (tigerTarget.row - tigerItem.row) / 2][tigerItem.column + (tigerTarget.column - tigerItem.column) / 2].count
              if(count > 1) {
                score -= 10 // 子多于一个，被吃掉一个，影响比较小
                score += count // 子越多影响越小
              } else {
                score -= 30 // 只有一个子，被吃掉，影响比较大
              }
            } else {
              score -= 2
            }
            if((tigerTarget.row % 2 === 0 && tigerTarget.column % 2 === 0) || (tigerTarget.row % 2 === 1 && tigerTarget.column % 2 === 1)) {
              // 就是可以走斜线的情况下，减的分多些
               score -= 1
            }
            tigerTarget
          })
        })
        return score
      }
      const itemAndTargets = []
      // let maxScore = Number.MIN_SAFE_INTEGER
      for (let i  = 0; i < sheepItems.length; i++) {
        const item = sheepItems[i]
        const targets = this.getTargetsWith(copyGrids, item.chess.type, item.row, item.column)
        for (let j = 0; j < targets.length; j++) {
          const target = targets[j]
          target.score = 0 // 用于评比最优目标
          let movedGrids = this.getGridsWithMove(copyGrids, item, target.row, target.column)
          let tigerItems = tigerItemsFromGids(movedGrids)
          target.score += scoreFromGrids(movedGrids, tigerItems)
          // tigerItems.forEach(tigerItem => {
          //   const tigerTargets = this.getTargetsWith(movedGrids, tigerItem.chess.type, tigerItem.row, tigerItem.column)
          //   tigerTargets.forEach(tigerTarget => {
          //     const tigetMovedGrids = this.getGridsWithMove(movedGrids, tigerItem, tigerTarget.row, tigerTarget.column)
          //     target.score += scoreFromGrids(tigetMovedGrids)
          //   })
          // })
          // console.log('target score', target.score)
          // if(target.score > maxScore) {
            // maxScore = target.score
            itemAndTargets.push({
              item,
              target
            })
          // }
        }
      }

      let bestItemAndTargets = itemAndTargets.filter(itemAndTarget => {
        if(
          itemAndTargets.length > 0
        && this.sheepLastAutoMove 
        && itemAndTarget.item.row === this.sheepLastAutoMove.target.row
        && itemAndTarget.item.column === this.sheepLastAutoMove.target.column
        && itemAndTarget.target.row === this.sheepLastAutoMove.item.row
        && itemAndTarget.target.column === this.sheepLastAutoMove.item.column) { // 如果有备选，不能移动回原来的位置
          return false
        }
        return true
      })
      bestItemAndTargets = bestItemAndTargets.sort((itemAndTarget1, itemAndTarget2) => itemAndTarget2.target.score - itemAndTarget1.target.score)
      // console.log('bestItemAndTargets', bestItemAndTargets)
      if(bestItemAndTargets.length > 0) {
        const bestItemAndTarget = bestItemAndTargets[0]
        this.handleItemClick(bestItemAndTarget.item.row, bestItemAndTarget.item.column, true)
        this.handleItemClick(bestItemAndTarget.target.row, bestItemAndTarget.target.column, true)
        this.sheepLastAutoMove = bestItemAndTarget
      }
    },
    tigerAutoMove() {
      const tigerItems = []
      this.grids.forEach(rowItems => {
        rowItems.forEach(item => {
          if(item.chess && item.chess.type === 'tiger' && item.chess.count > 0) {
            tigerItems.push(item)
          }
        })
      })
      let bestItem, bestTarget
      let maxScore = -1
      for (let i  = 0; i < tigerItems.length; i++) {
        const item = tigerItems[i]
        const targets = this.getTargetsWith(this.grids, item.chess.type, item.row, item.column)
        for (let j = 0; j < targets.length; j++) {
          const target = targets[j]
          target.score = 0 // 用于评比最优目标
          if(Math.abs(item.row - target.row) === 2 || Math.abs(item.column - target.column) === 2) {
            target.score += 10 // 能吃羊加10分
          } 
          const nextTargets = this.getTargetsWith(this.grids, item.chess.type, item.row, item.column)
          target.score += nextTargets.length // 下一步还能走的目标作为分值
          if(target.score > maxScore) {
            maxScore = target.score
            bestItem = item
            bestTarget = target
          }
        }
      }
      if(bestItem) {
        this.handleItemClick(bestItem.row, bestItem.column, true)
        this.handleItemClick(bestTarget.row, bestTarget.column, true)
      }
    },
    handleItemClick(row, column, isAuto) {
      // console.log(row, column)
      // 游戏结束
      if(this.status) {
        return
      }
      if(!isAuto) {
        if(this.role && this.role !== this.turn)
        return;
      }
      const item = this.grids[row][column]
      const moveChess = () => {
        const tigerItems = []
        // 移动棋子，老虎的话，可能吃掉羊, 去掉所有的可移动到的目标格子提示，棋子起始的位置加上起始提示标志，棋子落下的位置加上落下的提示标志， 被吃掉的棋子加上棋子消失的标志
        this.grids = this.getGridsWithMove(this.grids, this.selectedItem, row, column)
        this.grids = this.grids.concat() // 触发页面刷新
        // 判断羊的数量少于6个羊就输了
        const sheepCount = this.grids.reduce((total, rowItems) => {
          // console.log('total', total)
          return total + rowItems.reduce((rowTotal, item) => {
            if(item.chess && item.chess.type === 'tiger' && item.chess.count > 0) {
              tigerItems.push(item)
            }
            return rowTotal + ((item.chess && item.chess.type === 'sheep') ? item.chess.count : 0)
          }, 0)
        }, 0)
        // console.log('sheep count', sheepCount)
        if(sheepCount < 6) {
          this.status = 'sheep_lose'
          alert('羊太少了，老虎胜利，请点击重新开始')
        }
        // 判断老虎是否还能走动，不能走，老虎就输了
        let targetCount = 0
        tigerItems.forEach(item => {
          const targets = this.getTargetsWith(this.grids, item.chess.type, item.row, item.column)
          // console.log('tiger target', targets)
          targetCount += targets.length
        })
        // console.log('targetCount', targetCount)
        if(targetCount === 0) {
          this.status = 'sheep_win'
          alert('把老虎困住了，羊胜利，请点击重新开始')
        }
        this.turn = this.selectedItem.chess.type === 'sheep' ? 'tiger' : 'sheep'
        if(!this.role) {
          this.role = this.selectedItem.chess.type
        }
        this.selectedItem = null
        if(!isAuto) {
          if(this.turn === 'tiger') {
            this.tigerAutoMove()
          } else {
            this.sheepAutoMove()
          }
        }
      }

      const selectChess = () => {
        // 清除之前移动完留下的提示，也就是hintFrom和hintTo和hintDisapear都修改成undefined
        this.grids.forEach(rowItems =>  {
          rowItems.forEach(item => {
            item.chess && item.chess.hintFrom && (item.chess.hintFrom = undefined)
            item.chess && item.chess.hintTo && (item.chess.hintTo = undefined)
            item.chess && item.chess.hintTarget && (item.chess.hintTarget = undefined)
            item.chess && item.chess.hintDisapear && (item.chess.hintDisapear = undefined)
          })
        })
        // console.log('grids', this.grids)
        // 生成目标占位棋子，用来显示可移动到的目标格子
        this.grids = this.getGridsWithTarget(this.grids, row, column)
        this.grids = this.grids.concat() // 触发页面刷新
        // 记录下选中的棋子
        this.selectedItem = item
      }

      // 点击到一个棋子，并且轮到当前棋子角色移动了(this.turn为null时，可以移动任意一个角色)
      if(!this.isEmpty(this.grids, row, column) && (!this.turn || this.turn === item.chess.type)) {
        selectChess()
        return
      } 
      // 没有点击到棋子上，并且已经选中一个棋子
      if(this.selectedItem) {
        // 判断点中的是否是可移动到的目标格子，不是直接return
        if(!item.chess.hintTarget) {
          return
        }
        moveChess()
      }

    },
    handleBtnRestartClick() {
      this.grids = this.getGrids()
      this.grids = this.grids.concat() // 触发刷新
      this.turn = null
      this.status = null
      this.role = null
    },
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
.root {
  position: relative;
}
.game {
  position: relative;
  left: 0;
  right: 0;
  top: 0;
}
.board {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
  background: red;
}
.item-container {
  position: absolute;
  left: 0;
  right: 0;
  top: 0;
  display: flex;
  flex-wrap: wrap;
}
.grid {
  width: 20%;
  height: 20%;
  display: flex;
  justify-content: center;
  align-items: center;
  /* border:green solid 1px; */
}
.sheep {
  width: 40px;
  height: 40px;
  display: flex;
  justify-content: center;
  align-items: center;
  color: white;
  border-radius: 50%;
  font-size: 14px;
}
.sheep-bg {
  background: green;
}
.count {
  font-size: 10px;
}
.tiger {
  width: 45px;
  height: 45px;
  display: flex;
  justify-content: center;
  align-items: center;
  color: white;
  border-radius: 50%;
  font-size: 16px;
}
.tiger-bg {
  background: blue;
}
.sheep-hint-from {
  border: white dotted 3px;
}
.sheep-hint-to {
  border: white dotted 3px;
}
.sheep-hint-target {
  border: white dotted 3px;
}
.sheep-hint-disapear {
  border: white dotted 3px;
}
.tiger-hint-from {
  border: black dotted 3px;
}
.tiger-hint-to {
  border: black dotted 3px;
}
.tiger-hint-target {
  border: black dotted 3px;
}
.win {
  font-size: 20px;
  color: green;
  line-height: 200%;
}
.lose {
  font-size: 20px;
  color: red;
  line-height: 200%;
}
.btn-restart {
  font-size: 20px;
  color: green;
  border: green solid 1px;
  line-height: 200%;
}
.seprator {
  height: 10px;
}
</style>
