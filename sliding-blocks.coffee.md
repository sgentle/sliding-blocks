Sliding Blocks
==============

This is a simple sliding blocks puzzle game. If you're reading this and you
haven't played it, you might want to do that first:
https://demos.samgentle.com/sliding-blocks

Helpers
-------

These are some simple helpers for later on. `srcBase` is the parent of this
file, which is where we look for media.

    srcBase = do ->
      src = document.currentScript.src
      a = document.createElement 'a'
      a.href = src
      a.pathname = a.pathname.split('/').slice(0, -1).join('/') + '/'
      a.href

`El` and `SvgEl` are element creation helpers.

    El = (name, attribs={}, content) ->
      el = document.createElement name
      el.setAttribute k, v for k, v of attribs
      el.textContent = content if content
      el

    SvgEl = (name, attribs={}, content) ->
      el = document.createElementNS "http://www.w3.org/2000/svg", name
      el.setAttribute k, v for k, v of attribs
      el.textContent = content if content
      el

Blocks
------

Blocks are represented as an array with a bunch of methods on it.

The Blocks prototype is a big mess. If I were doing this again I would add a
separate `Block` proto and give individual blocks their own methods.

    blocksProto =

`posFor` and `nFor` convert back and forth between array indices and x/y coordinates.

      posFor: (n) ->
        y = n // @rows * @blockHeight
        x = n % @rows * @blockWidth
        {x, y}

      nFor: (x, y) ->
        y // @blockHeight * @cols + x // @blockWidth

`makeBlock` is basically our block constructor. It creates the relevant SVG
elements and adds event listeners to move them around.

      makeBlock: (n, title) ->
        g = SvgEl 'g',
          fill: 'white'

        {x, y} = @posFor n

        rect = SvgEl 'rect',
          x: x
          y: y
          width: @blockWidth
          height: @blockHeight
          stroke: 'black'

        text = SvgEl 'text',
          x: x + @blockWidth/2
          y: y + @blockHeight * (2/3)
          fill: '#444'
          stroke: '#000'
          'font-size': "#{@blockHeight/2}px"
          'font-family': '"verdana"'
          'font-weight': 'bold'
          'text-anchor': 'middle'
          'pointer-events': 'none'

        , title || n + 1

        move = (x, y) =>
          rect.setAttribute 'x', x
          rect.setAttribute 'y', y
          tx = x + @blockWidth/2
          ty = y + @blockHeight * (2/3)
          text.setAttribute 'x', tx
          text.setAttribute 'y', ty
          if text2
            text2.setAttribute 'x', tx
            text2.setAttribute 'y', ty

Here, we handle dragging. The mousedown/touchstart handler adds event
listeners to `window` and removes them afterwards, so we don't have to bother
filtering move events.

        startX = null
        startY = null
        dragState = null
        startdrag = (ev) =>
          return if dragState
          [x, y] = [Number(rect.getAttribute('x')), Number(rect.getAttribute('y'))]

          n = @nFor x, y
          ev.preventDefault()
          console.log "startdrag"
          dragState = 'dragging'
          startX = (ev.pageX ? ev.touches[0].pageX)
          startY = (ev.pageY ? ev.touches[0].pageY)

          window.addEventListener e, drag for e in ['mousemove', 'touchmove']
          window.addEventListener e, stopdrag for e in ['mouseup', 'touchend', 'mouseleave', 'touchcancel']

        stopdrag = (ev) =>
          return unless dragState
          console.log "stopdrag", dragState
          newn = {left: n - 1, right: n + 1, up: n - @cols, down: n + @cols}[dragState]
          dragState = null
          if newn? and @blocks[newn] is null
            @blocks[newn] = g
            @blocks[n] = null
            {x, y} = @posFor newn
            move(x, y)

            n = newn
            solved = @checkSolved()
            if solved and !@targets
              @addTargets()
            @checkIntercat() if @intercat

          else
            move(x, y)
          window.removeEventListener e, drag for e in ['mousemove', 'touchmove']
          window.removeEventListener e, stopdrag for e in ['mouseup', 'touchend', 'mouseleave', 'touchcancel']

        drag = (ev) =>
          diffX = (ev.pageX ? ev.touches[0].pageX) - startX
          diffY = (ev.pageY ? ev.touches[0].pageY) - startY
          dragState = 'dragging'
          if diffX > 0 && @blocks[n + 1] is null && ((n + 1) % @cols isnt 0)
            dragState = 'right' if diffX > @blockWidth/2
            move(x + Math.min(diffX, @blockWidth), y)
          else if diffX < 0 && @blocks[n - 1] is null && (n % @cols isnt 0)
            dragState = 'left' if diffX < -@blockWidth/2
            move(x + Math.max(diffX, -@blockWidth), y)
          else if diffY > 0 && @blocks[n + @cols] is null
            dragState = 'down' if diffY > @blockHeight/2
            move(x, y + Math.min(diffY, @blockHeight))
          else if diffY < 0 && @blocks[n - @cols] is null
            dragState = 'up' if diffY < -@blockHeight/2
            move(x, y + Math.max(diffY, -@blockHeight))
          else
            move(x, y)


        rect.addEventListener 'mousedown', startdrag
        rect.addEventListener 'touchstart', startdrag

        g.appendChild rect
        g.appendChild text

