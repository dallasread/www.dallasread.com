---
layout: post
title: The surprising effects of random mutation
description: An exploration in neural nets and how they evolve.
categories: progress
---

<div class="overflow-auto bg-gray-800 bg-gradient-to-r from-gray-900 to-sky-900 text-green-400 text-center p-4 lg:p-16 mb-10">
  <code id="neurenv" class="inline-block whitespace-pre text-left font-monospace"></code>
</div>

The source code is embedded on this page. Have a look in the developer console for more! 👻

<!--
- starting location
- starting attributes
- luck
- learning
- time
- no goal, only longevity/replication
- mutation

- color
- replicate every X time
- die every x time

Sensor:SEE_NORTH -> 0.5 -> Neuron:INNER_0
Neuron:INNER_0 -> 0.4 -> Action:MOVE_NORTH

Imagine the case where the top/bottom gates were death
Only the E/W will survive
All mutations towards N/S will die immediately

Sensors are zeroed between generations
-->

<script type="text/javascript">
  const CLEAR_SCREEN = '\033[2J'

  const shuffle = (arr, n) => arr.sort(() => 0.5 - Math.random()).slice(0, n)
  const rand = (min, max) => Math.random() * (max - min) + min
  const randInt = (min, max) => parseInt(rand(min, max))
  const createArray = (length, func) => Array.apply(null, { length }).map(func)

  const DIRECTION_NORTH = 'NORTH'
  const DIRECTION_SOUTH = 'SOUTH'
  const DIRECTION_EAST = 'EAST'
  const DIRECTION_WEST = 'WEST'

  class Neuron {
    constructor(name, value = 0) {
      this.name = name
      this.value = value
    }

    nameWithType() {
      return `${this.constructor.name}:${this.name}`
    }

    getValue() {
      return this.value
    }

    setValue(value) {
      this.value = value

      return this.value
    }

    addValue(value) {
      this.setValue((this.getValue() || 0) + value)
    }

    zeroValue() {
      this.value = 0
    }
  }

  Neuron.fromSource = (source, value) => {
    return new Neuron(source.name, source.value)
  }

  class Sensor extends Neuron {
    constructor(name, func) {
      super(name, 0)

      this.func = func
    }

    setValue(creature) {
      if (typeof this.func === 'function') {
        this.value = this.func.call(this, creature)
      }

      return this.getValue()
    }
  }

  Sensor.fromSource = (source) => {
    return new Sensor(source.name, source.func)
  }

  class Action extends Neuron {
    constructor(name, func) {
      super(name, 0)

      this.func = func
    }

    act(creature) {
      this.func.call(this, creature)
    }
  }

  Action.fromSource = (source) => {
    return new Action(source.name, source.func)
  }

  class Synapse {
    constructor(from, to, weight) {
      this.from = from
      this.to = to
      this.weight = weight
    }
  }

  Synapse.fromSourceWithMutation = (source, neurons, mutation) => {
    const from = neurons.find((n) => n.name === source.from.name)
    const to = neurons.find((n) => n.name === source.to.name)

    return new Synapse(from, to, source.weight + (source.weight * (mutation * (Math.random() > 0.5 ? 1 : -1))))
  }

  class Brain {
    constructor(sensors, urges, innerNeurons, synapses) {
      this.sensors = sensors
      this.innerNeurons = innerNeurons
      this.urges = urges
      this.synapses = synapses
    }

    fire(creature) {
      const weightedUrges = createArray(this.urges.length, () => 0)

      this.sensors
      //   .concat(this.innerNeurons)
        .concat(this.urges)
        .forEach((n) => n.zeroValue())

      this.sensors
        .forEach((s) => s.setValue(creature))

      // for multiple layers, this could be looping the layers,
      // then looping the synapses to it, from it
      this.synapses.forEach((synapse) => {
        synapse.to.addValue(synapse.from.getValue() * synapse.weight)
      })

      this.urges.forEach((urge) => {
        urge.setValue(Math.tanh(urge.value))
      })

      return this.urges
       // TODO: filter by some activation function?
        .filter((u) => u.getValue() > 0.5)
    }

    log() {
      console.log(
        this.synapses.map((s) => {
          return [s.from.nameWithType(), s.weight.toFixed(1), s.to.nameWithType()].join(' -> ')
        }).join('\n')
      )
    }
  }

  Brain.fromOptions = (options) => {
    if (!options.sensors.length) throw new Error('No sensors supplied')
    if (!options.urges.length) throw new Error('No urges supplied')

    const sensors = options.sensors
    const innerNeurons = options.innerNeurons || createArray(options.power, (_, index) => new Neuron(`INNER_${index}`))
    const urges = options.urges
    const synapses = options.synapses || []

    if (!synapses.length) {
      shuffle(sensors, randInt(1, sensors.length))
        .forEach((sensor) => {
          shuffle(innerNeurons, randInt(1, innerNeurons.length))
            .forEach((innerNeuron) => synapses.push(new Synapse(sensor, innerNeuron, Math.random())))
        })

      shuffle(innerNeurons, randInt(0, innerNeurons.length))
        .forEach((innerNeuronFrom) => {
          shuffle(innerNeurons, randInt(0, innerNeurons.length))
            .filter((n) => n !== innerNeuronFrom)
            .forEach((innerNeuronTo) => synapses.push(new Synapse(innerNeuronFrom, innerNeuronTo, Math.random())))
        })

      shuffle(innerNeurons, randInt(1, innerNeurons.length))
        .forEach((innerNeuron) => {
          shuffle(urges, randInt(1, urges.length))
            .forEach((action) => synapses.push(new Synapse(innerNeuron, action, Math.random())))
        })
    }

    return new Brain(sensors, urges, innerNeurons, synapses)
  }

  class Creature {
    constructor(world, brain, color) {
      this.world = world
      this.brain = brain
      this.isDestroyed = false
      this.color = color || { r: randInt(0, 255), g: randInt(0, 255), b: randInt(0, 255) }
    }

    adapt() {
      if (this.isDestroyed) {
        return
      }

      this.act( this.brain.fire(this) )
    }

    act(activatedUrges) {
      activatedUrges.forEach((urge) => urge.act(this))
    }

    move(deltaX, deltaY) {
      if (deltaX !== 0) {
        this.position.x = Math.max(Math.min(this.position.x + deltaX, this.world.width), 0)
      }

      if (deltaY !== 0) {
        const projectedY = this.position.y + deltaY

        this.position.y = Math.max(Math.min(projectedY, this.world.height), 0)

        if (projectedY >= this.world.height) {
          this.world.multiply(this)
          this.world.multiply(this)
          this.destroy()
        }
      }
    }

    look(direction) {
      if (direction === DIRECTION_NORTH) {
        return this.world.height - this.position.y
      } else if (direction === DIRECTION_SOUTH) {
        return this.position.y
      } else if (direction === DIRECTION_EAST) {
        return this.world.width - this.position.x
      } else if (direction === DIRECTION_WEST) {
        return this.position.x
      }
    }

    spawn() {
      this.position = {
        x: randInt(0, this.world.width),
        y: randInt(0, this.world.height),
      }
    }

    symbol() {
      return this.isDestroyed ? ' ' : `<span style="color: rgb(${this.color.r}, ${this.color.g}, ${this.color.b})">*</span>`
    }

    destroy() {
      this.isDestroyed = true
    }
  }

  class World {
    constructor(mapWidth, mapHeight, numberOfCreatures, brainOptions, mutation = 0.1) {
      this.width = mapWidth
      this.height = mapHeight
      this.brainOptions = brainOptions
      this.creatures = createArray(numberOfCreatures, () => new Creature(this, Brain.fromOptions(brainOptions)))
      this.mutation = mutation
    }

    start(n, delay, renderElement) {
      this.creatures.forEach((c) => c.spawn())
      this.loop(n, delay, renderElement)
    }

    loop(n, delay, renderElement) {
      this.creatures.forEach((c) => c.adapt())
      this.render(renderElement)

      if (n <= 0) return

      setTimeout(() => this.loop(n - 1, delay, renderElement), delay)
    }

    renderCreature(x, y) {
      const creature = this.creatures
        .find((c) => c.position.x === x && c.position.y === y)

      return creature ? creature.symbol() : ' '
    }

    multiply(source) {
      const sourceBrain = source.brain
      const brainOptions = {
        sensors: sourceBrain.sensors.map((s) => Sensor.fromSource(s)),
        innerNeurons: sourceBrain.innerNeurons.map((i) => Neuron.fromSource(i)),
        urges: sourceBrain.urges.map((a) => Action.fromSource(a)),
      }
      const neurons = brainOptions.sensors.concat(brainOptions.innerNeurons).concat(brainOptions.urges)

      brainOptions.synapses = sourceBrain.synapses.map((s) => Synapse.fromSourceWithMutation(s, neurons, this.mutation))

      const creature = new Creature(this, Brain.fromOptions(brainOptions), source.color)

      this.creatures.push(creature)

      creature.spawn()
    }

    render(renderElement) {
      const header = '|' + Array.from(Array(this.width + 3).keys(), () => '|').join('') + '|'
      const footer = '|' + Array.from(Array(this.width + 3).keys(), () => '_').join('') + '|'
      const rows = []

      for (var y = 0; y <= this.height; y++) {
        const row = []

        for (var x = 0; x <= this.width; x++) {
          row.push(this.renderCreature(x, this.height - y))
        }

        rows.push('| ' + row.join('') + ' |')
      }

      const LINE_BREAK = '\n'
      let output = header + LINE_BREAK + rows.join(LINE_BREAK) + LINE_BREAK + footer
      output += LINE_BREAK + 'REMAINING: ' + this.creatures.filter((c) => !c.isDestroyed).length

      if (renderElement) {
        renderElement.cols = this.width
        renderElement.rows = this.height

        renderElement.innerHTML = output
      } else {
        console.log(CLEAR_SCREEN)
        console.log(output)
      }
    }
  }

  const IS_MOBILE = window.outerWidth <= 640
  const NUMBER_OF_CREATURES = 5 // IS_MOBILE ? 10 : 20
  const MAP_WIDTH = IS_MOBILE ? 20 : 60
  const MAP_HEIGHT = IS_MOBILE ? 15 : 20
  const ITERATIONS = 50
  const ITERATION_DELAY = 200
  const RENDER_ELEMENT = document.querySelector('#neurenv')
  const BRAIN_OPTIONS = {
    sensors: [
      new Sensor(`SEE_${DIRECTION_NORTH}`, (creature) => creature.look(DIRECTION_NORTH)),
      new Sensor(`SEE_${DIRECTION_SOUTH}`, (creature) => creature.look(DIRECTION_SOUTH)),
      new Sensor(`SEE_${DIRECTION_EAST}`, (creature) => creature.look(DIRECTION_EAST)),
      new Sensor(`SEE_${DIRECTION_WEST}`, (creature) => creature.look(DIRECTION_WEST)),
    ],
    urges: [
      new Action(`MOVE_${DIRECTION_NORTH}`, (creature) => creature.move(0, +1)),
      new Action(`MOVE_${DIRECTION_SOUTH}`, (creature) => creature.move(0, -1)),
      new Action(`MOVE_${DIRECTION_EAST}`, (creature) => creature.move(1, 0)),
      new Action(`MOVE_${DIRECTION_WEST}`, (creature) => creature.move(-1, 0)),
    ],
    power: 2
  }

  const world = new World(MAP_WIDTH, MAP_HEIGHT, NUMBER_OF_CREATURES, BRAIN_OPTIONS)
  world.start(ITERATIONS, ITERATION_DELAY, RENDER_ELEMENT)
  // world.creatures.forEach((c) => c.brain.log())
  // world.render(RENDER_ELEMENT)
</script>
