---
layout: principle
title: Neurenv
description: An exploration in neural nets and how they evolve.
---

<div class="overflow-auto bg-gray-800 bg-gradient-to-r from-gray-900 to-sky-900 text-green-400 text-center p-16 mb-10">
  <code id="neurenv" class="inline-block whitespace-pre text-left"></code>
</div>

The source code is embedded on this page. Have a look in the developer console for more! 👻

<script type="text/javascript">
  const CLEAR_SCREEN = '\033[2J'
  const shuffle = (arr, n) => arr.sort(() => 0.5 - Math.random()).slice(0, n)
  const rand = (min, max) => Math.random() * (max - min) + min
  const randInt = (min, max) => parseInt(rand(min, max))
  const createArray = (length, func) => Array.apply(null, { length }).map(func)

  class Neuron {
    constructor(name) {
      this.name = name
    }

    nameWithType() {
      return `${this.constructor.name}:${this.name}`
    }
  }

  class Sensor extends Neuron { }

  class SightSensor extends Sensor { }

  class Action extends Neuron {
    constructor(name, func) {
      super(name)

      this.func = func
    }
  }

  class Synapse {
    constructor(from, to, weight, activation) {
      this.from = from
      this.to = to
      this.weight = weight
      this.activation = activation
    }
  }

  class Brain {
    constructor(creature, sensors, power, urges) {
      this.creature = creature
      this.sensors = sensors
      this.innerNeurons = createArray(power, (index) => new Neuron(`INNER_${index}`))
      this.urges = urges
      this.synapses = []

      this.wireRandomly()
    }

    wireRandomly() {
      shuffle(this.sensors, randInt(1, this.sensors.length))
        .forEach((sensor) => {
          shuffle(this.innerNeurons, randInt(1, this.innerNeurons.length))
            .forEach((innerNeuron) => this.addSynapse(sensor, innerNeuron))
        })

      shuffle(this.innerNeurons, randInt(0, this.innerNeurons.length))
        .forEach((innerNeuronFrom) => {
          shuffle(this.innerNeurons, randInt(0, this.innerNeurons.length))
            .filter((n) => n !== innerNeuronFrom)
            .forEach((innerNeuronTo) => this.addSynapse(innerNeuronFrom, innerNeuronTo))
        })

      shuffle(this.innerNeurons, randInt(1, this.innerNeurons.length))
        .forEach((innerNeuron) => {
          shuffle(this.urges, randInt(1, this.urges.length))
            .forEach((action) => this.addSynapse(innerNeuron, action))
        })
    }

    addSynapse(from, to) {
      this.synapses.push(new Synapse(from, to, Math.random(), Math.random()))
    }

    execute() {
      const weightedUrges = createArray(this.urges.length, () => Math.random())

//       this.sensors.forEach(() => {
//
//       })

      return weightedUrges
    }

    log() {
      console.log(
        this.synapses.map((s) => {
          return [s.from.nameWithType(), s.weight.toFixed(1), s.to.nameWithType()].join(' -> ')
        }).join('\n')
      )
    }
  }

  class Creature {
    constructor(world, brainPower) {
      this.world = world
      this.sensors = [
        new Sensor('SEE_NORTH'),
        new Sensor('SEE_SOUTH'),
        new Sensor('SEE_EAST'),
        new Sensor('SEE_WEST'),
      ]
      this.urges = [
        new Action('MOVE_NORTH', () => this.move(0, +1)),
        new Action('MOVE_SOUTH', () => this.move(0, -1)),
        new Action('MOVE_EAST', () => this.move(1, 0)),
        new Action('MOVE_WEST', () => this.move(-1, 0)),
      ]
      this.brain = new Brain(this, this.sensors, brainPower, this.urges)
      this.position = {
        x: randInt(0, this.world.width),
        y: randInt(0, this.world.height),
      }
      this.isDestroyed = false
    }

    adapt() {
      if (this.isDestroyed) {
        return
      }

      this.act( this.brain.execute() )
    }

    move(deltaX, deltaY) {
      if (deltaX !== 0) {
        this.position.x = Math.max(Math.min(this.position.x + deltaX, this.world.width), 0)
      }

      if (deltaY !== 0) {
        this.position.y = Math.max(Math.min(this.position.y + deltaY, this.world.height), 0)
      }
    }

    act(weightedUrges) {
      this.urges
        .filter((_, index) => weightedUrges[index] > 0.25)
        // TODO: filter by some activation function?
        .forEach((action) => action.func.call(this))
    }

    symbol() {
      return this.isDestroyed ? 'x' : '*'
    }

    destroy() {
      this.isDestroyed = true
    }
  }

  class World {
    constructor(mapWidth, mapHeight, numberOfCreatures, brainPower) {
      this.width = mapWidth
      this.height = mapHeight
      this.brainPower = brainPower
      this.creatures = createArray(numberOfCreatures, () => new Creature(this, brainPower))
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

    render(renderElement) {
      const header = '|' + Array.from(Array(this.width + 3).keys(), () => '-').join('') + '|'
      const rows = []

      for (var y = 0; y <= this.height; y++) {
        const row = []

        for (var x = 0; x <= this.width; x++) {
          row.push(this.renderCreature(x, this.height - y))
        }

        rows.push('| ' + row.join('') + ' |')
      }

      const LINE_BREAK = '\n'
      let output = header + LINE_BREAK + rows.join(LINE_BREAK) + LINE_BREAK + header
      output += LINE_BREAK + 'REMAINING: ' + this.creatures.filter((c) => !c.isDestroyed).length

      if (renderElement) {
        renderElement.cols = this.width
        renderElement.rows = this.height

        renderElement.textContent = output
      } else {
        console.log(output)
      }
    }
  }

  const NUMBER_OF_CREATURES = 20
  const MAP_WIDTH = 60
  const MAP_HEIGHT = 15
  const BRAIN_POWER = 8
  const ITERATIONS = 1000
  const ITERATION_DELAY = 100
  const RENDER_ELEMENT = document.querySelector('#neurenv')

  const world = new World(MAP_WIDTH, MAP_HEIGHT, NUMBER_OF_CREATURES, BRAIN_POWER)
  world.creatures.forEach((c) => c.brain.log())
  world.loop(ITERATIONS, ITERATION_DELAY, RENDER_ELEMENT)
  // world.render(RENDER_ELEMENT)
</script>