This is a bit of a hack to support distinguishing between "T." and "T", we add
a second text layer and offset it a bit. (If we did things the naive way then
the T. would be positioned differently to the other letters and look weird.

        if title and title[title.length-1] == '.'
          text2 = text.cloneNode()
          text.textContent = title.slice(0, title.length-1)
          text2.textContent = '.'
          setTimeout ->
            text2.setAttribute 'dx', text.getBBox().width / 2 + 'px'
          , 10
          g.appendChild text2 if text2

        g

`add` adds a block to our Blocks. `origBlocks` is kept so we can check whether we have a solution.

      add: (title) ->
        if title?
          el = @makeBlock @blocks.length, title
          @el.appendChild el
          @blocks.push el
          @origBlocks.push el
        else
          @blocks.push null
          @origBlocks.push null

When we're done adding blocks, we can colour them with `addColors`, which does a simple hue cycle.

      addColors: ->
        for el, i in @blocks when el
          hue = 360 * (i / (@blocks.length - 1))
          el.setAttribute 'fill', "hsl(#{hue}, 55%, 70%)"

`drawTarget` is responsible for the little targets underneath once you have solved one version of the puzzle

      drawTarget: (n, done) ->
        oy = @height + 10
        count = @blocks.length
        spacing = @blockWidth / count
        adjWidth = @width + spacing * (count)
        scale = adjWidth / count - 10
        ratio = @height / @width
        if @targets[n]
          @targets[n].remove()

        el = SvgEl 'g'
        @targets[n] = el
        el.appendChild SvgEl 'rect',
          x: n * scale
          y: oy
          width: scale - spacing
          height: ratio * scale
          fill: 'white'

        blockperm = @origBlocks.slice().filter(Boolean)
        blockperm.splice(n, 0, null)
        for block, i in blockperm when block
          blx = i % @rows
          bly = i // @rows
          el.appendChild SvgEl 'rect',
            x: n * scale + (blx * (@blockWidth * (3/4) / count))
            y: oy + (bly * (@blockHeight * (3/4) / count))
            width: (@blockWidth * (3/4)) / count
            height: @blockHeight * (3/4) / count
            fill: if done then block?.getAttribute('fill') else '#ddd'
        @el.appendChild el

`addTargets` just loops through the blocks and calls `drawTarget`. Also makes a bit of extra space in the SVG element.

      addTargets: ->
        @el.setAttribute 'height', @height + @height/4
        @targets = []
        @solved = []
        for n in [0..@blocks.length-1]
          @drawTarget n
          @solved[n] = false
        @checkSolved()

`checkSolved` is how we know when we've found a solution. To account for
variable blank placement, we skip nulls in blocks and origBlocks

      checkSolved: ->
        n = null
        i = 0
        j = 0
        while i < @blocks.length
          if @blocks[i] is null
            n = i++
            continue
          if @origBlocks[j] is null
            j++
            continue

          return false if @blocks[i] != @origBlocks[j]

          i++
          j++

        if @targets
          @drawTarget n, true unless @solved[n]
          @solved[n] = true
          @addIntercat() if @solved.filter(Boolean).length == @blocks.length

        return n

These are Intercat related things

      addIntercat: ->
        return if @intercat
        @intercat = SvgEl 'g',
          fill: '#ddd'
        @intercat.appendChild SvgEl 'text',
          x: @width / 2
          y: @height + @blockHeight / 2 + 10
          'font-size': "#{@blockHeight/4}px"
          'font-family': '"verdana"'
          'font-weight': 'bold'
          'text-anchor': 'middle'
        , "INTERCAT"

        @el.appendChild @intercat

      checkIntercat: ->
        match = "INTERCAT"
        for el, i in @blocks.filter(Boolean)
          # console.log "check", el.querySelector('text').textContent, match[i]
          return false unless match[i] == el.querySelector('text').textContent

        @finishIntercat() unless @finished
        return true

      finishIntercat: ->
        @finished = true
        @intercat.setAttribute 'fill', 'url(#rainbow)'
        audio = new Audio()
        audio.src = srcBase + 'music.mp3' if audio.canPlayType('audio/mp3')
        audio.src = srcBase + 'music.ogg' if audio.canPlayType('audio/ogg')
        audio.currentTime
        audio.play()

        @intercat.style.transformOrigin = "50% 50%"
        @intercat.style.transformBox = "fill-box"
        @intercat.style.animation = '423ms sb-pulse 0s infinite'
        for el, i in @blocks.filter(Boolean)
          el.style.transformOrigin = '50% 50%'
          el.style.transformBox = "fill-box"
          el.style.animation = "#{423 * 8}ms sb-spin #{Math.round(i * 423)}ms infinite"

        for el, i in @targets
          el.style.transformOrigin = '50% 50%'
          el.style.transformBox = "fill-box"
          el.style.animation = "423ms sb-bounce #{Math.round(i/9 * 423)}ms infinite"

        @el.querySelector('rect').setAttribute 'fill', 'url(#rainbow)'

This is the code that shuffles the board at the start.

      shuffle: (times=50) ->
        console.log "shuffle"
        i = 0
        n = @blocks.indexOf(null)
        return if n is -1
        lastswap = null
        swap = (n, m) =>
          console.log("swap", n, m)
          lastswap = m
          [@blocks[n], @blocks[m]] = [@blocks[m], @blocks[n]]
          @move @blocks[n], n
          @move @blocks[m], m
          m
        up = (n) => swap(n, n - @cols) if n > @cols
        down = (n) => swap(n, n + @cols) if n < @blocks.length - @cols
        left = (n) => swap(n, n - 1) if n > 0 and n % @cols != 0
        right = (n) => swap(n, n + 1) if n < @blocks.length - 1 and (n + 1) % @cols != 0
        while i < times or @checkSolved()
          m = null
          dir = Math.floor(Math.random() * 4)
          until (m = [up, left, down, right][dir](n))?
            dir = (dir + 1) % 4
          lastdir = dir
          n = m if m?
          i++

`move` can't cause a block to move by itself, so we have to just reach in and
fiddle with its properties. This is part of the reason it would be better to
have a separate Block object.

      move: (el, n) ->
        return unless el
        {x, y} = @posFor n
        rect = el.querySelector 'rect'
        [text, text2] = el.querySelectorAll 'text'

        rect.setAttribute 'x', x
        rect.setAttribute 'y', y
        tx = x + @blockWidth/2
        ty = y + @blockHeight * (2/3)
        text.setAttribute 'x', tx
        text.setAttribute 'y', ty
        if text2
          text2.setAttribute 'x', tx
          text2.setAttribute 'y', ty

This is the constructor for our puzzle. We set up an SVG element and put some
basic things in there, but we expect addBlocks to be called to populate it.

    Blocks = (width=300, height=300, rows=3, cols=3) ->
      blocks = Object.create blocksProto
      blocks.blocks = []
      blocks.origBlocks = []
      blocks[k] = v for k, v of {width, height, rows, cols}
      blocks.blockWidth = blocks.width / blocks.cols
      blocks.blockHeight = blocks.height / blocks.rows
      blocks.el = SvgEl 'svg',
        xmlns: 'http://www.w3.org/2000/svg'
        width: width
        height: height
      blocks.defs = defs = SvgEl 'defs'
      blocks.el.appendChild defs

      rainbow = SvgEl 'linearGradient',
        id: 'rainbow'
        x1: '0%', y1: '0%', x2: '100%', y2: '100%'
        spreadMethod: 'pad'

      rainbow.appendChild SvgEl 'stop', offset: '0%', 'stop-color': '#ff0000'
      rainbow.appendChild SvgEl 'stop', offset: '17%', 'stop-color': '#ffff00'
      rainbow.appendChild SvgEl 'stop', offset: '34%', 'stop-color': '#00ff00'
      rainbow.appendChild SvgEl 'stop', offset: '50%', 'stop-color': '#00ffff'
      rainbow.appendChild SvgEl 'stop', offset: '66%', 'stop-color': '#0000ff'
      rainbow.appendChild SvgEl 'stop', offset: '82%', 'stop-color': '#ff00ff'
      rainbow.appendChild SvgEl 'stop', offset: '100%', 'stop-color': '#ff0000'

      defs.appendChild rainbow

      blocks.el.appendChild SvgEl 'rect',
        x: 0, y: 0, width: width, height: height
        fill: '#555'

      blocks

This is the glue to connect the `Blocks` instance to a DOM element. We read
the `<s-block>` and `<s-blank>` elements out and then turn them into our own
representation of blocks and blanks.

    bind = (el) ->
      opts = (el.getAttribute Number(x) or undefined for x in ['width', 'height', 'rows', 'cols'])
      blocks = Blocks(opts...)
      for child in el.children
        switch child.nodeName
          when 'S-BLOCK'
            blocks.add child.textContent
          when 'S-BLANK'
            blocks.add null
          else
            console.warn "unknown block type: #{child.nodeName}"
      el.innerHTML = ""

      blocks.addColors()
      blocks.shuffle() if el.getAttribute('shuffle')?
      blocks.withTargets = true if el.getAttribute('targets')?

      el.appendChild blocks.el


    bind el for el in document.querySelectorAll('sliding-blocks')

Some css animations, because yay globals.

    document.body.appendChild El 'style', {}, """
    @keyframes sb-spin {
      0% { transform: rotate(0deg) }
      12.5% { transform: rotate(360deg) }
      100% { transform: rotate(360deg) }
    }
    @keyframes sb-pulse {
      0% { transform: scale(1) }
      50% { transform: scale(1.2) }
      0% { transform: scale(1) }
    }
    @keyframes sb-bounce {
      0% { transform: translateY(0px) }
      33.33% { transform: translateY(-30px) }
      100% { transform: translateY(0px) }
    }
    """