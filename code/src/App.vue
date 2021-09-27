<template>
  <div id="app">
    <canvas id="canvas" :width="cw" :height="ch" style="border: 1px #ccc solid;"></canvas>
    <img src="" id="img">
    <br>
    <input type="file" id='file'><br>
    <button id="clip">clip</button>
    <button id="reclip">reclip</button>
    <a id="save" href=""><button>save</button></a>
    <ul>
      <li>固定裁剪框比例<input type="checkbox" v-model="w_h"></li>
      <li>固定裁剪框大小<input type="checkbox" v-model="lockCtl"></li>
      <li>固定被剪裁图像大小<input type="checkbox" v-model="lockPic"></li>
    </ul>
    <div id="wh">
      <span v-html="parseInt(borderw)"></span>,
      <span v-html="parseInt(borderh)"></span>
    </div>
  </div>
</template>

<style lang="less">
#wh {
  position: absolute;
  height: 20px;
  min-width: 65px;
  padding: 0 3px;
  line-height: 20px;
  text-align: center;
  color: #e2565d;
  border: 2px solid #e2565d;
}
#img {
  margin: 0 5px;
  padding: 10px;
  border: 1px solid #ccc;
}
</style>

<script>
export default {
  data: function () {
    return {
      imgURL: require('@/assets/images/test.png'),
      coord: [
        // 裁剪框四角初始坐标
        [100, 100],
        [400, 100],
        [400, 400],
        [100, 400]
      ],
      w: 8,
      step: 1.1,
      strokeStyle: 'rgb(250,55,50)',
      fillStyle: 'rgba(250,255,255,0.3)',
      cursor: ['nw-resize', 'n-resize', 'ne-resize', 'e-resize', 'se-resize', 's-resize', 'sw-resize', 'w-resize', 'move', 'default'],
      lineDash: [5, 10],
      currentCp: 9,
      down: false,
      movep: false,
      cliped: false,
      lockCtl: false,
      w_h: false,
      lockPic: false,
      dbltouch: false,
      ox: 10,
      oy: 10,
      cw: 300,
      ch: 300,
      pw: 480,
      ph: 480,
      borderw: 300,
      borderh: 300
    }
  },
  methods: {
    init: function () {
      // 初始化
      this.c = document.querySelector('#canvas')
      this.ctx = this.c.getContext('2d')
      this.ctx.strokeStyle = this.strokeStyle
      this.ctx.fillStyle = this.fillStyle
      this.image = new Image()
      this.image.src = this.imgURL
      this.img = document.querySelector('#img')
      this.wh = document.querySelector('#wh')
      this.clip = document.querySelector('#clip')
      this.reclip = document.querySelector('#reclip')
      this.file = document.querySelector('#file')
      this.temp = document.createElement('canvas')
      this.save = document.querySelector('#save')
      // onload后执行
      this.image.onload = () => {
        this.pw = this.image.width
        this.ph = this.image.height
        while (this.pw >= this.cw - 5 || this.ph >= this.ch - 5) {
          this.pw /= this.step
          this.ph /= this.step
        }
        this.ox = (this.cw - this.pw) / 2
        this.oy = (this.ch - this.ph) / 2
        this.coord = [
          [this.ox, this.oy],
          [this.ox + this.pw, this.oy],
          [this.ox + this.pw, this.oy + this.ph],
          [this.ox, this.oy + this.ph]
        ]
        this.wh.style.left = this.c.offsetLeft + this.coord[2][0] + 8 + 'px'
        this.wh.style.top = this.c.offsetTop + this.coord[2][1] - 27 + 'px'
        this.ctx.clearRect(0, 0, this.cw, this.ch)
        this.ctx.drawImage(this.image, this.ox, this.oy, this.pw, this.ph)
        this.borderw = this.pw
        this.borderh = this.ph
        this.drawBorder()
        this.drawControl()
      }
    },
    calc: function (coord) {
      // 计算控制器位置
      const cps = []
      for (let i = 0; i < coord.length; i++) {
        cps.push({
          index: i * 2,
          cursor: this.cursor[i * 2],
          coord: [coord[i][0], coord[i][1]]
        })
      }
      for (let i = 0; i < coord.length; i++) {
        const nxt = i === 3 ? 0 : i + 1
        cps.push({
          index: i * 2 + 1,
          cursor: this.cursor[i * 2 + 1],
          coord: [(coord[i][0] + coord[nxt][0]) / 2, (coord[i][1] + coord[nxt][1]) / 2]
        })
      }
      cps.push({
        index: 8,
        cursor: this.cursor[8],
        coord: [(coord[0][0] + coord[2][0]) / 2, (coord[0][1] + coord[2][1]) / 2]
      })
      return cps
    },
    drawBorder: function () {
      this.ctx.save()
      this.ctx.setLineDash([5, 5])
      this.ctx.beginPath()
      this.ctx.moveTo(this.coord[0][0], this.coord[0][1])
      for (let i = 0; i < this.coord.length; i++) {
        if (i < 3) {
          this.ctx.lineTo(this.coord[i + 1][0], this.coord[i + 1][1])
        } else {
          this.ctx.lineTo(this.coord[0][0], this.coord[0][1])
        }
      }
      this.ctx.stroke()
      this.ctx.fill()
      this.ctx.restore()
    },
    drawControl: function () {
      this.cps = this.calc(this.coord)
      const c = this.ctx
      for (let i = 0; i < this.cps.length; i++) {
        const x = this.cps[i].coord[0]
        const y = this.cps[i].coord[1]
        const w = this.w
        c.beginPath()
        c.lineTo(x - w, y + w)
        c.lineTo(x + w, y + w)
        c.lineTo(x + w, y - w)
        c.lineTo(x - w, y - w)
        c.closePath()
        c.stroke()
      }
    },
    getCurrentCp: function (x, y) {
      const cps = this.cps
      const w = this.w
      for (let i = 0; i < cps.length; i++) {
        const px = cps[i].coord[0]
        const py = cps[i].coord[1]
        if (x >= px - w && x <= px + w && y >= py - w && y <= py + w) {
          return cps[i].index
        }
      }
      return 9
    },
    moveControl: function (x, y) {
      const mx = x - this.dx
      const my = y - this.dy
      this.ctx.clearRect(0, 0, this.ch, this.cw)
      this.ctx.drawImage(this.image, this.ox, this.oy, this.pw, this.ph)
      for (let i = 0; i < this.coord.length; i++) {
        this.coord[i][0] += mx
        this.coord[i][1] += my
      }
      this.drawBorder()
      this.drawControl()
      this.dx = x
      this.dy = y
      this.wh.style.left = this.c.offsetLeft + this.coord[2][0] + 8 + 'px'
      this.wh.style.top = this.c.offsetTop + this.coord[2][1] - 27 + 'px'
    },
    resizeControl: function (x, y) {
      if (this.lockCtl) {
        return
      }
      let mx = x - this.dx
      let my = y - this.dy
      this.ctx.clearRect(0, 0, this.ch, this.cw)
      this.ctx.drawImage(this.image, this.ox, this.oy, this.pw, this.ph)
      if (this.currentCp % 2 === 0) {
        const n1 = this.currentCp / 2
        const n2 = n1 === 3 ? 0 : n1 + 1
        const n3 = n1 === 0 ? 3 : n1 - 1
        if (n1 === 0 || n1 === 2) {
          if (this.w_h) {
            const w = this.coord[2][0] - this.coord[0][0]
            const h = this.coord[2][1] - this.coord[0][1]
            if (Math.abs(mx) < Math.abs(my)) {
              mx = (my * w) / h
            } else {
              my = (mx * h) / w
            }
          }
          this.coord[n1][0] += mx
          this.coord[n1][1] += my
          this.coord[n2][1] += my
          this.coord[n3][0] += mx
        } else {
          if (this.w_h) {
            const w = this.coord[2][0] - this.coord[0][0]
            const h = this.coord[2][1] - this.coord[0][1]
            if (Math.abs(mx) < Math.abs(my)) {
              mx = (-my * w) / h
            } else {
              my = (-mx * h) / w
            }
          }
          this.coord[n1][0] += mx
          this.coord[n1][1] += my
          this.coord[n2][0] += mx
          this.coord[n3][1] += my
        }
      } else {
        if (this.w_h === false) {
          const n1 = (this.currentCp - 1) / 2
          const n2 = n1 === 3 ? 0 : n1 + 1
          if (this.coord[n1][0] === this.coord[n2][0]) {
            this.coord[n1][0] += mx
            this.coord[n2][0] += mx
          } else {
            this.coord[n1][1] += my
            this.coord[n2][1] += my
          }
        }
      }
      this.drawBorder()
      this.drawControl()
      this.dx = x
      this.dy = y
      this.borderw = this.coord[2][0] - this.coord[0][0]
      this.borderh = this.coord[2][1] - this.coord[0][1]
      this.wh.style.left = this.c.offsetLeft + this.coord[2][0] + 8 + 'px'
      this.wh.style.top = this.c.offsetTop + this.coord[2][1] - 27 + 'px'
    },
    resizePic: function (n, scale) {
      if (this.lockPic) {
        return
      }
      const oldCenter = [this.ox + this.pw / 2, this.oy + this.ph / 2]
      if (this.dbltouch && scale) {
        this.pw *= scale
        this.ph *= scale
      } else {
        if (n) {
          this.pw *= this.step
          this.ph *= this.step
        } else {
          this.pw /= this.step
          this.ph /= this.step
        }
      }
      const newCenter = [this.ox + this.pw / 2, this.oy + this.ph / 2]
      this.movePic(newCenter[0] - oldCenter[0], newCenter[1] - oldCenter[1], 0)
    },
    movePic: function (x, y, n) {
      if (n) {
        const mx = x - this.dx
        const my = y - this.dy
        this.ox += mx
        this.oy += my
      } else {
        this.ox -= x
        this.oy -= y
      }
      this.ctx.clearRect(0, 0, this.ch, this.cw)
      this.ctx.drawImage(this.image, this.ox, this.oy, this.pw, this.ph)
      this.drawBorder()
      this.drawControl()
      this.dx = x
      this.dy = y
    },
    getDistance: function (a, b) {
      const x = b.clientX - a.clientX
      const y = b.clientY - a.clientY
      return Math.sqrt(x * x + y * y)
    }
  },
  mounted: function () {
    document.onselectstart = () => {
      return false
    }
    this.init()
    this.c.addEventListener('mousemove', e => {
      const x = e.clientX - this.c.offsetLeft
      const y = e.clientY - this.c.offsetTop
      if (this.down === false && this.movep === false) {
        this.currentCp = this.getCurrentCp(x, y)
        this.c.style.cursor = this.cursor[this.currentCp]
      } else if (this.down === true) {
        if (this.currentCp === 8) {
          this.moveControl(x, y)
        } else if (this.currentCp >= 0 && this.currentCp <= 7) {
          this.resizeControl(x, y)
        }
      } else if (this.movep === true) {
        this.movePic(x, y, 1)
      }
    })
    this.c.addEventListener('touchmove', e => {
      e.preventDefault()
      if (this.dbltouch) {
        this.end = e.touches
        var scale = this.getDistance(this.end[0], this.end[1]) / this.getDistance(this.start[0], this.start[1])
        this.resizePic(1, scale)
        this.start = e.touches
      } else {
        const x = e.targetTouches[0].clientX - this.c.offsetLeft
        const y = e.targetTouches[0].clientY - this.c.offsetTop
        if (this.down === false && this.movep === false) {
          this.currentCp = this.getCurrentCp(x, y)
          this.c.style.cursor = this.cursor[this.currentCp]
        } else if (this.down === true) {
          if (this.currentCp === 8) {
            this.moveControl(x, y)
          } else if (this.currentCp >= 0 && this.currentCp <= 7) {
            this.resizeControl(x, y)
          }
        } else if (this.movep === true) {
          this.movePic(x, y, 1)
        }
      }
    })
    this.c.addEventListener('mousedown', e => {
      if (this.cliped) {
        return
      }
      const x = e.clientX - this.c.offsetLeft
      const y = e.clientY - this.c.offsetTop
      this.currentCp = this.getCurrentCp(x, y)
      if (this.currentCp < 9) {
        this.down = true
        this.movep = false
      } else if (this.currentCp === 9) {
        this.movep = true
        this.down = false
      }
      this.dx = e.clientX - this.c.offsetLeft
      this.dy = e.clientY - this.c.offsetTop
    })
    this.c.addEventListener('touchstart', e => {
      if (this.cliped) {
        return
      }
      if (e.touches.length >= 2) {
        this.dbltouch = true
        this.start = e.touches
      }
      const x = e.targetTouches[0].clientX - this.c.offsetLeft
      const y = e.targetTouches[0].clientY - this.c.offsetTop
      this.currentCp = this.getCurrentCp(x, y)
      if (this.currentCp < 9) {
        this.down = true
        this.movep = false
      } else if (this.currentCp === 9) {
        this.movep = true
        this.down = false
      }
      this.dx = e.targetTouches[0].clientX - this.c.offsetLeft
      this.dy = e.targetTouches[0].clientY - this.c.offsetTop
    })
    this.c.addEventListener('mouseup', e => {
      if (this.down) {
        this.down = false
      }
      if (this.movep) {
        this.movep = false
      }
    })
    this.c.addEventListener('touchend', e => {
      if (this.down) {
        this.down = false
      }
      if (this.movep) {
        this.movep = false
      }
      if (this.dbltouch) {
        this.dbltouch = false
      }
    })
    this.c.addEventListener('mousewheel', e => {
      e.preventDefault()
      if (this.cliped) {
        return
      }
      if (e.wheelDelta > 0) {
        this.resizePic(1, 0)
      } else {
        this.resizePic(0, 0)
      }
    })
    this.clip.addEventListener('click', () => {
      if (this.cliped) {
        return
      }
      this.cliped = true
      const width = this.coord[2][0] - this.coord[0][0]
      const height = this.coord[2][1] - this.coord[0][1]
      this.ctx.save()
      this.ctx.clearRect(0, 0, this.cw, this.ch)
      this.ctx.beginPath()
      this.ctx.rect(this.coord[0][0], this.coord[0][1], width, height)
      this.ctx.closePath()
      this.ctx.clip()
      this.ctx.drawImage(this.image, this.ox, this.oy, this.pw, this.ph)
      var data = this.ctx.getImageData(this.coord[0][0], this.coord[0][1], this.borderw, this.borderh)
      this.temp.width = this.borderw
      this.temp.height = this.borderh
      this.tempCtx = this.temp.getContext('2d')
      this.tempCtx.putImageData(data, 0, 0, 0, 0, this.borderw, this.borderh)
      this.img.src = this.temp.toDataURL('image/png')
      this.save.href = this.img.src
      this.save.download = 'pic-edit' + new Date().getTime()
    })
    this.reclip.addEventListener('click', () => {
      this.cliped = false
      this.img.src = ''
      this.ctx.restore()
      this.ctx.drawImage(this.image, this.ox, this.oy, this.pw, this.ph)
      this.drawBorder()
      this.drawControl()
    })
    this.file.addEventListener('change', e => {
      this.reclip.click()
      const dataURL = window.URL.createObjectURL(this.file.files[0])
      this.image.src = dataURL
    })
  }
}
</script>
